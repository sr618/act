# **Phase 3: Inventory Module**
**Duration:** 8 Weeks  
**Team:** Backend (2), Frontend (2), QA (1)  
**Dependencies:** Phase 1 (Core Accounting) Complete  

---

## **1. Overview**

The Inventory Module provides comprehensive stock management capabilities:
- Multi-godown inventory tracking
- Batch and Expiry management (FEFO)
- Stock valuation methods (FIFO, LIFO, Weighted Average)
- Stock transfers and adjustments
- Real-time stock visibility

This module integrates with Core Accounting to auto-generate stock vouchers.

---

## **2. Module Architecture**

```
┌─────────────────────────────────────────────────────────────────┐
│                     INVENTORY MODULE                             │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    Item     │  │   Stock     │  │   Godown    │             │
│  │   Master    │  │ Transactions│  │ Management  │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│  ┌──────┴────────────────┴────────────────┴──────┐             │
│  │            STOCK VALUATION ENGINE             │             │
│  │     (FIFO, LIFO, Weighted Average, Batch)     │             │
│  └───────────────────────┬───────────────────────┘             │
│                          │                                      │
│  ┌───────────────────────┴───────────────────────┐             │
│  │             BATCH & EXPIRY ENGINE             │             │
│  │          (FEFO - First Expiry First Out)      │             │
│  └───────────────────────────────────────────────┘             │
├─────────────────────────────────────────────────────────────────┤
│  EVENT BUS: Subscribes to SalesVoucherPostedEvent               │
│             Publishes: StockUpdatedEvent, LowStockAlertEvent    │
└─────────────────────────────────────────────────────────────────┘
```

---

## **3. Domain Model**

### **3.1 Item Master**

```csharp
// Domain/Entities/Item.cs
public class Item : AggregateRoot<ItemId>
{
    public TenantId TenantId { get; private set; }
    
    // Basic Info
    public string Name { get; private set; }
    public string? Code { get; private set; }
    public string? Alias { get; private set; }
    public string? Description { get; private set; }
    public ItemCategoryId CategoryId { get; private set; }
    
    // Classification
    public ItemType Type { get; private set; }  // StockItem, Service, FixedAsset
    public ItemNature Nature { get; private set; }  // RawMaterial, SemiFinished, FinishedGoods, Consumable
    
    // Units
    public UnitOfMeasureId PrimaryUomId { get; private set; }
    public UnitOfMeasureId? SecondaryUomId { get; private set; }
    public decimal ConversionFactor { get; private set; } = 1;  // How many primary = 1 secondary
    
    // GST & HSN
    public string HsnCode { get; private set; }
    public decimal GstRate { get; private set; }
    public decimal? CessRate { get; private set; }
    public string? SacCode { get; private set; }  // For services
    
    // Stock Settings
    public StockValuationMethod ValuationMethod { get; private set; }
    public bool MaintainBatches { get; private set; }
    public bool TrackExpiry { get; private set; }
    public bool TrackSerialNumbers { get; private set; }
    public int? ShelfLife { get; private set; }  // Days
    
    // Reorder
    public decimal? ReorderLevel { get; private set; }
    public decimal? ReorderQuantity { get; private set; }
    public decimal? MinimumStock { get; private set; }
    public decimal? MaximumStock { get; private set; }
    
    // Pricing
    public Money? StandardCost { get; private set; }
    public Money? SellingPrice { get; private set; }
    public Money? MRP { get; private set; }
    
    // Default Ledgers for Accounting Integration
    public LedgerId? SalesLedgerId { get; private set; }
    public LedgerId? PurchaseLedgerId { get; private set; }
    public LedgerId? StockLedgerId { get; private set; }
    
    // Flags
    public bool IsActive { get; private set; } = true;
    public bool AllowNegativeStock { get; private set; } = false;
    
    // Computed Current Stock
    public decimal CurrentStock { get; private set; }
    public Money CurrentValue { get; private set; }
    
    // BOM (if this item is manufactured)
    private readonly List<BomComponent> _bomComponents = new();
    public IReadOnlyCollection<BomComponent> BomComponents => _bomComponents.AsReadOnly();
    
    public bool HasBom => _bomComponents.Any();
}

public enum ItemType
{
    StockItem = 1,
    Service = 2,
    FixedAsset = 3
}

public enum ItemNature
{
    RawMaterial = 1,
    SemiFinished = 2,
    FinishedGoods = 3,
    Consumable = 4,
    Scrap = 5,
    ByProduct = 6
}

public enum StockValuationMethod
{
    Fifo = 1,           // First In First Out
    Lifo = 2,           // Last In First Out
    WeightedAverage = 3, // Moving Weighted Average
    SpecificBatch = 4,   // Track by batch
    StandardCost = 5     // Use standard cost
}

// Domain/Entities/ItemCategory.cs
public class ItemCategory : AggregateRoot<ItemCategoryId>
{
    public TenantId TenantId { get; private set; }
    public string Name { get; private set; }
    public string? Code { get; private set; }
    public ItemCategoryId? ParentCategoryId { get; private set; }
    public int Level { get; private set; }
    public bool IsActive { get; private set; }
    
    // Default settings for items in this category
    public StockValuationMethod? DefaultValuationMethod { get; private set; }
    public bool? DefaultMaintainBatches { get; private set; }
    public decimal? DefaultGstRate { get; private set; }
}

// Domain/Entities/UnitOfMeasure.cs
public class UnitOfMeasure : AggregateRoot<UnitOfMeasureId>
{
    public TenantId TenantId { get; private set; }
    public string Name { get; private set; }
    public string Symbol { get; private set; }  // Kg, Pcs, Mtr, Ltr
    public UomType Type { get; private set; }   // Weight, Count, Length, Volume
    public int DecimalPlaces { get; private set; }
    public bool IsSystemUnit { get; private set; }
}
```

### **3.2 Godown (Warehouse) Management**

```csharp
// Domain/Entities/Godown.cs
public class Godown : AggregateRoot<GodownId>
{
    public TenantId TenantId { get; private set; }
    public string Name { get; private set; }
    public string? Code { get; private set; }
    public GodownId? ParentGodownId { get; private set; }  // For sub-locations
    public GodownType Type { get; private set; }
    
    // Address
    public Address? Address { get; private set; }
    
    // Responsible Person
    public UserId? InChargeUserId { get; private set; }
    public string? InChargeName { get; private set; }
    public string? InChargePhone { get; private set; }
    
    // Flags
    public bool IsActive { get; private set; }
    public bool IsDefault { get; private set; }
    public bool IsVirtual { get; private set; }  // For Job Work tracking
    
    // Constraints
    public bool AllowNegativeStock { get; private set; }
    
    // GST (for multi-state businesses)
    public string? GstinForGodown { get; private set; }
}

public enum GodownType
{
    Warehouse = 1,
    Shop = 2,
    Factory = 3,
    Vehicle = 4,
    JobWorker = 5,  // Virtual godown for job work tracking
    Transit = 6,     // Goods in transit
    Rejected = 7     // Quality rejected items
}
```

### **3.3 Stock Transactions**

```csharp
// Domain/Entities/StockTransaction.cs
public class StockTransaction : AggregateRoot<StockTransactionId>
{
    public TenantId TenantId { get; private set; }
    public FinancialYearId FinancialYearId { get; private set; }
    
    // Source
    public StockTransactionType Type { get; private set; }
    public string TransactionNumber { get; private set; }
    public DateOnly TransactionDate { get; private set; }
    
    // Reference to source document
    public string SourceModule { get; private set; }  // "Sales", "Purchase", "Manufacturing"
    public Guid? SourceDocumentId { get; private set; }
    public Guid? LinkedVoucherId { get; private set; }  // Accounting voucher
    
    // Lines
    private readonly List<StockTransactionLine> _lines = new();
    public IReadOnlyCollection<StockTransactionLine> Lines => _lines.AsReadOnly();
    
    // Totals
    public decimal TotalQuantity => _lines.Sum(l => l.Quantity);
    public Money TotalValue => new Money(_lines.Sum(l => l.TotalValue.Amount), Currency.INR);
    
    // Status
    public StockTransactionStatus Status { get; private set; }
    
    public void AddLine(
        ItemId itemId,
        GodownId godownId,
        decimal quantity,
        Money rate,
        StockDirection direction,
        BatchId? batchId = null)
    {
        var line = new StockTransactionLine(
            id: StockTransactionLineId.Create(),
            transactionId: Id,
            itemId: itemId,
            godownId: godownId,
            quantity: quantity,
            rate: rate,
            direction: direction,
            batchId: batchId,
            lineNumber: _lines.Count + 1
        );
        
        _lines.Add(line);
        AddDomainEvent(new StockLineAddedEvent(line));
    }
}

public class StockTransactionLine : Entity<StockTransactionLineId>
{
    public StockTransactionId TransactionId { get; private set; }
    public ItemId ItemId { get; private set; }
    public GodownId GodownId { get; private set; }
    
    // Quantity & Value
    public decimal Quantity { get; private set; }
    public Money Rate { get; private set; }
    public Money TotalValue => Rate.Multiply(Quantity);
    public StockDirection Direction { get; private set; }  // In or Out
    
    // Batch (if applicable)
    public BatchId? BatchId { get; private set; }
    public string? BatchNumber { get; private set; }
    public DateOnly? ExpiryDate { get; private set; }
    
    // Serial Numbers (if applicable)
    private readonly List<string> _serialNumbers = new();
    public IReadOnlyCollection<string> SerialNumbers => _serialNumbers.AsReadOnly();
    
    // For Transfers
    public GodownId? DestinationGodownId { get; private set; }
    
    public int LineNumber { get; private set; }
}

public enum StockTransactionType
{
    Purchase = 1,
    PurchaseReturn = 2,
    Sales = 3,
    SalesReturn = 4,
    StockTransfer = 5,
    StockAdjustment = 6,
    OpeningStock = 7,
    Manufacturing = 8,
    MaterialConsumption = 9,
    Scrap = 10,
    JobWorkOut = 11,
    JobWorkIn = 12
}

public enum StockDirection
{
    In = 1,
    Out = 2
}
```

### **3.4 Batch & Expiry Tracking**

```csharp
// Domain/Entities/Batch.cs
public class Batch : AggregateRoot<BatchId>
{
    public TenantId TenantId { get; private set; }
    public ItemId ItemId { get; private set; }
    public GodownId GodownId { get; private set; }
    
    public string BatchNumber { get; private set; }
    public DateOnly ManufacturingDate { get; private set; }
    public DateOnly ExpiryDate { get; private set; }
    
    // Current State
    public decimal CurrentQuantity { get; private set; }
    public Money CurrentValue { get; private set; }
    public decimal CostPerUnit { get; private set; }
    
    // Source
    public Guid? SourceTransactionId { get; private set; }
    public string? SupplierBatchNumber { get; private set; }
    
    // Status
    public BatchStatus Status { get; private set; }
    
    public int DaysToExpiry => (ExpiryDate.ToDateTime(TimeOnly.MinValue) - DateTime.Today).Days;
    public bool IsExpired => DaysToExpiry < 0;
    public bool IsNearExpiry(int warningDays) => DaysToExpiry <= warningDays && DaysToExpiry >= 0;
    
    public void ReduceQuantity(decimal quantity)
    {
        if (quantity > CurrentQuantity)
            throw new DomainException($"Cannot reduce {quantity} from batch {BatchNumber}. Available: {CurrentQuantity}");
        
        CurrentQuantity -= quantity;
        CurrentValue = new Money(CurrentQuantity * CostPerUnit, Currency.INR);
        
        if (CurrentQuantity == 0)
            Status = BatchStatus.Exhausted;
    }
}

public enum BatchStatus
{
    Active = 1,
    Exhausted = 2,
    Expired = 3,
    Blocked = 4,
    Quarantine = 5
}

// Domain/Services/FefoService.cs (First Expiry First Out)
public class FefoService
{
    private readonly IBatchRepository _batchRepo;
    
    public async Task<List<BatchAllocation>> AllocateBatchesAsync(
        ItemId itemId,
        GodownId godownId,
        decimal requiredQuantity,
        CancellationToken ct)
    {
        // Get batches sorted by expiry date (earliest first)
        var batches = await _batchRepo.GetActiveBatchesAsync(itemId, godownId, ct);
        var sortedBatches = batches
            .Where(b => !b.IsExpired && b.CurrentQuantity > 0)
            .OrderBy(b => b.ExpiryDate)
            .ToList();
        
        var allocations = new List<BatchAllocation>();
        var remainingQty = requiredQuantity;
        
        foreach (var batch in sortedBatches)
        {
            if (remainingQty <= 0) break;
            
            var allocateQty = Math.Min(remainingQty, batch.CurrentQuantity);
            allocations.Add(new BatchAllocation(batch.Id, batch.BatchNumber, allocateQty, batch.ExpiryDate));
            remainingQty -= allocateQty;
        }
        
        if (remainingQty > 0)
        {
            throw new DomainException($"Insufficient stock. Required: {requiredQuantity}, Available: {requiredQuantity - remainingQty}");
        }
        
        return allocations;
    }
}
```

### **3.5 Stock Ledger (Running Balance)**

```csharp
// Domain/Entities/StockLedger.cs
// Maintains running balance per Item per Godown
public class StockLedgerEntry : Entity<StockLedgerEntryId>
{
    public TenantId TenantId { get; private set; }
    public ItemId ItemId { get; private set; }
    public GodownId GodownId { get; private set; }
    public BatchId? BatchId { get; private set; }
    
    public DateOnly Date { get; private set; }
    public StockTransactionId TransactionId { get; private set; }
    public string TransactionNumber { get; private set; }
    public StockTransactionType TransactionType { get; private set; }
    
    // In
    public decimal InQuantity { get; private set; }
    public Money InRate { get; private set; }
    public Money InValue { get; private set; }
    
    // Out
    public decimal OutQuantity { get; private set; }
    public Money OutRate { get; private set; }
    public Money OutValue { get; private set; }
    
    // Running Balance
    public decimal BalanceQuantity { get; private set; }
    public Money BalanceValue { get; private set; }
    public Money AverageRate { get; private set; }
}
```

---

## **4. Stock Valuation Engine**

```csharp
// Domain/Services/StockValuationService.cs
public class StockValuationService : IStockValuationService
{
    public Money CalculateIssueRate(
        Item item,
        GodownId godownId,
        decimal quantity,
        List<StockLedgerEntry> stockLedger)
    {
        return item.ValuationMethod switch
        {
            StockValuationMethod.Fifo => CalculateFifoRate(stockLedger, quantity),
            StockValuationMethod.Lifo => CalculateLifoRate(stockLedger, quantity),
            StockValuationMethod.WeightedAverage => CalculateWeightedAverageRate(stockLedger),
            StockValuationMethod.StandardCost => item.StandardCost ?? Money.Zero,
            _ => throw new DomainException($"Unsupported valuation method: {item.ValuationMethod}")
        };
    }
    
    private Money CalculateFifoRate(List<StockLedgerEntry> ledger, decimal quantity)
    {
        var inEntries = ledger
            .Where(e => e.InQuantity > 0)
            .OrderBy(e => e.Date)
            .ToList();
        
        decimal remainingQty = quantity;
        decimal totalValue = 0;
        
        foreach (var entry in inEntries)
        {
            var availableQty = entry.InQuantity;  // Simplified - should track consumed qty
            var consumeQty = Math.Min(remainingQty, availableQty);
            
            totalValue += entry.InRate.Amount * consumeQty;
            remainingQty -= consumeQty;
            
            if (remainingQty <= 0) break;
        }
        
        return Money.INR(totalValue / quantity);
    }
    
    private Money CalculateWeightedAverageRate(List<StockLedgerEntry> ledger)
    {
        var lastEntry = ledger.OrderByDescending(e => e.Date).FirstOrDefault();
        return lastEntry?.AverageRate ?? Money.Zero;
    }
    
    // Update weighted average after each receipt
    public Money RecalculateWeightedAverage(
        decimal existingQty,
        Money existingValue,
        decimal newQty,
        Money newRate)
    {
        var totalQty = existingQty + newQty;
        var totalValue = existingValue.Amount + (newQty * newRate.Amount);
        
        return totalQty > 0 
            ? Money.INR(totalValue / totalQty) 
            : Money.Zero;
    }
}
```

---

## **5. Application Layer**

### **5.1 Commands**

```csharp
// Application/Commands/CreateItem/CreateItemCommand.cs
public record CreateItemCommand : IRequest<Result<ItemId>>
{
    public string Name { get; init; } = string.Empty;
    public string? Code { get; init; }
    public Guid CategoryId { get; init; }
    public ItemType Type { get; init; }
    public ItemNature Nature { get; init; }
    public Guid PrimaryUomId { get; init; }
    public string HsnCode { get; init; } = string.Empty;
    public decimal GstRate { get; init; }
    public StockValuationMethod ValuationMethod { get; init; }
    public bool MaintainBatches { get; init; }
    public bool TrackExpiry { get; init; }
    public decimal? ReorderLevel { get; init; }
    public decimal? SellingPrice { get; init; }
}

// Application/Commands/RecordStockTransaction/RecordStockTransactionCommand.cs
public record RecordStockTransactionCommand : IRequest<Result<StockTransactionId>>
{
    public StockTransactionType Type { get; init; }
    public DateOnly TransactionDate { get; init; }
    public Guid? SourceDocumentId { get; init; }
    public string SourceModule { get; init; } = "Inventory";
    public List<StockLineDto> Lines { get; init; } = new();
}

public record StockLineDto
{
    public Guid ItemId { get; init; }
    public Guid GodownId { get; init; }
    public decimal Quantity { get; init; }
    public decimal? Rate { get; init; }
    public StockDirection Direction { get; init; }
    public string? BatchNumber { get; init; }
    public DateOnly? ExpiryDate { get; init; }
    public Guid? DestinationGodownId { get; init; }  // For transfers
}

public class RecordStockTransactionCommandHandler : IRequestHandler<RecordStockTransactionCommand, Result<StockTransactionId>>
{
    private readonly IStockTransactionRepository _stockRepo;
    private readonly IItemRepository _itemRepo;
    private readonly IStockValuationService _valuationService;
    private readonly IEventBus _eventBus;
    
    public async Task<Result<StockTransactionId>> Handle(RecordStockTransactionCommand request, CancellationToken ct)
    {
        var transaction = StockTransaction.Create(
            type: request.Type,
            transactionDate: request.TransactionDate,
            sourceModule: request.SourceModule,
            sourceDocumentId: request.SourceDocumentId
        );
        
        foreach (var line in request.Lines)
        {
            var item = await _itemRepo.GetByIdAsync(ItemId.From(line.ItemId), ct);
            
            // Calculate rate for outgoing stock
            var rate = line.Direction == StockDirection.Out
                ? await CalculateIssueRateAsync(item, line.GodownId, line.Quantity, ct)
                : Money.INR(line.Rate ?? 0);
            
            // Handle batches
            BatchId? batchId = null;
            if (item.MaintainBatches && line.Direction == StockDirection.In)
            {
                batchId = await CreateOrGetBatchAsync(item, line, rate, ct);
            }
            else if (item.MaintainBatches && line.Direction == StockDirection.Out)
            {
                // Auto-allocate batches using FEFO
                var allocations = await _fefoService.AllocateBatchesAsync(
                    item.Id, GodownId.From(line.GodownId), line.Quantity, ct);
                
                foreach (var allocation in allocations)
                {
                    transaction.AddLine(
                        itemId: item.Id,
                        godownId: GodownId.From(line.GodownId),
                        quantity: allocation.Quantity,
                        rate: rate,
                        direction: line.Direction,
                        batchId: allocation.BatchId
                    );
                }
                continue;  // Skip normal line addition
            }
            
            transaction.AddLine(
                itemId: item.Id,
                godownId: GodownId.From(line.GodownId),
                quantity: line.Quantity,
                rate: rate,
                direction: line.Direction,
                batchId: batchId
            );
        }
        
        await _stockRepo.AddAsync(transaction, ct);
        
        // Publish event for other modules
        await _eventBus.PublishAsync(new StockTransactionCreatedEvent(transaction));
        
        // Check and raise low stock alerts
        await CheckLowStockAlertsAsync(transaction, ct);
        
        return Result.Success(transaction.Id);
    }
}

// Application/Commands/StockTransfer/StockTransferCommand.cs
public record StockTransferCommand : IRequest<Result<StockTransactionId>>
{
    public DateOnly TransferDate { get; init; }
    public Guid SourceGodownId { get; init; }
    public Guid DestinationGodownId { get; init; }
    public List<TransferLineDto> Lines { get; init; } = new();
    public string? Remarks { get; init; }
}

// Application/Commands/StockAdjustment/StockAdjustmentCommand.cs
public record StockAdjustmentCommand : IRequest<Result<StockTransactionId>>
{
    public DateOnly AdjustmentDate { get; init; }
    public AdjustmentReason Reason { get; init; }
    public string? Remarks { get; init; }
    public List<AdjustmentLineDto> Lines { get; init; } = new();
}

public enum AdjustmentReason
{
    PhysicalCount = 1,
    Damage = 2,
    Expired = 3,
    Theft = 4,
    Other = 5
}
```

### **5.2 Queries**

```csharp
// Application/Queries/GetStockSummary/GetStockSummaryQuery.cs
public record GetStockSummaryQuery : IRequest<StockSummaryDto>
{
    public Guid? ItemId { get; init; }
    public Guid? GodownId { get; init; }
    public Guid? CategoryId { get; init; }
    public bool IncludeZeroStock { get; init; }
}

public record StockSummaryDto
{
    public List<StockSummaryLineDto> Items { get; init; } = new();
    public decimal TotalValue { get; init; }
    public int TotalItems { get; init; }
    public int LowStockItems { get; init; }
    public int ExpiringItems { get; init; }
}

// Application/Queries/GetStockRegister/GetStockRegisterQuery.cs
public record GetStockRegisterQuery : IRequest<StockRegisterDto>
{
    public Guid ItemId { get; init; }
    public Guid? GodownId { get; init; }
    public DateOnly FromDate { get; init; }
    public DateOnly ToDate { get; init; }
}

// Application/Queries/GetExpiringStock/GetExpiringStockQuery.cs
public record GetExpiringStockQuery : IRequest<List<ExpiringStockDto>>
{
    public int DaysThreshold { get; init; } = 30;  // Items expiring in next 30 days
    public Guid? GodownId { get; init; }
}

// Application/Queries/GetReorderReport/GetReorderReportQuery.cs
public record GetReorderReportQuery : IRequest<List<ReorderItemDto>>
{
    // Items below reorder level
}
```

---

## **6. API Endpoints**

```csharp
// API/Controllers/ItemsController.cs
[ApiController]
[Route("api/v1/inventory/items")]
[Authorize]
[RequireModule(ModuleType.Inventory)]
public class ItemsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<PagedResult<ItemDto>>> GetAll(
        [FromQuery] string? search,
        [FromQuery] Guid? categoryId,
        [FromQuery] ItemType? type,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 50)
    {
        return Ok(await _mediator.Send(new GetItemsQuery { /* ... */ }));
    }
    
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<ItemDetailsDto>> GetById(Guid id)
    {
        return Ok(await _mediator.Send(new GetItemByIdQuery { Id = id }));
    }
    
    [HttpPost]
    [RequirePermission(Permissions.Inventory.CreateItem)]
    public async Task<ActionResult<ItemId>> Create([FromBody] CreateItemCommand command)
    {
        var result = await _mediator.Send(command);
        if (result.IsFailure)
            return BadRequest(result.Errors);
        return CreatedAtAction(nameof(GetById), new { id = result.Value }, result.Value);
    }
    
    [HttpGet("{id:guid}/stock")]
    public async Task<ActionResult<ItemStockDto>> GetStock(Guid id, [FromQuery] Guid? godownId)
    {
        return Ok(await _mediator.Send(new GetItemStockQuery { ItemId = id, GodownId = godownId }));
    }
    
    [HttpGet("{id:guid}/batches")]
    public async Task<ActionResult<List<BatchDto>>> GetBatches(Guid id, [FromQuery] Guid? godownId)
    {
        return Ok(await _mediator.Send(new GetItemBatchesQuery { ItemId = id, GodownId = godownId }));
    }
    
    [HttpGet("search")]
    public async Task<ActionResult<List<ItemSearchResultDto>>> Search(
        [FromQuery] string q,
        [FromQuery] int limit = 10)
    {
        // Fast search for transaction entry
        return Ok(await _mediator.Send(new SearchItemsQuery { SearchTerm = q, Limit = limit }));
    }
}

// API/Controllers/StockController.cs
[ApiController]
[Route("api/v1/inventory/stock")]
[Authorize]
[RequireModule(ModuleType.Inventory)]
public class StockController : ControllerBase
{
    [HttpGet("summary")]
    public async Task<ActionResult<StockSummaryDto>> GetSummary(
        [FromQuery] Guid? godownId,
        [FromQuery] Guid? categoryId)
    {
        return Ok(await _mediator.Send(new GetStockSummaryQuery { GodownId = godownId, CategoryId = categoryId }));
    }
    
    [HttpPost("transaction")]
    [RequirePermission(Permissions.Inventory.RecordStock)]
    public async Task<ActionResult<StockTransactionId>> RecordTransaction([FromBody] RecordStockTransactionCommand command)
    {
        var result = await _mediator.Send(command);
        if (result.IsFailure)
            return BadRequest(result.Errors);
        return Ok(result.Value);
    }
    
    [HttpPost("transfer")]
    [RequirePermission(Permissions.Inventory.Transfer)]
    public async Task<ActionResult<StockTransactionId>> Transfer([FromBody] StockTransferCommand command)
    {
        var result = await _mediator.Send(command);
        if (result.IsFailure)
            return BadRequest(result.Errors);
        return Ok(result.Value);
    }
    
    [HttpPost("adjustment")]
    [RequirePermission(Permissions.Inventory.Adjust)]
    public async Task<ActionResult<StockTransactionId>> Adjust([FromBody] StockAdjustmentCommand command)
    {
        var result = await _mediator.Send(command);
        if (result.IsFailure)
            return BadRequest(result.Errors);
        return Ok(result.Value);
    }
    
    [HttpGet("register")]
    public async Task<ActionResult<StockRegisterDto>> GetRegister(
        [FromQuery] Guid itemId,
        [FromQuery] Guid? godownId,
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate)
    {
        return Ok(await _mediator.Send(new GetStockRegisterQuery { /* ... */ }));
    }
    
    [HttpGet("expiring")]
    public async Task<ActionResult<List<ExpiringStockDto>>> GetExpiringStock([FromQuery] int days = 30)
    {
        return Ok(await _mediator.Send(new GetExpiringStockQuery { DaysThreshold = days }));
    }
    
    [HttpGet("low-stock")]
    public async Task<ActionResult<List<LowStockItemDto>>> GetLowStock()
    {
        return Ok(await _mediator.Send(new GetLowStockQuery()));
    }
    
    [HttpGet("valuation")]
    public async Task<ActionResult<StockValuationReportDto>> GetValuation(
        [FromQuery] DateOnly asOnDate,
        [FromQuery] Guid? godownId)
    {
        return Ok(await _mediator.Send(new GetStockValuationQuery { AsOnDate = asOnDate, GodownId = godownId }));
    }
}

// API/Controllers/GodownsController.cs
[ApiController]
[Route("api/v1/inventory/godowns")]
[Authorize]
[RequireModule(ModuleType.Inventory)]
public class GodownsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<List<GodownDto>>> GetAll()
    {
        return Ok(await _mediator.Send(new GetGodownsQuery()));
    }
    
    [HttpPost]
    public async Task<ActionResult<GodownId>> Create([FromBody] CreateGodownCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpGet("{id:guid}/stock")]
    public async Task<ActionResult<GodownStockDto>> GetGodownStock(Guid id)
    {
        return Ok(await _mediator.Send(new GetGodownStockQuery { GodownId = id }));
    }
}
```

---

## **7. Event Handlers (Integration with Core & Sales)**

```csharp
// Application/EventHandlers/SalesVoucherPostedEventHandler.cs
public class SalesVoucherPostedEventHandler : INotificationHandler<VoucherPostedIntegrationEvent>
{
    private readonly ISender _mediator;
    
    public async Task Handle(VoucherPostedIntegrationEvent notification, CancellationToken ct)
    {
        // Only handle sales vouchers
        if (notification.Type is not (VoucherType.Sales or VoucherType.SalesReturn))
            return;
        
        // Extract stock lines from voucher
        var stockLines = notification.StockLines.Select(l => new StockLineDto
        {
            ItemId = l.ItemId,
            GodownId = l.GodownId,
            Quantity = l.Quantity,
            Direction = notification.Type == VoucherType.Sales ? StockDirection.Out : StockDirection.In
        }).ToList();
        
        // Create stock transaction
        var command = new RecordStockTransactionCommand
        {
            Type = notification.Type == VoucherType.Sales 
                ? StockTransactionType.Sales 
                : StockTransactionType.SalesReturn,
            TransactionDate = notification.VoucherDate,
            SourceModule = "Sales",
            SourceDocumentId = notification.VoucherId,
            Lines = stockLines
        };
        
        await _mediator.Send(command, ct);
    }
}

// Application/EventHandlers/PurchaseVoucherPostedEventHandler.cs
public class PurchaseVoucherPostedEventHandler : INotificationHandler<VoucherPostedIntegrationEvent>
{
    public async Task Handle(VoucherPostedIntegrationEvent notification, CancellationToken ct)
    {
        if (notification.Type is not (VoucherType.Purchase or VoucherType.PurchaseReturn))
            return;
        
        // Similar logic for purchase stock updates
    }
}
```

---

## **8. Database Schema**

```sql
-- Inventory Module Tables

CREATE TABLE item_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20),
    parent_category_id UUID REFERENCES item_categories(id),
    level INT DEFAULT 1,
    default_valuation_method VARCHAR(20),
    default_gst_rate DECIMAL(5,2),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_category_name UNIQUE (tenant_id, name)
);

CREATE TABLE units_of_measure (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(50) NOT NULL,
    symbol VARCHAR(10) NOT NULL,
    type VARCHAR(20) NOT NULL,
    decimal_places INT DEFAULT 2,
    is_system_unit BOOLEAN DEFAULT FALSE,
    
    CONSTRAINT unique_uom_name UNIQUE (tenant_id, name)
);

CREATE TABLE items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(200) NOT NULL,
    code VARCHAR(50),
    alias VARCHAR(100),
    description TEXT,
    category_id UUID REFERENCES item_categories(id),
    
    item_type VARCHAR(20) NOT NULL,
    item_nature VARCHAR(20) NOT NULL,
    
    primary_uom_id UUID NOT NULL REFERENCES units_of_measure(id),
    secondary_uom_id UUID REFERENCES units_of_measure(id),
    conversion_factor DECIMAL(18,6) DEFAULT 1,
    
    hsn_code VARCHAR(8) NOT NULL,
    gst_rate DECIMAL(5,2) NOT NULL,
    cess_rate DECIMAL(5,2),
    
    valuation_method VARCHAR(20) NOT NULL DEFAULT 'WeightedAverage',
    maintain_batches BOOLEAN DEFAULT FALSE,
    track_expiry BOOLEAN DEFAULT FALSE,
    track_serial_numbers BOOLEAN DEFAULT FALSE,
    shelf_life INT,
    
    reorder_level DECIMAL(18,4),
    reorder_quantity DECIMAL(18,4),
    minimum_stock DECIMAL(18,4),
    maximum_stock DECIMAL(18,4),
    
    standard_cost DECIMAL(18,4),
    selling_price DECIMAL(18,4),
    mrp DECIMAL(18,4),
    
    sales_ledger_id UUID,
    purchase_ledger_id UUID,
    stock_ledger_id UUID,
    
    allow_negative_stock BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Computed fields (updated by triggers)
    current_stock DECIMAL(18,4) DEFAULT 0,
    current_value DECIMAL(18,4) DEFAULT 0,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ,
    
    CONSTRAINT unique_item_name UNIQUE (tenant_id, name)
);

CREATE TABLE godowns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20),
    parent_godown_id UUID REFERENCES godowns(id),
    godown_type VARCHAR(20) NOT NULL DEFAULT 'Warehouse',
    address JSONB,
    in_charge_user_id UUID REFERENCES users(id),
    in_charge_name VARCHAR(100),
    in_charge_phone VARCHAR(20),
    gstin VARCHAR(15),
    is_active BOOLEAN DEFAULT TRUE,
    is_default BOOLEAN DEFAULT FALSE,
    is_virtual BOOLEAN DEFAULT FALSE,
    allow_negative_stock BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_godown_name UNIQUE (tenant_id, name)
);

CREATE TABLE stock_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    financial_year_id UUID NOT NULL REFERENCES financial_years(id),
    
    transaction_type VARCHAR(30) NOT NULL,
    transaction_number VARCHAR(50) NOT NULL,
    transaction_date DATE NOT NULL,
    
    source_module VARCHAR(50) DEFAULT 'Inventory',
    source_document_id UUID,
    linked_voucher_id UUID REFERENCES vouchers(id),
    
    status VARCHAR(20) DEFAULT 'Posted',
    remarks TEXT,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    created_by UUID NOT NULL REFERENCES users(id),
    
    CONSTRAINT unique_stock_txn_number UNIQUE (tenant_id, financial_year_id, transaction_type, transaction_number)
);

CREATE TABLE stock_transaction_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID NOT NULL REFERENCES stock_transactions(id) ON DELETE CASCADE,
    item_id UUID NOT NULL REFERENCES items(id),
    godown_id UUID NOT NULL REFERENCES godowns(id),
    
    quantity DECIMAL(18,4) NOT NULL,
    rate DECIMAL(18,4) NOT NULL,
    total_value DECIMAL(18,4) NOT NULL,
    direction VARCHAR(10) NOT NULL CHECK (direction IN ('In', 'Out')),
    
    batch_id UUID REFERENCES batches(id),
    batch_number VARCHAR(50),
    expiry_date DATE,
    
    destination_godown_id UUID REFERENCES godowns(id),
    
    serial_numbers TEXT[],
    line_number INT NOT NULL
);

CREATE TABLE batches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    item_id UUID NOT NULL REFERENCES items(id),
    godown_id UUID NOT NULL REFERENCES godowns(id),
    
    batch_number VARCHAR(50) NOT NULL,
    manufacturing_date DATE NOT NULL,
    expiry_date DATE NOT NULL,
    
    current_quantity DECIMAL(18,4) NOT NULL DEFAULT 0,
    current_value DECIMAL(18,4) NOT NULL DEFAULT 0,
    cost_per_unit DECIMAL(18,4) NOT NULL,
    
    source_transaction_id UUID REFERENCES stock_transactions(id),
    supplier_batch_number VARCHAR(50),
    
    status VARCHAR(20) DEFAULT 'Active',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_batch_per_item UNIQUE (tenant_id, item_id, godown_id, batch_number)
);

CREATE TABLE stock_ledger (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    item_id UUID NOT NULL REFERENCES items(id),
    godown_id UUID NOT NULL REFERENCES godowns(id),
    batch_id UUID REFERENCES batches(id),
    
    date DATE NOT NULL,
    transaction_id UUID NOT NULL REFERENCES stock_transactions(id),
    transaction_number VARCHAR(50) NOT NULL,
    transaction_type VARCHAR(30) NOT NULL,
    
    in_quantity DECIMAL(18,4) DEFAULT 0,
    in_rate DECIMAL(18,4) DEFAULT 0,
    in_value DECIMAL(18,4) DEFAULT 0,
    
    out_quantity DECIMAL(18,4) DEFAULT 0,
    out_rate DECIMAL(18,4) DEFAULT 0,
    out_value DECIMAL(18,4) DEFAULT 0,
    
    balance_quantity DECIMAL(18,4) NOT NULL,
    balance_value DECIMAL(18,4) NOT NULL,
    average_rate DECIMAL(18,4) NOT NULL,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_items_category ON items(tenant_id, category_id);
CREATE INDEX idx_items_hsn ON items(tenant_id, hsn_code);
CREATE INDEX idx_stock_txn_date ON stock_transactions(tenant_id, transaction_date);
CREATE INDEX idx_stock_lines_item ON stock_transaction_lines(item_id);
CREATE INDEX idx_batches_item ON batches(item_id, godown_id);
CREATE INDEX idx_batches_expiry ON batches(expiry_date) WHERE status = 'Active';
CREATE INDEX idx_stock_ledger_item ON stock_ledger(item_id, godown_id, date);

-- Trigger to update item current stock
CREATE OR REPLACE FUNCTION update_item_stock()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE items i
    SET 
        current_stock = COALESCE((
            SELECT SUM(CASE WHEN sl.in_quantity > 0 THEN sl.in_quantity ELSE -sl.out_quantity END)
            FROM stock_ledger sl
            WHERE sl.item_id = i.id
        ), 0),
        current_value = COALESCE((
            SELECT SUM(balance_value)
            FROM stock_ledger sl
            WHERE sl.item_id = i.id
            AND sl.id = (
                SELECT id FROM stock_ledger 
                WHERE item_id = sl.item_id 
                ORDER BY date DESC, created_at DESC 
                LIMIT 1
            )
        ), 0),
        updated_at = NOW()
    WHERE i.id = NEW.item_id;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_item_stock
AFTER INSERT ON stock_ledger
FOR EACH ROW
EXECUTE FUNCTION update_item_stock();
```

---

## **9. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| Item Creation | Create item with all attributes in < 1s |
| Stock Query | Get stock summary for 10,000 items in < 2s |
| Batch Allocation | FEFO allocation for 100 batches in < 500ms |
| Stock Register | Generate register for 1 year data in < 3s |
| Expiry Alert | Identify all expiring items in < 1s |
| Stock Valuation | Calculate FIFO/Weighted Avg correctly |
| Multi-Godown | Track same item across 50 godowns |
| Real-time Update | Reflect stock changes within 1s of transaction |

---

## **10. Definition of Done**

- [ ] Item master with full configuration options
- [ ] Multi-godown management
- [ ] All stock transaction types implemented
- [ ] Batch and expiry tracking (FEFO)
- [ ] FIFO, LIFO, Weighted Average valuation
- [ ] Stock register and movement reports
- [ ] Low stock and reorder alerts
- [ ] Integration with Core Accounting vouchers
- [ ] Real-time stock balance updates
- [ ] Stock valuation report
- [ ] Performance benchmarks met
- [ ] Unit test coverage > 85%

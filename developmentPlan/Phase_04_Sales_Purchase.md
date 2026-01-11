# **Phase 4: Sales & Purchase Module**
**Duration:** 6 Weeks  
**Team:** Backend (2), Frontend (2), QA (1)  
**Dependencies:** Phase 1 (Core Accounting), Phase 3 (Inventory)  

---

## **1. Overview**

The Sales & Purchase Module handles the complete transaction lifecycle:
- Quotation → Sales Order → Delivery Challan → Invoice → Receipt
- Purchase Request → Purchase Order → GRN → Bill → Payment

This module orchestrates between Core Accounting (vouchers) and Inventory (stock).

---

## **2. Module Architecture**

```
┌─────────────────────────────────────────────────────────────────┐
│                   SALES & PURCHASE MODULE                        │
├─────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    SALES WORKFLOW                           │ │
│  │  Quotation → Order → Challan → Invoice → Receipt           │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   PURCHASE WORKFLOW                         │ │
│  │  Request → Order → GRN → Bill → Payment                    │ │
│  └────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                 PRICING & DISCOUNT ENGINE                        │
│        (Price Lists, Schemes, Quantity Breaks)                  │
├─────────────────────────────────────────────────────────────────┤
│  INTEGRATIONS:                                                   │
│  → Core Accounting: Auto-create vouchers on Invoice/Bill        │
│  → Inventory: Auto-update stock on Challan/GRN                  │
│  → GST Module: Auto-generate E-Invoice on Sales Invoice         │
└─────────────────────────────────────────────────────────────────┘
```

---

## **3. Domain Model**

### **3.1 Sales Documents**

```csharp
// Domain/Entities/SalesQuotation.cs
public class SalesQuotation : AggregateRoot<SalesQuotationId>
{
    public TenantId TenantId { get; private set; }
    public string QuotationNumber { get; private set; }
    public DateOnly QuotationDate { get; private set; }
    public DateOnly ValidUntil { get; private set; }
    
    // Customer
    public LedgerId CustomerId { get; private set; }
    public string CustomerName { get; private set; }
    public Address? BillingAddress { get; private set; }
    public Address? ShippingAddress { get; private set; }
    
    // GST Details
    public string? CustomerGstin { get; private set; }
    public string PlaceOfSupply { get; private set; }
    public bool IsInterState { get; private set; }
    
    // Terms
    public string? TermsAndConditions { get; private set; }
    public string? Notes { get; private set; }
    
    // Lines
    private readonly List<SalesQuotationLine> _lines = new();
    public IReadOnlyCollection<SalesQuotationLine> Lines => _lines.AsReadOnly();
    
    // Totals
    public Money SubTotal => Money.INR(_lines.Sum(l => l.TaxableValue.Amount));
    public Money TaxAmount => Money.INR(_lines.Sum(l => l.TaxAmount.Amount));
    public Money TotalAmount => SubTotal.Add(TaxAmount);
    
    // Status
    public QuotationStatus Status { get; private set; }
    public Guid? ConvertedToOrderId { get; private set; }
    
    public SalesOrder ConvertToOrder()
    {
        if (Status != QuotationStatus.Approved)
            throw new DomainException("Only approved quotations can be converted to orders");
        
        var order = SalesOrder.CreateFromQuotation(this);
        Status = QuotationStatus.ConvertedToOrder;
        ConvertedToOrderId = order.Id.Value;
        
        return order;
    }
}

// Domain/Entities/SalesOrder.cs
public class SalesOrder : AggregateRoot<SalesOrderId>
{
    public TenantId TenantId { get; private set; }
    public string OrderNumber { get; private set; }
    public DateOnly OrderDate { get; private set; }
    public DateOnly? ExpectedDeliveryDate { get; private set; }
    
    // Customer
    public LedgerId CustomerId { get; private set; }
    public string CustomerName { get; private set; }
    public Address? BillingAddress { get; private set; }
    public Address? ShippingAddress { get; private set; }
    
    // Reference
    public SalesQuotationId? QuotationId { get; private set; }
    public string? CustomerPoNumber { get; private set; }
    public DateOnly? CustomerPoDate { get; private set; }
    
    // Lines
    private readonly List<SalesOrderLine> _lines = new();
    public IReadOnlyCollection<SalesOrderLine> Lines => _lines.AsReadOnly();
    
    // Fulfillment Status
    public OrderFulfillmentStatus FulfillmentStatus { get; private set; }
    public decimal PercentFulfilled => _lines.Average(l => l.PercentDelivered);
    
    // Related Documents
    private readonly List<Guid> _deliveryChallanIds = new();
    private readonly List<Guid> _invoiceIds = new();
    
    public SalesOrderLine GetLine(ItemId itemId)
    {
        return _lines.FirstOrDefault(l => l.ItemId == itemId);
    }
    
    public void UpdateFulfillmentStatus()
    {
        var totalQty = _lines.Sum(l => l.Quantity);
        var deliveredQty = _lines.Sum(l => l.DeliveredQuantity);
        
        FulfillmentStatus = deliveredQty == 0 
            ? OrderFulfillmentStatus.Pending
            : deliveredQty >= totalQty 
                ? OrderFulfillmentStatus.FullyDelivered 
                : OrderFulfillmentStatus.PartiallyDelivered;
    }
}

// Domain/Entities/DeliveryChallan.cs
public class DeliveryChallan : AggregateRoot<DeliveryChallanId>
{
    public TenantId TenantId { get; private set; }
    public string ChallanNumber { get; private set; }
    public DateOnly ChallanDate { get; private set; }
    
    // Customer
    public LedgerId CustomerId { get; private set; }
    public Address ShippingAddress { get; private set; }
    
    // Transport
    public string? TransporterName { get; private set; }
    public string? VehicleNumber { get; private set; }
    public string? LrNumber { get; private set; }
    public DateOnly? LrDate { get; private set; }
    
    // Reference
    public SalesOrderId? SalesOrderId { get; private set; }
    
    // Lines
    private readonly List<DeliveryChallanLine> _lines = new();
    public IReadOnlyCollection<DeliveryChallanLine> Lines => _lines.AsReadOnly();
    
    // Stock Transaction created by this challan
    public Guid? StockTransactionId { get; private set; }
    
    // E-Way Bill
    public string? EwayBillNumber { get; private set; }
    public DateTime? EwayBillDate { get; private set; }
    
    // Status
    public ChallanStatus Status { get; private set; }
    public Guid? ConvertedToInvoiceId { get; private set; }
}

// Domain/Entities/SalesInvoice.cs
public class SalesInvoice : AggregateRoot<SalesInvoiceId>
{
    public TenantId TenantId { get; private set; }
    public FinancialYearId FinancialYearId { get; private set; }
    
    public string InvoiceNumber { get; private set; }
    public DateOnly InvoiceDate { get; private set; }
    public DateOnly DueDate { get; private set; }
    
    // Customer
    public LedgerId CustomerId { get; private set; }
    public string CustomerName { get; private set; }
    public Address BillingAddress { get; private set; }
    public Address? ShippingAddress { get; private set; }
    
    // GST
    public string? CustomerGstin { get; private set; }
    public string PlaceOfSupply { get; private set; }
    public bool IsInterState { get; private set; }
    public bool IsReverseCharge { get; private set; }
    
    // Reference Documents
    public SalesOrderId? SalesOrderId { get; private set; }
    public DeliveryChallanId? DeliveryChallanId { get; private set; }
    
    // Lines
    private readonly List<SalesInvoiceLine> _lines = new();
    public IReadOnlyCollection<SalesInvoiceLine> Lines => _lines.AsReadOnly();
    
    // Amounts
    public Money SubTotal { get; private set; }
    public Money DiscountAmount { get; private set; }
    public Money TaxableValue { get; private set; }
    public Money CgstAmount { get; private set; }
    public Money SgstAmount { get; private set; }
    public Money IgstAmount { get; private set; }
    public Money CessAmount { get; private set; }
    public Money RoundOff { get; private set; }
    public Money TotalAmount { get; private set; }
    
    // Payment Terms
    public int CreditDays { get; private set; }
    
    // Status
    public InvoiceStatus Status { get; private set; }
    public Money AmountPaid { get; private set; }
    public Money AmountDue => TotalAmount.Subtract(AmountPaid);
    public PaymentStatus PaymentStatus { get; private set; }
    
    // Linked Entities
    public Guid? AccountingVoucherId { get; private set; }
    public Guid? StockTransactionId { get; private set; }
    public Guid? EInvoiceId { get; private set; }
    public string? Irn { get; private set; }
    
    public void CalculateTotals()
    {
        SubTotal = Money.INR(_lines.Sum(l => l.Amount.Amount));
        
        // Apply invoice-level discount
        TaxableValue = SubTotal.Subtract(DiscountAmount);
        
        // Calculate taxes
        if (IsInterState)
        {
            IgstAmount = Money.INR(_lines.Sum(l => l.IgstAmount.Amount));
            CgstAmount = Money.Zero;
            SgstAmount = Money.Zero;
        }
        else
        {
            CgstAmount = Money.INR(_lines.Sum(l => l.CgstAmount.Amount));
            SgstAmount = Money.INR(_lines.Sum(l => l.SgstAmount.Amount));
            IgstAmount = Money.Zero;
        }
        
        CessAmount = Money.INR(_lines.Sum(l => l.CessAmount.Amount));
        
        var grossTotal = TaxableValue.Add(CgstAmount).Add(SgstAmount).Add(IgstAmount).Add(CessAmount);
        
        // Round off
        var rounded = Math.Round(grossTotal.Amount, 0, MidpointRounding.AwayFromZero);
        RoundOff = Money.INR(rounded - grossTotal.Amount);
        TotalAmount = Money.INR(rounded);
    }
    
    public Result Post()
    {
        if (Status != InvoiceStatus.Draft)
            return Result.Failure("Invoice is already posted");
        
        CalculateTotals();
        Status = InvoiceStatus.Posted;
        
        AddDomainEvent(new SalesInvoicePostedEvent(this));
        
        return Result.Success();
    }
    
    public void RecordPayment(Money amount, Guid receiptVoucherId)
    {
        AmountPaid = AmountPaid.Add(amount);
        
        if (AmountPaid.Amount >= TotalAmount.Amount)
            PaymentStatus = PaymentStatus.Paid;
        else if (AmountPaid.Amount > 0)
            PaymentStatus = PaymentStatus.PartiallyPaid;
        
        AddDomainEvent(new PaymentReceivedEvent(Id, amount, receiptVoucherId));
    }
}

// Domain/Entities/SalesInvoiceLine.cs
public class SalesInvoiceLine : Entity<SalesInvoiceLineId>
{
    public SalesInvoiceId InvoiceId { get; private set; }
    public int LineNumber { get; private set; }
    
    // Item
    public ItemId ItemId { get; private set; }
    public string ItemName { get; private set; }
    public string HsnCode { get; private set; }
    public string Uom { get; private set; }
    
    // Quantity & Price
    public decimal Quantity { get; private set; }
    public Money Rate { get; private set; }
    public Money Amount => Rate.Multiply(Quantity);
    
    // Discount
    public decimal DiscountPercent { get; private set; }
    public Money DiscountAmount { get; private set; }
    public Money TaxableValue => Amount.Subtract(DiscountAmount);
    
    // Tax
    public decimal GstRate { get; private set; }
    public Money CgstAmount { get; private set; }
    public Money SgstAmount { get; private set; }
    public Money IgstAmount { get; private set; }
    public decimal CessRate { get; private set; }
    public Money CessAmount { get; private set; }
    public Money TaxAmount => CgstAmount.Add(SgstAmount).Add(IgstAmount).Add(CessAmount);
    
    // Total
    public Money TotalValue => TaxableValue.Add(TaxAmount);
    
    // Batch (if applicable)
    public Guid? BatchId { get; private set; }
    public string? BatchNumber { get; private set; }
    
    // Godown
    public GodownId GodownId { get; private set; }
}
```

### **3.2 Purchase Documents**

```csharp
// Domain/Entities/PurchaseOrder.cs
public class PurchaseOrder : AggregateRoot<PurchaseOrderId>
{
    public TenantId TenantId { get; private set; }
    public string OrderNumber { get; private set; }
    public DateOnly OrderDate { get; private set; }
    public DateOnly ExpectedDeliveryDate { get; private set; }
    
    // Vendor
    public LedgerId VendorId { get; private set; }
    public string VendorName { get; private set; }
    public string? VendorGstin { get; private set; }
    public Address? VendorAddress { get; private set; }
    
    // Delivery
    public GodownId DeliveryGodownId { get; private set; }
    public Address DeliveryAddress { get; private set; }
    
    // Lines
    private readonly List<PurchaseOrderLine> _lines = new();
    public IReadOnlyCollection<PurchaseOrderLine> Lines => _lines.AsReadOnly();
    
    // Totals
    public Money TotalAmount { get; private set; }
    
    // Status
    public PurchaseOrderStatus Status { get; private set; }
    public OrderFulfillmentStatus FulfillmentStatus { get; private set; }
}

// Domain/Entities/GoodsReceiptNote.cs (GRN)
public class GoodsReceiptNote : AggregateRoot<GrnId>
{
    public TenantId TenantId { get; private set; }
    public string GrnNumber { get; private set; }
    public DateOnly GrnDate { get; private set; }
    
    // Vendor
    public LedgerId VendorId { get; private set; }
    public string VendorName { get; private set; }
    
    // Reference
    public PurchaseOrderId? PurchaseOrderId { get; private set; }
    public string? VendorChallanNumber { get; private set; }
    public DateOnly? VendorChallanDate { get; private set; }
    
    // Receiving
    public GodownId ReceivingGodownId { get; private set; }
    public UserId ReceivedBy { get; private set; }
    
    // Lines
    private readonly List<GrnLine> _lines = new();
    public IReadOnlyCollection<GrnLine> Lines => _lines.AsReadOnly();
    
    // Quality Check
    public bool RequiresQc { get; private set; }
    public QcStatus QcStatus { get; private set; }
    
    // Stock Transaction
    public Guid? StockTransactionId { get; private set; }
    
    // Status
    public GrnStatus Status { get; private set; }
    public Guid? ConvertedToBillId { get; private set; }
}

// Domain/Entities/PurchaseBill.cs
public class PurchaseBill : AggregateRoot<PurchaseBillId>
{
    public TenantId TenantId { get; private set; }
    public FinancialYearId FinancialYearId { get; private set; }
    
    // Bill Details
    public string BillNumber { get; private set; }  // Our internal number
    public DateOnly BillDate { get; private set; }
    public DateOnly DueDate { get; private set; }
    
    // Vendor Invoice Reference
    public string VendorInvoiceNumber { get; private set; }
    public DateOnly VendorInvoiceDate { get; private set; }
    
    // Vendor
    public LedgerId VendorId { get; private set; }
    public string VendorName { get; private set; }
    public string? VendorGstin { get; private set; }
    public Address VendorAddress { get; private set; }
    
    // GST
    public string PlaceOfSupply { get; private set; }
    public bool IsInterState { get; private set; }
    public bool IsReverseCharge { get; private set; }
    
    // Reference Documents
    public PurchaseOrderId? PurchaseOrderId { get; private set; }
    public GrnId? GrnId { get; private set; }
    
    // Lines
    private readonly List<PurchaseBillLine> _lines = new();
    public IReadOnlyCollection<PurchaseBillLine> Lines => _lines.AsReadOnly();
    
    // Amounts (similar to Sales Invoice)
    public Money TotalAmount { get; private set; }
    public Money AmountPaid { get; private set; }
    public Money AmountDue => TotalAmount.Subtract(AmountPaid);
    
    // TDS
    public bool IsTdsApplicable { get; private set; }
    public string? TdsSection { get; private set; }
    public decimal TdsRate { get; private set; }
    public Money TdsAmount { get; private set; }
    public Money NetPayable => TotalAmount.Subtract(TdsAmount);
    
    // Status
    public BillStatus Status { get; private set; }
    public PaymentStatus PaymentStatus { get; private set; }
    
    // Linked Entities
    public Guid? AccountingVoucherId { get; private set; }
}
```

### **3.3 Pricing Engine**

```csharp
// Domain/Entities/PriceList.cs
public class PriceList : AggregateRoot<PriceListId>
{
    public TenantId TenantId { get; private set; }
    public string Name { get; private set; }
    public PriceListType Type { get; private set; }  // Sales, Purchase
    public DateOnly EffectiveFrom { get; private set; }
    public DateOnly? EffectiveUntil { get; private set; }
    public bool IsDefault { get; private set; }
    
    // Prices
    private readonly List<PriceListItem> _items = new();
    public IReadOnlyCollection<PriceListItem> Items => _items.AsReadOnly();
}

public class PriceListItem : Entity<PriceListItemId>
{
    public PriceListId PriceListId { get; private set; }
    public ItemId ItemId { get; private set; }
    public Money Price { get; private set; }
    public Money? MinPrice { get; private set; }  // Minimum selling price
    public Money? MaxDiscount { get; private set; }
}

// Domain/Entities/DiscountScheme.cs
public class DiscountScheme : AggregateRoot<DiscountSchemeId>
{
    public TenantId TenantId { get; private set; }
    public string Name { get; private set; }
    public DiscountSchemeType Type { get; private set; }
    public DateOnly EffectiveFrom { get; private set; }
    public DateOnly EffectiveUntil { get; private set; }
    
    // Applicability
    public List<ItemId> ApplicableItemIds { get; private set; }
    public List<ItemCategoryId> ApplicableCategoryIds { get; private set; }
    public List<LedgerId> ApplicableCustomerIds { get; private set; }
    
    // Discount Rules
    private readonly List<DiscountSlab> _slabs = new();
    public IReadOnlyCollection<DiscountSlab> Slabs => _slabs.AsReadOnly();
}

public class DiscountSlab : ValueObject
{
    public decimal MinQuantity { get; init; }
    public decimal? MaxQuantity { get; init; }
    public decimal DiscountPercent { get; init; }
    public Money? FlatDiscount { get; init; }
}

// Domain/Services/PricingService.cs
public class PricingService : IPricingService
{
    public async Task<PricingResult> CalculatePriceAsync(
        ItemId itemId,
        LedgerId customerId,
        decimal quantity,
        DateOnly date,
        CancellationToken ct)
    {
        // 1. Get base price from price list
        var priceList = await GetApplicablePriceListAsync(customerId, date, ct);
        var basePrice = await GetItemPriceAsync(itemId, priceList, ct);
        
        // 2. Apply quantity-based discounts
        var discountSchemes = await GetApplicableDiscountSchemesAsync(itemId, customerId, date, ct);
        var discount = CalculateBestDiscount(discountSchemes, quantity);
        
        // 3. Calculate final price
        var finalPrice = basePrice.Multiply(1 - discount);
        
        return new PricingResult(basePrice, discount, finalPrice);
    }
}
```

---

## **4. Workflow Integration**

### **4.1 Sales Invoice to Accounting Voucher**

```csharp
// Application/EventHandlers/SalesInvoicePostedEventHandler.cs
public class SalesInvoicePostedEventHandler : INotificationHandler<SalesInvoicePostedEvent>
{
    private readonly IVoucherService _voucherService;
    private readonly IStockTransactionService _stockService;
    private readonly IEInvoiceService _eInvoiceService;
    
    public async Task Handle(SalesInvoicePostedEvent notification, CancellationToken ct)
    {
        var invoice = notification.Invoice;
        
        // 1. Create Accounting Voucher
        var voucherLines = new List<VoucherLineDto>
        {
            // Debit: Customer Account
            new VoucherLineDto
            {
                LedgerId = invoice.CustomerId.Value,
                Amount = invoice.TotalAmount.Amount,
                Type = BalanceType.Debit,
                Particulars = $"Sales Invoice {invoice.InvoiceNumber}"
            },
            
            // Credit: Sales Account (per item)
            // ...lines for each item's sales ledger
        };
        
        // Add tax ledger entries
        if (invoice.IsInterState)
        {
            voucherLines.Add(new VoucherLineDto
            {
                LedgerId = _systemLedgers.IgstOutputId,
                Amount = invoice.IgstAmount.Amount,
                Type = BalanceType.Credit
            });
        }
        else
        {
            voucherLines.Add(new VoucherLineDto
            {
                LedgerId = _systemLedgers.CgstOutputId,
                Amount = invoice.CgstAmount.Amount,
                Type = BalanceType.Credit
            });
            voucherLines.Add(new VoucherLineDto
            {
                LedgerId = _systemLedgers.SgstOutputId,
                Amount = invoice.SgstAmount.Amount,
                Type = BalanceType.Credit
            });
        }
        
        var voucherId = await _voucherService.CreateSalesVoucherAsync(
            voucherDate: invoice.InvoiceDate,
            voucherLines: voucherLines,
            sourceModule: "Sales",
            sourceDocumentId: invoice.Id.Value,
            ct: ct
        );
        
        invoice.LinkAccountingVoucher(voucherId);
        
        // 2. Create Stock Transaction (if items present)
        if (invoice.Lines.Any(l => l.ItemId != null))
        {
            var stockLines = invoice.Lines
                .Where(l => l.ItemId != null)
                .Select(l => new StockLineDto
                {
                    ItemId = l.ItemId.Value,
                    GodownId = l.GodownId.Value,
                    Quantity = l.Quantity,
                    Direction = StockDirection.Out,
                    BatchId = l.BatchId
                }).ToList();
            
            var stockTxnId = await _stockService.RecordStockAsync(
                type: StockTransactionType.Sales,
                date: invoice.InvoiceDate,
                lines: stockLines,
                sourceModule: "Sales",
                sourceDocumentId: invoice.Id.Value,
                linkedVoucherId: voucherId,
                ct: ct
            );
            
            invoice.LinkStockTransaction(stockTxnId);
        }
        
        // 3. Auto-generate E-Invoice if applicable
        if (await _eInvoiceService.IsEInvoiceMandatoryAsync(invoice, ct))
        {
            await _eInvoiceService.QueueEInvoiceGenerationAsync(invoice.Id.Value, ct);
        }
    }
}
```

---

## **5. API Endpoints**

```csharp
// API/Controllers/SalesController.cs
[ApiController]
[Route("api/v1/sales")]
[Authorize]
public class SalesController : ControllerBase
{
    // Quotations
    [HttpPost("quotations")]
    public async Task<ActionResult<SalesQuotationId>> CreateQuotation([FromBody] CreateQuotationCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("quotations/{id}/convert-to-order")]
    public async Task<ActionResult<SalesOrderId>> ConvertToOrder(Guid id)
    {
        return Ok(await _mediator.Send(new ConvertQuotationToOrderCommand { QuotationId = id }));
    }
    
    // Orders
    [HttpGet("orders")]
    public async Task<ActionResult<PagedResult<SalesOrderDto>>> GetOrders(
        [FromQuery] OrderStatus? status,
        [FromQuery] DateOnly? fromDate,
        [FromQuery] DateOnly? toDate)
    {
        return Ok(await _mediator.Send(new GetSalesOrdersQuery { /* ... */ }));
    }
    
    [HttpGet("orders/{id}/pending-delivery")]
    public async Task<ActionResult<PendingDeliveryDto>> GetPendingDelivery(Guid id)
    {
        // Get items pending delivery for this order
        return Ok(await _mediator.Send(new GetPendingDeliveryQuery { OrderId = id }));
    }
    
    // Delivery Challans
    [HttpPost("challans")]
    public async Task<ActionResult<DeliveryChallanId>> CreateChallan([FromBody] CreateDeliveryChallanCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    // Invoices
    [HttpPost("invoices")]
    [RequirePermission(Permissions.Sales.CreateInvoice)]
    public async Task<ActionResult<SalesInvoiceId>> CreateInvoice([FromBody] CreateSalesInvoiceCommand command)
    {
        var result = await _mediator.Send(command);
        if (result.IsFailure)
            return BadRequest(result.Errors);
        return CreatedAtAction(nameof(GetInvoice), new { id = result.Value }, result.Value);
    }
    
    [HttpGet("invoices/{id}")]
    public async Task<ActionResult<SalesInvoiceDto>> GetInvoice(Guid id)
    {
        return Ok(await _mediator.Send(new GetSalesInvoiceQuery { Id = id }));
    }
    
    [HttpGet("invoices/{id}/pdf")]
    public async Task<IActionResult> GetInvoicePdf(Guid id)
    {
        var pdf = await _mediator.Send(new GenerateInvoicePdfQuery { Id = id });
        return File(pdf, "application/pdf", $"Invoice_{id}.pdf");
    }
    
    [HttpPost("invoices/{id}/post")]
    [RequirePermission(Permissions.Sales.PostInvoice)]
    public async Task<ActionResult> PostInvoice(Guid id)
    {
        var result = await _mediator.Send(new PostSalesInvoiceCommand { Id = id });
        if (result.IsFailure)
            return BadRequest(result.Errors);
        return Ok(new { Message = "Invoice posted successfully" });
    }
    
    // Credit/Debit Notes
    [HttpPost("credit-notes")]
    public async Task<ActionResult<CreditNoteId>> CreateCreditNote([FromBody] CreateCreditNoteCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    // Outstanding
    [HttpGet("outstanding")]
    public async Task<ActionResult<List<OutstandingDto>>> GetOutstanding([FromQuery] Guid? customerId)
    {
        return Ok(await _mediator.Send(new GetSalesOutstandingQuery { CustomerId = customerId }));
    }
    
    // Reports
    [HttpGet("reports/sales-register")]
    public async Task<ActionResult<SalesRegisterDto>> GetSalesRegister(
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate)
    {
        return Ok(await _mediator.Send(new GetSalesRegisterQuery { FromDate = fromDate, ToDate = toDate }));
    }
}

// API/Controllers/PurchaseController.cs
[ApiController]
[Route("api/v1/purchase")]
[Authorize]
public class PurchaseController : ControllerBase
{
    // Purchase Orders
    [HttpPost("orders")]
    public async Task<ActionResult<PurchaseOrderId>> CreateOrder([FromBody] CreatePurchaseOrderCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    // GRN
    [HttpPost("grn")]
    public async Task<ActionResult<GrnId>> CreateGrn([FromBody] CreateGrnCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpGet("grn/pending-from-order/{orderId}")]
    public async Task<ActionResult<PendingGrnDto>> GetPendingFromOrder(Guid orderId)
    {
        return Ok(await _mediator.Send(new GetPendingGrnQuery { PurchaseOrderId = orderId }));
    }
    
    // Bills
    [HttpPost("bills")]
    [RequirePermission(Permissions.Purchase.CreateBill)]
    public async Task<ActionResult<PurchaseBillId>> CreateBill([FromBody] CreatePurchaseBillCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpGet("bills/{id}")]
    public async Task<ActionResult<PurchaseBillDto>> GetBill(Guid id)
    {
        return Ok(await _mediator.Send(new GetPurchaseBillQuery { Id = id }));
    }
    
    // Outstanding (Payables)
    [HttpGet("outstanding")]
    public async Task<ActionResult<List<PayableDto>>> GetPayables([FromQuery] Guid? vendorId)
    {
        return Ok(await _mediator.Send(new GetPurchaseOutstandingQuery { VendorId = vendorId }));
    }
}
```

---

## **6. Frontend - Invoice Entry**

```typescript
// src/app/(dashboard)/sales/invoices/new/page.tsx
'use client';

import { useState, useCallback } from 'react';
import { useForm, useFieldArray } from 'react-hook-form';
import { useQuery, useMutation } from '@tanstack/react-query';
import { CustomerSearch } from '@/components/sales/customer-search';
import { ItemSearch } from '@/components/inventory/item-search';
import { MoneyInput } from '@/components/ui/money-input';
import { useHotkeys } from 'react-hotkeys-hook';

export default function NewSalesInvoicePage() {
  const { register, control, watch, setValue, handleSubmit } = useForm({
    defaultValues: {
      invoiceDate: new Date().toISOString().split('T')[0],
      customerId: '',
      customerGstin: '',
      placeOfSupply: '',
      lines: [{ itemId: '', quantity: 1, rate: 0, discount: 0, gstRate: 0 }],
    }
  });
  
  const { fields, append, remove } = useFieldArray({ control, name: 'lines' });
  const lines = watch('lines');
  
  // Calculate totals
  const totals = useMemo(() => {
    let subTotal = 0;
    let taxTotal = 0;
    
    lines.forEach(line => {
      const amount = line.quantity * line.rate;
      const discountAmt = amount * (line.discount / 100);
      const taxable = amount - discountAmt;
      const tax = taxable * (line.gstRate / 100);
      
      subTotal += taxable;
      taxTotal += tax;
    });
    
    const grandTotal = Math.round(subTotal + taxTotal);
    const roundOff = grandTotal - (subTotal + taxTotal);
    
    return { subTotal, taxTotal, grandTotal, roundOff };
  }, [lines]);
  
  // Customer selection auto-fills GSTIN and Place of Supply
  const handleCustomerSelect = (customer: any) => {
    setValue('customerId', customer.id);
    setValue('customerGstin', customer.gstin);
    setValue('placeOfSupply', customer.stateCode);
  };
  
  // Item selection auto-fills rate and GST
  const handleItemSelect = (index: number, item: any) => {
    setValue(`lines.${index}.itemId`, item.id);
    setValue(`lines.${index}.rate`, item.sellingPrice);
    setValue(`lines.${index}.gstRate`, item.gstRate);
  };
  
  // Keyboard shortcuts
  useHotkeys('ctrl+n', () => append({ itemId: '', quantity: 1, rate: 0, discount: 0, gstRate: 0 }));
  useHotkeys('ctrl+enter', () => handleSubmit(onSubmit)());
  
  const onSubmit = async (data: any) => {
    // Submit to API
  };
  
  return (
    <div className="p-6 max-w-6xl mx-auto">
      <h1 className="text-2xl font-bold mb-6">New Sales Invoice</h1>
      
      <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
        {/* Header */}
        <div className="grid grid-cols-4 gap-4 p-4 bg-white rounded-lg shadow">
          <div>
            <label className="block text-sm font-medium mb-1">Invoice Date</label>
            <input type="date" {...register('invoiceDate')} className="w-full border rounded px-3 py-2" />
          </div>
          <div className="col-span-2">
            <label className="block text-sm font-medium mb-1">Customer</label>
            <CustomerSearch onSelect={handleCustomerSelect} />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1">GSTIN</label>
            <input {...register('customerGstin')} className="w-full border rounded px-3 py-2" readOnly />
          </div>
        </div>
        
        {/* Line Items */}
        <div className="bg-white rounded-lg shadow overflow-hidden">
          <table className="w-full">
            <thead className="bg-gray-50">
              <tr>
                <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Item</th>
                <th className="px-4 py-3 text-right text-xs font-medium text-gray-500 uppercase w-24">Qty</th>
                <th className="px-4 py-3 text-right text-xs font-medium text-gray-500 uppercase w-32">Rate</th>
                <th className="px-4 py-3 text-right text-xs font-medium text-gray-500 uppercase w-24">Disc %</th>
                <th className="px-4 py-3 text-right text-xs font-medium text-gray-500 uppercase w-24">GST %</th>
                <th className="px-4 py-3 text-right text-xs font-medium text-gray-500 uppercase w-32">Amount</th>
                <th className="w-10"></th>
              </tr>
            </thead>
            <tbody className="divide-y">
              {fields.map((field, index) => (
                <InvoiceLineRow
                  key={field.id}
                  index={index}
                  register={register}
                  control={control}
                  onItemSelect={(item) => handleItemSelect(index, item)}
                  onRemove={() => remove(index)}
                />
              ))}
            </tbody>
          </table>
          
          <div className="p-4 bg-gray-50 flex justify-between">
            <button type="button" onClick={() => append({ itemId: '', quantity: 1, rate: 0, discount: 0, gstRate: 0 })} className="text-blue-600">
              + Add Line (Ctrl+N)
            </button>
          </div>
        </div>
        
        {/* Totals */}
        <div className="flex justify-end">
          <div className="w-80 bg-white rounded-lg shadow p-4 space-y-2">
            <div className="flex justify-between">
              <span>Sub Total:</span>
              <span>₹{totals.subTotal.toLocaleString('en-IN', { minimumFractionDigits: 2 })}</span>
            </div>
            <div className="flex justify-between">
              <span>Tax:</span>
              <span>₹{totals.taxTotal.toLocaleString('en-IN', { minimumFractionDigits: 2 })}</span>
            </div>
            {totals.roundOff !== 0 && (
              <div className="flex justify-between text-sm text-gray-500">
                <span>Round Off:</span>
                <span>₹{totals.roundOff.toLocaleString('en-IN', { minimumFractionDigits: 2 })}</span>
              </div>
            )}
            <div className="flex justify-between font-bold text-lg border-t pt-2">
              <span>Grand Total:</span>
              <span>₹{totals.grandTotal.toLocaleString('en-IN')}</span>
            </div>
          </div>
        </div>
        
        {/* Actions */}
        <div className="flex justify-end gap-4">
          <button type="button" className="px-6 py-2 border rounded">Save as Draft</button>
          <button type="submit" className="px-6 py-2 bg-blue-600 text-white rounded">Save & Post (Ctrl+Enter)</button>
        </div>
      </form>
    </div>
  );
}
```

---

## **7. Database Schema**

```sql
-- Sales & Purchase Module Tables

-- Price Lists
CREATE TABLE price_lists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    type VARCHAR(20) NOT NULL CHECK (type IN ('Sales', 'Purchase')),
    effective_from DATE NOT NULL,
    effective_until DATE,
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE price_list_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    price_list_id UUID NOT NULL REFERENCES price_lists(id) ON DELETE CASCADE,
    item_id UUID NOT NULL REFERENCES items(id),
    price DECIMAL(18,4) NOT NULL,
    min_price DECIMAL(18,4),
    max_discount DECIMAL(18,4),
    
    CONSTRAINT unique_item_per_list UNIQUE (price_list_id, item_id)
);

-- Sales Documents
CREATE TABLE sales_quotations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    quotation_number VARCHAR(50) NOT NULL,
    quotation_date DATE NOT NULL,
    valid_until DATE NOT NULL,
    
    customer_id UUID NOT NULL REFERENCES ledgers(id),
    customer_name VARCHAR(200) NOT NULL,
    customer_gstin VARCHAR(15),
    billing_address JSONB,
    shipping_address JSONB,
    place_of_supply VARCHAR(2),
    is_inter_state BOOLEAN DEFAULT FALSE,
    
    sub_total DECIMAL(18,2) DEFAULT 0,
    tax_amount DECIMAL(18,2) DEFAULT 0,
    total_amount DECIMAL(18,2) DEFAULT 0,
    
    terms_and_conditions TEXT,
    notes TEXT,
    
    status VARCHAR(20) DEFAULT 'Draft',
    converted_to_order_id UUID,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    created_by UUID NOT NULL REFERENCES users(id)
);

CREATE TABLE sales_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    order_number VARCHAR(50) NOT NULL,
    order_date DATE NOT NULL,
    expected_delivery_date DATE,
    
    customer_id UUID NOT NULL REFERENCES ledgers(id),
    customer_name VARCHAR(200) NOT NULL,
    customer_gstin VARCHAR(15),
    billing_address JSONB,
    shipping_address JSONB,
    
    quotation_id UUID REFERENCES sales_quotations(id),
    customer_po_number VARCHAR(50),
    customer_po_date DATE,
    
    total_amount DECIMAL(18,2) DEFAULT 0,
    
    status VARCHAR(20) DEFAULT 'Open',
    fulfillment_status VARCHAR(20) DEFAULT 'Pending',
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sales_invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    financial_year_id UUID NOT NULL REFERENCES financial_years(id),
    
    invoice_number VARCHAR(50) NOT NULL,
    invoice_date DATE NOT NULL,
    due_date DATE NOT NULL,
    
    customer_id UUID NOT NULL REFERENCES ledgers(id),
    customer_name VARCHAR(200) NOT NULL,
    customer_gstin VARCHAR(15),
    billing_address JSONB NOT NULL,
    shipping_address JSONB,
    
    place_of_supply VARCHAR(2) NOT NULL,
    is_inter_state BOOLEAN DEFAULT FALSE,
    is_reverse_charge BOOLEAN DEFAULT FALSE,
    
    sales_order_id UUID REFERENCES sales_orders(id),
    delivery_challan_id UUID,
    
    sub_total DECIMAL(18,2) NOT NULL DEFAULT 0,
    discount_amount DECIMAL(18,2) DEFAULT 0,
    taxable_value DECIMAL(18,2) NOT NULL DEFAULT 0,
    cgst_amount DECIMAL(18,2) DEFAULT 0,
    sgst_amount DECIMAL(18,2) DEFAULT 0,
    igst_amount DECIMAL(18,2) DEFAULT 0,
    cess_amount DECIMAL(18,2) DEFAULT 0,
    round_off DECIMAL(18,2) DEFAULT 0,
    total_amount DECIMAL(18,2) NOT NULL DEFAULT 0,
    
    credit_days INT DEFAULT 0,
    
    status VARCHAR(20) DEFAULT 'Draft',
    amount_paid DECIMAL(18,2) DEFAULT 0,
    payment_status VARCHAR(20) DEFAULT 'Unpaid',
    
    accounting_voucher_id UUID REFERENCES vouchers(id),
    stock_transaction_id UUID REFERENCES stock_transactions(id),
    e_invoice_id UUID REFERENCES e_invoices(id),
    irn VARCHAR(64),
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    created_by UUID NOT NULL REFERENCES users(id),
    posted_at TIMESTAMPTZ,
    posted_by UUID REFERENCES users(id),
    
    CONSTRAINT unique_invoice_number UNIQUE (tenant_id, financial_year_id, invoice_number)
);

CREATE TABLE sales_invoice_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES sales_invoices(id) ON DELETE CASCADE,
    line_number INT NOT NULL,
    
    item_id UUID REFERENCES items(id),
    item_name VARCHAR(200) NOT NULL,
    hsn_code VARCHAR(8) NOT NULL,
    uom VARCHAR(20),
    
    quantity DECIMAL(18,4) NOT NULL,
    rate DECIMAL(18,4) NOT NULL,
    amount DECIMAL(18,2) NOT NULL,
    
    discount_percent DECIMAL(5,2) DEFAULT 0,
    discount_amount DECIMAL(18,2) DEFAULT 0,
    taxable_value DECIMAL(18,2) NOT NULL,
    
    gst_rate DECIMAL(5,2) NOT NULL,
    cgst_amount DECIMAL(18,2) DEFAULT 0,
    sgst_amount DECIMAL(18,2) DEFAULT 0,
    igst_amount DECIMAL(18,2) DEFAULT 0,
    cess_rate DECIMAL(5,2) DEFAULT 0,
    cess_amount DECIMAL(18,2) DEFAULT 0,
    
    total_value DECIMAL(18,2) NOT NULL,
    
    godown_id UUID REFERENCES godowns(id),
    batch_id UUID REFERENCES batches(id),
    batch_number VARCHAR(50),
    
    CONSTRAINT unique_line_per_invoice UNIQUE (invoice_id, line_number)
);

-- Similar tables for Purchase (purchase_orders, grn, purchase_bills, etc.)

-- Indexes
CREATE INDEX idx_sales_invoices_customer ON sales_invoices(tenant_id, customer_id);
CREATE INDEX idx_sales_invoices_date ON sales_invoices(tenant_id, invoice_date);
CREATE INDEX idx_sales_invoices_status ON sales_invoices(tenant_id, status, payment_status);
```

---

## **8. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| Quotation to Order | Convert with single click, preserve all data |
| Order to Invoice | Partial invoicing supported |
| Invoice Creation | < 2 seconds including tax calculation |
| Auto Voucher | Create accounting voucher on invoice post |
| Auto Stock | Update stock on invoice post |
| E-Invoice Queue | Auto-queue E-Invoice generation |
| PDF Generation | Generate invoice PDF in < 1 second |
| Outstanding | Show customer-wise receivables/payables |

---

## **9. Definition of Done**

- [ ] Complete sales workflow (Quote → Order → Challan → Invoice)
- [ ] Complete purchase workflow (PO → GRN → Bill)
- [ ] Price list and discount scheme support
- [ ] Auto-creation of accounting vouchers
- [ ] Auto-creation of stock transactions
- [ ] Credit/Debit note handling
- [ ] Outstanding reports (receivables/payables)
- [ ] Invoice PDF with GST-compliant format
- [ ] Integration tests for complete workflows
- [ ] Performance benchmarks met

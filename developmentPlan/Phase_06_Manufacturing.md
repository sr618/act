# **Phase 6: Manufacturing Module**
**Duration:** 10 Weeks  
**Team:** Backend (3), Frontend (2), QA (1)  
**Dependencies:** Phase 1 (Core Accounting), Phase 3 (Inventory)  

---

## **1. Overview**

The Manufacturing Module provides:
- Bill of Materials (BOM) with multi-level structures
- Production planning and orders
- Job Work management (GST compliant)
- Work-in-Progress (WIP) tracking
- Scrap and by-product management
- Form ITC-04 generation

---

## **2. Key Features**

### **2.1 Bill of Materials (BOM)**
- Multi-level BOM (nested assemblies)
- Version control for BOMs
- Scrap percentage per component
- Alternative materials
- Cost rollup calculation

### **2.2 Production Orders**
- Create from sales orders or manual
- Material requisition and issue
- Production stages/operations
- Quality checkpoints
- Output with variances

### **2.3 Job Work (Indian GST Requirement)**
- Job Work Out: Send materials to contractors
- Job Work In: Receive work from customers
- Challan-based tracking
- 180-day return compliance
- Form ITC-04 quarterly filing

---

## **3. Domain Model**

```csharp
// Domain/Entities/BillOfMaterial.cs
public class BillOfMaterial : AggregateRoot<BomId>
{
    public TenantId TenantId { get; private set; }
    public ItemId FinishedGoodId { get; private set; }  // The manufactured item
    
    public string BomCode { get; private set; }
    public string BomName { get; private set; }
    public int Version { get; private set; }
    public BomStatus Status { get; private set; }
    
    public decimal StandardBatchSize { get; private set; }
    public UnitOfMeasureId UnitId { get; private set; }
    
    // Cost
    public Money EstimatedMaterialCost { get; private set; }
    public Money EstimatedLabourCost { get; private set; }
    public Money EstimatedOverheadCost { get; private set; }
    public Money TotalEstimatedCost => EstimatedMaterialCost
        .Add(EstimatedLabourCost)
        .Add(EstimatedOverheadCost);
    
    // Components
    private readonly List<BomComponent> _components = new();
    public IReadOnlyCollection<BomComponent> Components => _components.AsReadOnly();
    
    // Operations (manufacturing steps)
    private readonly List<BomOperation> _operations = new();
    public IReadOnlyCollection<BomOperation> Operations => _operations.AsReadOnly();
    
    // By-products and scrap
    private readonly List<BomByProduct> _byProducts = new();
    public IReadOnlyCollection<BomByProduct> ByProducts => _byProducts.AsReadOnly();
    
    public void AddComponent(ItemId itemId, decimal quantity, decimal scrapPercentage)
    {
        var component = new BomComponent(itemId, quantity, scrapPercentage);
        _components.Add(component);
        RecalculateCosts();
    }
}

// Domain/Entities/BomComponent.cs
public class BomComponent : Entity<BomComponentId>
{
    public BomId BomId { get; private set; }
    public ItemId ItemId { get; private set; }
    
    public decimal Quantity { get; private set; }  // Per unit of finished good
    public UnitOfMeasureId UnitId { get; private set; }
    
    public decimal ScrapPercentage { get; private set; }
    public decimal EffectiveQuantity => Quantity * (1 + ScrapPercentage / 100);
    
    // For sub-assemblies
    public BomId? SubBomId { get; private set; }  // If component is also manufactured
    public bool IsSubAssembly => SubBomId != null;
    
    // Alternative components
    public ItemId? AlternativeItemId { get; private set; }
    public decimal AlternativeRatio { get; private set; }
}

// Domain/Entities/ProductionOrder.cs
public class ProductionOrder : AggregateRoot<ProductionOrderId>
{
    public TenantId TenantId { get; private set; }
    public BomId BomId { get; private set; }
    public ItemId FinishedGoodId { get; private set; }
    
    public string OrderNumber { get; private set; }
    public DateOnly PlannedStartDate { get; private set; }
    public DateOnly PlannedEndDate { get; private set; }
    
    public decimal PlannedQuantity { get; private set; }
    public decimal CompletedQuantity { get; private set; }
    public decimal ScrapQuantity { get; private set; }
    
    public ProductionOrderStatus Status { get; private set; }
    public GodownId OutputGodownId { get; private set; }
    
    // Linked Sales Order (if any)
    public SalesOrderId? SalesOrderId { get; private set; }
    
    // Material Requisitions
    private readonly List<MaterialRequisition> _requisitions = new();
    public IReadOnlyCollection<MaterialRequisition> Requisitions => _requisitions.AsReadOnly();
    
    // Production Stages
    private readonly List<ProductionStage> _stages = new();
    public IReadOnlyCollection<ProductionStage> Stages => _stages.AsReadOnly();
    
    public void StartProduction()
    {
        if (Status != ProductionOrderStatus.Planned)
            throw new DomainException("Can only start a planned production order");
        
        Status = ProductionOrderStatus.InProgress;
        AddDomainEvent(new ProductionOrderStartedEvent(Id));
    }
    
    public void RecordOutput(decimal quantity, decimal scrapQty, GodownId godownId)
    {
        CompletedQuantity += quantity;
        ScrapQuantity += scrapQty;
        
        if (CompletedQuantity >= PlannedQuantity)
        {
            Status = ProductionOrderStatus.Completed;
            AddDomainEvent(new ProductionOrderCompletedEvent(Id, CompletedQuantity, ScrapQuantity));
        }
    }
}

public enum ProductionOrderStatus
{
    Draft = 1,
    Planned = 2,
    InProgress = 3,
    Completed = 4,
    Cancelled = 5
}

// Domain/Entities/MaterialRequisition.cs
public class MaterialRequisition : Entity<MaterialRequisitionId>
{
    public ProductionOrderId ProductionOrderId { get; private set; }
    public string RequisitionNumber { get; private set; }
    public DateOnly RequisitionDate { get; private set; }
    
    public MaterialRequisitionStatus Status { get; private set; }
    
    private readonly List<MaterialRequisitionLine> _lines = new();
    public IReadOnlyCollection<MaterialRequisitionLine> Lines => _lines.AsReadOnly();
    
    public void Issue(GodownId fromGodownId)
    {
        foreach (var line in _lines.Where(l => l.Status == LineStatus.Pending))
        {
            // Create stock transaction
            line.MarkAsIssued(fromGodownId);
        }
        Status = MaterialRequisitionStatus.Issued;
        AddDomainEvent(new MaterialIssuedEvent(ProductionOrderId, _lines.ToList()));
    }
}
```

---

## **4. Job Work Module (GST Compliance)**

```csharp
// Domain/Entities/JobWorkChallan.cs
public class JobWorkChallan : AggregateRoot<JobWorkChallanId>
{
    public TenantId TenantId { get; private set; }
    public JobWorkType Type { get; private set; }  // Out or In
    
    public string ChallanNumber { get; private set; }
    public DateOnly ChallanDate { get; private set; }
    
    // Job Worker (contractor)
    public LedgerId JobWorkerId { get; private set; }
    public string JobWorkerGstin { get; private set; }
    
    // Compliance tracking
    public DateOnly DueReturnDate => ChallanDate.AddDays(180);  // GST: 180 days limit
    public int DaysRemaining => (DueReturnDate.ToDateTime(TimeOnly.MinValue) - DateTime.Today).Days;
    public bool IsOverdue => DaysRemaining < 0;
    
    public JobWorkChallanStatus Status { get; private set; }
    
    private readonly List<JobWorkChallanItem> _items = new();
    public IReadOnlyCollection<JobWorkChallanItem> Items => _items.AsReadOnly();
    
    // Return tracking
    private readonly List<JobWorkReturn> _returns = new();
    public IReadOnlyCollection<JobWorkReturn> Returns => _returns.AsReadOnly();
    
    public decimal TotalSentQuantity => _items.Sum(i => i.SentQuantity);
    public decimal TotalReturnedQuantity => _returns.Sum(r => r.ReceivedQuantity);
    public decimal PendingQuantity => TotalSentQuantity - TotalReturnedQuantity;
    
    public void RecordReturn(DateOnly returnDate, List<(ItemId, decimal)> returnedItems)
    {
        var jobWorkReturn = new JobWorkReturn(returnDate, returnedItems);
        _returns.Add(jobWorkReturn);
        
        if (PendingQuantity <= 0)
        {
            Status = JobWorkChallanStatus.Closed;
        }
        
        AddDomainEvent(new JobWorkReturnRecordedEvent(Id, jobWorkReturn));
    }
}

public enum JobWorkType
{
    Out = 1,  // We send materials to job worker
    In = 2   // We receive materials for job work
}

public enum JobWorkChallanStatus
{
    Open = 1,
    PartiallyReturned = 2,
    Closed = 3,
    ConvertedToSale = 4  // If not returned, becomes deemed supply
}

// Domain/Entities/JobWorkChallanItem.cs
public class JobWorkChallanItem : Entity<JobWorkChallanItemId>
{
    public JobWorkChallanId ChallanId { get; private set; }
    public ItemId ItemId { get; private set; }
    
    public decimal SentQuantity { get; private set; }
    public UnitOfMeasureId UnitId { get; private set; }
    
    // For GST valuation
    public Money ValuePerUnit { get; private set; }
    public Money TotalValue => ValuePerUnit.Multiply(SentQuantity);
    
    // Tracking
    public decimal ReturnedQuantity { get; private set; }
    public decimal PendingQuantity => SentQuantity - ReturnedQuantity;
}
```

---

## **5. Form ITC-04 Generation**

```csharp
// Application/Commands/GenerateItc04Command.cs
public class GenerateItc04Command : IRequest<Itc04Response>
{
    public FinancialYearId FinancialYearId { get; set; }
    public Itc04Period Period { get; set; }  // Quarterly
}

public class GenerateItc04Handler : IRequestHandler<GenerateItc04Command, Itc04Response>
{
    public async Task<Itc04Response> Handle(GenerateItc04Command request, CancellationToken ct)
    {
        // Table 4: Goods sent to job worker
        var goodsSent = await _context.JobWorkChallans
            .Where(c => c.Type == JobWorkType.Out)
            .Where(c => c.ChallanDate >= request.Period.StartDate && c.ChallanDate <= request.Period.EndDate)
            .SelectMany(c => c.Items.Select(i => new Itc04TableRow
            {
                JobWorkerGstin = c.JobWorkerGstin,
                ChallanNumber = c.ChallanNumber,
                ChallanDate = c.ChallanDate,
                Description = i.Item.Name,
                Quantity = i.SentQuantity,
                UnitOfMeasure = i.Unit.Code,
                TaxableValue = i.TotalValue.Amount
            }))
            .ToListAsync(ct);
        
        // Table 5: Goods received back from job worker
        var goodsReceived = await GetGoodsReceivedAsync(request.Period);
        
        // Table 6: Goods sent to another job worker from job worker
        var goodsTransferred = await GetGoodsTransferredAsync(request.Period);
        
        return new Itc04Response
        {
            Period = request.Period,
            Table4_GoodsSent = goodsSent,
            Table5_GoodsReceived = goodsReceived,
            Table6_GoodsTransferred = goodsTransferred,
            GeneratedAt = DateTime.UtcNow
        };
    }
}
```

---

## **6. API Endpoints**

```csharp
// API/Controllers/ManufacturingController.cs
[ApiController]
[Route("api/v1/manufacturing")]
[Authorize]
[RequireModule(ModuleType.Manufacturing)]
public class ManufacturingController : ControllerBase
{
    // BOM
    [HttpGet("boms")]
    public async Task<ActionResult<List<BomDto>>> GetBoms([FromQuery] BomFilter filter)
    {
        return Ok(await _mediator.Send(new GetBomsQuery { Filter = filter }));
    }
    
    [HttpGet("boms/{bomId}/cost-breakdown")]
    public async Task<ActionResult<BomCostBreakdownDto>> GetBomCostBreakdown(Guid bomId)
    {
        return Ok(await _mediator.Send(new GetBomCostBreakdownQuery { BomId = bomId }));
    }
    
    [HttpPost("boms")]
    public async Task<ActionResult<BomId>> CreateBom([FromBody] CreateBomCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("boms/{bomId}/new-version")]
    public async Task<ActionResult<BomId>> CreateNewVersion(Guid bomId)
    {
        return Ok(await _mediator.Send(new CreateBomVersionCommand { SourceBomId = bomId }));
    }
    
    // Production Orders
    [HttpGet("production-orders")]
    public async Task<ActionResult<PagedList<ProductionOrderDto>>> GetProductionOrders(
        [FromQuery] ProductionOrderFilter filter)
    {
        return Ok(await _mediator.Send(new GetProductionOrdersQuery { Filter = filter }));
    }
    
    [HttpPost("production-orders")]
    public async Task<ActionResult<ProductionOrderId>> CreateProductionOrder(
        [FromBody] CreateProductionOrderCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("production-orders/{orderId}/start")]
    public async Task<ActionResult> StartProduction(Guid orderId)
    {
        return Ok(await _mediator.Send(new StartProductionCommand { OrderId = orderId }));
    }
    
    [HttpPost("production-orders/{orderId}/issue-materials")]
    public async Task<ActionResult> IssueMaterials(
        Guid orderId, 
        [FromBody] IssueMaterialsCommand command)
    {
        command.ProductionOrderId = orderId;
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("production-orders/{orderId}/record-output")]
    public async Task<ActionResult> RecordOutput(
        Guid orderId, 
        [FromBody] RecordProductionOutputCommand command)
    {
        command.ProductionOrderId = orderId;
        return Ok(await _mediator.Send(command));
    }
    
    // Job Work
    [HttpGet("job-work/challans")]
    public async Task<ActionResult<PagedList<JobWorkChallanDto>>> GetJobWorkChallans(
        [FromQuery] JobWorkChallanFilter filter)
    {
        return Ok(await _mediator.Send(new GetJobWorkChallansQuery { Filter = filter }));
    }
    
    [HttpGet("job-work/overdue")]
    public async Task<ActionResult<List<JobWorkChallanDto>>> GetOverdueChallans()
    {
        return Ok(await _mediator.Send(new GetOverdueJobWorkChallansQuery()));
    }
    
    [HttpPost("job-work/challans")]
    public async Task<ActionResult<JobWorkChallanId>> CreateChallan(
        [FromBody] CreateJobWorkChallanCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("job-work/challans/{challanId}/return")]
    public async Task<ActionResult> RecordReturn(
        Guid challanId, 
        [FromBody] RecordJobWorkReturnCommand command)
    {
        command.ChallanId = challanId;
        return Ok(await _mediator.Send(command));
    }
    
    // GST Forms
    [HttpGet("gst/itc-04")]
    public async Task<ActionResult<Itc04Response>> GenerateItc04(
        [FromQuery] FinancialYearId fyId,
        [FromQuery] Itc04Period period)
    {
        return Ok(await _mediator.Send(new GenerateItc04Command 
        { 
            FinancialYearId = fyId, 
            Period = period 
        }));
    }
}
```

---

## **7. Frontend - Production Order Dashboard**

```typescript
// src/app/(dashboard)/manufacturing/production/page.tsx
'use client';

import { useQuery } from '@tanstack/react-query';
import { Progress } from '@/components/ui/progress';
import { Badge } from '@/components/ui/badge';
import { Card, CardHeader, CardContent } from '@/components/ui/card';

export default function ProductionDashboard() {
  const { data: orders } = useQuery({
    queryKey: ['production-orders', { status: 'InProgress' }],
    queryFn: () => fetch('/api/v1/manufacturing/production-orders?status=InProgress').then(r => r.json())
  });
  
  const { data: overdueJobWork } = useQuery({
    queryKey: ['job-work-overdue'],
    queryFn: () => fetch('/api/v1/manufacturing/job-work/overdue').then(r => r.json())
  });
  
  return (
    <div className="p-6 space-y-6">
      <h1 className="text-2xl font-bold">Production Dashboard</h1>
      
      {/* Alerts */}
      {overdueJobWork?.length > 0 && (
        <div className="bg-red-50 border-l-4 border-red-500 p-4">
          <h3 className="text-red-800 font-medium">
            {overdueJobWork.length} Job Work Challans Overdue!
          </h3>
          <p className="text-red-700 text-sm">
            Materials not returned within 180 days will be treated as deemed supply under GST.
          </p>
        </div>
      )}
      
      {/* In-Progress Orders */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {orders?.items?.map((order: any) => (
          <Card key={order.id}>
            <CardHeader className="pb-2">
              <div className="flex justify-between items-start">
                <div>
                  <p className="text-sm text-gray-500">{order.orderNumber}</p>
                  <p className="font-medium">{order.finishedGood.name}</p>
                </div>
                <Badge variant={order.status === 'InProgress' ? 'default' : 'secondary'}>
                  {order.status}
                </Badge>
              </div>
            </CardHeader>
            <CardContent>
              <div className="space-y-4">
                {/* Progress */}
                <div>
                  <div className="flex justify-between text-sm mb-1">
                    <span>Progress</span>
                    <span>
                      {order.completedQuantity} / {order.plannedQuantity} {order.unit}
                    </span>
                  </div>
                  <Progress 
                    value={(order.completedQuantity / order.plannedQuantity) * 100} 
                  />
                </div>
                
                {/* Dates */}
                <div className="flex justify-between text-sm">
                  <span className="text-gray-500">Target Date</span>
                  <span className={order.isDelayed ? 'text-red-500' : ''}>
                    {order.plannedEndDate}
                  </span>
                </div>
                
                {/* Variance */}
                {order.scrapQuantity > 0 && (
                  <div className="text-sm text-orange-600">
                    Scrap: {order.scrapQuantity} {order.unit} 
                    ({((order.scrapQuantity / order.plannedQuantity) * 100).toFixed(1)}%)
                  </div>
                )}
              </div>
            </CardContent>
          </Card>
        ))}
      </div>
      
      {/* Job Work 180-Day Tracker */}
      <Card>
        <CardHeader>
          <h2 className="text-lg font-medium">Job Work - 180 Day Tracker</h2>
        </CardHeader>
        <CardContent>
          <table className="w-full">
            <thead>
              <tr className="text-left text-sm text-gray-500">
                <th className="pb-2">Challan</th>
                <th className="pb-2">Job Worker</th>
                <th className="pb-2">Sent Date</th>
                <th className="pb-2">Due Date</th>
                <th className="pb-2">Days Left</th>
                <th className="pb-2">Status</th>
              </tr>
            </thead>
            <tbody>
              {overdueJobWork?.map((challan: any) => (
                <tr key={challan.id} className="border-t">
                  <td className="py-3">{challan.challanNumber}</td>
                  <td>{challan.jobWorkerName}</td>
                  <td>{challan.challanDate}</td>
                  <td>{challan.dueReturnDate}</td>
                  <td>
                    <Badge variant={challan.daysRemaining < 0 ? 'destructive' : 
                                   challan.daysRemaining < 30 ? 'warning' : 'default'}>
                      {challan.daysRemaining < 0 
                        ? `${Math.abs(challan.daysRemaining)} days overdue`
                        : `${challan.daysRemaining} days`}
                    </Badge>
                  </td>
                  <td>
                    <span className="text-sm">{challan.status}</span>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

## **8. Database Schema**

```sql
-- Manufacturing Module Tables

-- Bill of Materials
CREATE TABLE bill_of_materials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    finished_good_id UUID NOT NULL REFERENCES items(id),
    
    bom_code VARCHAR(50) NOT NULL,
    bom_name VARCHAR(200) NOT NULL,
    version INT DEFAULT 1,
    status VARCHAR(20) DEFAULT 'Draft',
    
    standard_batch_size DECIMAL(18,4) DEFAULT 1,
    unit_id UUID NOT NULL REFERENCES units_of_measure(id),
    
    estimated_material_cost DECIMAL(18,2) DEFAULT 0,
    estimated_labour_cost DECIMAL(18,2) DEFAULT 0,
    estimated_overhead_cost DECIMAL(18,2) DEFAULT 0,
    
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_bom_version UNIQUE (tenant_id, finished_good_id, version)
);

CREATE TABLE bom_components (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bom_id UUID NOT NULL REFERENCES bill_of_materials(id) ON DELETE CASCADE,
    item_id UUID NOT NULL REFERENCES items(id),
    
    quantity DECIMAL(18,4) NOT NULL,
    unit_id UUID NOT NULL REFERENCES units_of_measure(id),
    scrap_percentage DECIMAL(5,2) DEFAULT 0,
    
    sub_bom_id UUID REFERENCES bill_of_materials(id),
    alternative_item_id UUID REFERENCES items(id),
    alternative_ratio DECIMAL(10,4),
    
    sequence_number INT DEFAULT 0
);

-- Production Orders
CREATE TABLE production_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    bom_id UUID NOT NULL REFERENCES bill_of_materials(id),
    finished_good_id UUID NOT NULL REFERENCES items(id),
    
    order_number VARCHAR(50) NOT NULL,
    planned_start_date DATE NOT NULL,
    planned_end_date DATE NOT NULL,
    actual_start_date DATE,
    actual_end_date DATE,
    
    planned_quantity DECIMAL(18,4) NOT NULL,
    completed_quantity DECIMAL(18,4) DEFAULT 0,
    scrap_quantity DECIMAL(18,4) DEFAULT 0,
    
    status VARCHAR(20) DEFAULT 'Draft',
    output_godown_id UUID REFERENCES godowns(id),
    sales_order_id UUID REFERENCES sales_orders(id),
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_production_order UNIQUE (tenant_id, order_number)
);

-- Job Work
CREATE TABLE job_work_challans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    type VARCHAR(10) NOT NULL,  -- 'Out' or 'In'
    
    challan_number VARCHAR(50) NOT NULL,
    challan_date DATE NOT NULL,
    due_return_date DATE NOT NULL,
    
    job_worker_id UUID NOT NULL REFERENCES ledgers(id),
    job_worker_gstin VARCHAR(15),
    
    status VARCHAR(30) DEFAULT 'Open',
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_jw_challan UNIQUE (tenant_id, challan_number)
);

CREATE TABLE job_work_challan_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    challan_id UUID NOT NULL REFERENCES job_work_challans(id) ON DELETE CASCADE,
    item_id UUID NOT NULL REFERENCES items(id),
    
    sent_quantity DECIMAL(18,4) NOT NULL,
    unit_id UUID NOT NULL REFERENCES units_of_measure(id),
    
    value_per_unit DECIMAL(18,2) NOT NULL,
    returned_quantity DECIMAL(18,4) DEFAULT 0
);

-- Indexes
CREATE INDEX idx_production_orders_status ON production_orders(tenant_id, status);
CREATE INDEX idx_job_work_due_date ON job_work_challans(tenant_id, due_return_date) 
    WHERE status IN ('Open', 'PartiallyReturned');
```

---

## **9. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| Multi-level BOM | Support up to 5 nesting levels |
| BOM Versioning | Create new version without affecting existing orders |
| Material Requisition | Auto-calculate based on BOM and quantity |
| Production Output | Record output with variance tracking |
| Job Work 180-Day | Alert 30 days before due date |
| Form ITC-04 | Generate compliant JSON for filing |

---

## **10. Definition of Done**

- [ ] BOM CRUD with multi-level support
- [ ] BOM cost rollup calculation
- [ ] Production order lifecycle (Plan → Start → Complete)
- [ ] Material requisition and issue
- [ ] Production output recording with scrap
- [ ] Job Work challan creation
- [ ] Job Work return tracking
- [ ] 180-day compliance alerts
- [ ] Form ITC-04 generation
- [ ] Dashboard with production KPIs
- [ ] Integration tests for BOM explosion

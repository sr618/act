# **Phase 7: Reporting & Analytics Module**
**Duration:** 8 Weeks  
**Team:** Backend (2), Frontend (2), QA (1)  
**Dependencies:** Phase 1-6 (All prior modules)  

---

## **1. Overview**

The Reporting & Analytics Module provides:
- Statutory financial statements (Balance Sheet, P&L, Cash Flow)
- Real-time dashboards with KPIs
- Custom report builder
- Scheduled report generation
- Export to Excel, PDF, and GST JSON

---

## **2. Key Reports**

### **2.1 Financial Statements**
| Report | Standard | Indian Compliance |
|--------|----------|-------------------|
| Balance Sheet | Schedule III format | Companies Act 2013 |
| Profit & Loss | Schedule III format | Companies Act 2013 |
| Cash Flow | Direct/Indirect method | AS-3 / Ind AS 7 |
| Trial Balance | Multi-column format | Standard |
| Fund Flow | Sources and Uses | Standard |

### **2.2 GST Reports**
| Report | Description |
|--------|-------------|
| GSTR-1 Summary | Outward supplies |
| GSTR-2B Reconciliation | Input credit matching |
| GSTR-3B Working | Monthly summary |
| HSN Summary | HSN-wise sales |
| E-Way Bill Register | Transportation tracking |

### **2.3 MIS Reports**
| Report | Description |
|--------|-------------|
| Sales Analysis | By customer, product, region, period |
| Purchase Analysis | By supplier, product, period |
| Inventory Aging | Stock holding analysis |
| Receivables Aging | Debtor analysis |
| Payables Aging | Creditor analysis |

---

## **3. Domain Model**

```csharp
// Domain/Entities/ReportDefinition.cs
public class ReportDefinition : AggregateRoot<ReportDefinitionId>
{
    public TenantId TenantId { get; private set; }
    
    public string ReportCode { get; private set; }
    public string ReportName { get; private set; }
    public string Description { get; private set; }
    
    public ReportCategory Category { get; private set; }
    public ReportType Type { get; private set; }  // Tabular, Chart, Statement
    
    // Query definition
    public string BaseQuery { get; private set; }  // SQL or LINQ
    public List<ReportColumn> Columns { get; private set; }
    public List<ReportParameter> Parameters { get; private set; }
    public List<ReportFilter> Filters { get; private set; }
    
    // Grouping and sorting
    public List<string> GroupByColumns { get; private set; }
    public List<ReportSortOrder> SortOrders { get; private set; }
    
    // Visualization
    public ChartConfiguration? ChartConfig { get; private set; }
    
    public bool IsSystemReport { get; private set; }
    public bool IsActive { get; private set; }
}

public enum ReportCategory
{
    Financial = 1,
    GST = 2,
    Sales = 3,
    Purchase = 4,
    Inventory = 5,
    Banking = 6,
    Manufacturing = 7,
    Custom = 8
}

// Domain/Entities/ScheduledReport.cs
public class ScheduledReport : AggregateRoot<ScheduledReportId>
{
    public TenantId TenantId { get; private set; }
    public ReportDefinitionId ReportDefinitionId { get; private set; }
    
    public string Name { get; private set; }
    public ScheduleType ScheduleType { get; private set; }  // Daily, Weekly, Monthly
    public string CronExpression { get; private set; }
    
    public ExportFormat Format { get; private set; }  // PDF, Excel
    public List<string> Recipients { get; private set; }  // Email addresses
    
    public Dictionary<string, object> ParameterValues { get; private set; }
    
    public DateTime? LastRunAt { get; private set; }
    public DateTime? NextRunAt { get; private set; }
    public bool IsActive { get; private set; }
}
```

---

## **4. Report Engine**

```csharp
// Application/Services/ReportEngine.cs
public class ReportEngine : IReportEngine
{
    private readonly IDbConnection _connection;
    private readonly ITenantContext _tenantContext;
    
    public async Task<ReportResult> ExecuteReportAsync(
        ReportDefinition definition,
        Dictionary<string, object> parameters,
        ReportOptions options)
    {
        var stopwatch = Stopwatch.StartNew();
        
        // Build query
        var query = BuildQuery(definition, parameters, options);
        
        // Execute
        var data = await _connection.QueryAsync<dynamic>(query, parameters);
        
        // Apply transformations
        var transformedData = ApplyTransformations(data, definition);
        
        // Calculate aggregates
        var aggregates = CalculateAggregates(transformedData, definition);
        
        return new ReportResult
        {
            Data = transformedData.ToList(),
            Aggregates = aggregates,
            RowCount = transformedData.Count(),
            ExecutionTime = stopwatch.ElapsedMilliseconds,
            GeneratedAt = DateTime.UtcNow
        };
    }
    
    private string BuildQuery(ReportDefinition definition, Dictionary<string, object> parameters, ReportOptions options)
    {
        var sb = new StringBuilder();
        sb.Append(definition.BaseQuery);
        
        // Apply tenant isolation
        sb.Replace("{TenantId}", _tenantContext.TenantId.ToString());
        
        // Apply filters
        var whereClause = BuildWhereClause(definition.Filters, parameters);
        if (!string.IsNullOrEmpty(whereClause))
        {
            sb.Append($" AND {whereClause}");
        }
        
        // Apply grouping
        if (definition.GroupByColumns.Any())
        {
            sb.Append($" GROUP BY {string.Join(", ", definition.GroupByColumns)}");
        }
        
        // Apply sorting
        if (definition.SortOrders.Any())
        {
            var orderBy = string.Join(", ", definition.SortOrders.Select(s => $"{s.Column} {s.Direction}"));
            sb.Append($" ORDER BY {orderBy}");
        }
        
        // Pagination
        if (options.PageSize > 0)
        {
            sb.Append($" LIMIT {options.PageSize} OFFSET {options.Page * options.PageSize}");
        }
        
        return sb.ToString();
    }
}
```

---

## **5. Financial Statement Generator**

```csharp
// Application/Queries/GetBalanceSheetQuery.cs
public class GetBalanceSheetQuery : IRequest<BalanceSheetDto>
{
    public FinancialYearId FinancialYearId { get; set; }
    public DateOnly AsOnDate { get; set; }
    public BalanceSheetFormat Format { get; set; }  // ScheduleIII, Tally, Custom
    public bool Comparative { get; set; }  // Include previous year
}

public class GetBalanceSheetHandler : IRequestHandler<GetBalanceSheetQuery, BalanceSheetDto>
{
    public async Task<BalanceSheetDto> Handle(GetBalanceSheetQuery request, CancellationToken ct)
    {
        // Get trial balance as on date
        var trialBalance = await _ledgerRepository.GetTrialBalanceAsync(
            request.FinancialYearId, 
            request.AsOnDate);
        
        // Build Schedule III structure
        var balanceSheet = new BalanceSheetDto
        {
            AsOnDate = request.AsOnDate,
            
            // I. EQUITY AND LIABILITIES
            EquityAndLiabilities = new EquityAndLiabilitiesSection
            {
                // 1. Shareholders' Funds
                ShareholdersEquity = new ShareholdersEquitySection
                {
                    ShareCapital = GetGroupBalance(trialBalance, "Share Capital"),
                    ReservesAndSurplus = GetGroupBalance(trialBalance, "Reserves and Surplus"),
                    MoneyReceivedAgainstShares = GetGroupBalance(trialBalance, "Money Received Against Shares")
                },
                
                // 2. Non-Current Liabilities
                NonCurrentLiabilities = new NonCurrentLiabilitiesSection
                {
                    LongTermBorrowings = GetGroupBalance(trialBalance, "Long-Term Borrowings"),
                    DeferredTaxLiabilities = GetGroupBalance(trialBalance, "Deferred Tax Liabilities"),
                    OtherLongTermLiabilities = GetGroupBalance(trialBalance, "Other Long-Term Liabilities"),
                    LongTermProvisions = GetGroupBalance(trialBalance, "Long-Term Provisions")
                },
                
                // 3. Current Liabilities
                CurrentLiabilities = new CurrentLiabilitiesSection
                {
                    ShortTermBorrowings = GetGroupBalance(trialBalance, "Short-Term Borrowings"),
                    TradePayables = GetGroupBalance(trialBalance, "Sundry Creditors"),
                    OtherCurrentLiabilities = GetGroupBalance(trialBalance, "Other Current Liabilities"),
                    ShortTermProvisions = GetGroupBalance(trialBalance, "Short-Term Provisions")
                }
            },
            
            // II. ASSETS
            Assets = new AssetsSection
            {
                // 1. Non-Current Assets
                NonCurrentAssets = new NonCurrentAssetsSection
                {
                    FixedAssets = new FixedAssetsSection
                    {
                        TangibleAssets = GetGroupBalance(trialBalance, "Fixed Assets"),
                        IntangibleAssets = GetGroupBalance(trialBalance, "Intangible Assets"),
                        CapitalWorkInProgress = GetGroupBalance(trialBalance, "Capital Work in Progress")
                    },
                    NonCurrentInvestments = GetGroupBalance(trialBalance, "Non-Current Investments"),
                    LongTermLoansAndAdvances = GetGroupBalance(trialBalance, "Long-Term Loans and Advances"),
                    OtherNonCurrentAssets = GetGroupBalance(trialBalance, "Other Non-Current Assets")
                },
                
                // 2. Current Assets
                CurrentAssets = new CurrentAssetsSection
                {
                    CurrentInvestments = GetGroupBalance(trialBalance, "Current Investments"),
                    Inventories = await GetInventoryValueAsync(request.AsOnDate),
                    TradeReceivables = GetGroupBalance(trialBalance, "Sundry Debtors"),
                    CashAndCashEquivalents = GetGroupBalance(trialBalance, "Cash and Bank"),
                    ShortTermLoansAndAdvances = GetGroupBalance(trialBalance, "Short-Term Loans and Advances"),
                    OtherCurrentAssets = GetGroupBalance(trialBalance, "Other Current Assets")
                }
            }
        };
        
        // Validate Balance Sheet equation
        if (balanceSheet.EquityAndLiabilities.Total != balanceSheet.Assets.Total)
        {
            throw new DomainException($"Balance Sheet does not balance: " +
                $"Liabilities={balanceSheet.EquityAndLiabilities.Total}, Assets={balanceSheet.Assets.Total}");
        }
        
        return balanceSheet;
    }
}
```

---

## **6. Dashboard Engine**

```csharp
// Application/Queries/GetDashboardQuery.cs
public class GetDashboardQuery : IRequest<DashboardDto>
{
    public DateOnly FromDate { get; set; }
    public DateOnly ToDate { get; set; }
}

public class GetDashboardHandler : IRequestHandler<GetDashboardQuery, DashboardDto>
{
    public async Task<DashboardDto> Handle(GetDashboardQuery request, CancellationToken ct)
    {
        // Execute all dashboard queries in parallel
        var salesTask = GetSalesSummaryAsync(request.FromDate, request.ToDate);
        var purchaseTask = GetPurchaseSummaryAsync(request.FromDate, request.ToDate);
        var cashFlowTask = GetCashFlowSummaryAsync(request.FromDate, request.ToDate);
        var receivablesTask = GetReceivablesAgingAsync();
        var payablesTask = GetPayablesAgingAsync();
        var gstTask = GetGstSummaryAsync(request.FromDate, request.ToDate);
        var inventoryTask = GetInventorySummaryAsync();
        
        await Task.WhenAll(salesTask, purchaseTask, cashFlowTask, 
                           receivablesTask, payablesTask, gstTask, inventoryTask);
        
        return new DashboardDto
        {
            Period = new PeriodDto(request.FromDate, request.ToDate),
            
            // KPI Cards
            KPIs = new KPICardsDto
            {
                TotalSales = salesTask.Result.Total,
                SalesGrowth = salesTask.Result.GrowthPercentage,
                TotalPurchases = purchaseTask.Result.Total,
                GrossProfit = salesTask.Result.Total - purchaseTask.Result.Total,
                CashInHand = cashFlowTask.Result.ClosingBalance,
                ReceivablesOutstanding = receivablesTask.Result.Total,
                PayablesOutstanding = payablesTask.Result.Total
            },
            
            // Charts Data
            SalesTrend = await GetSalesTrendAsync(request.FromDate, request.ToDate),
            TopCustomers = await GetTopCustomersAsync(request.FromDate, request.ToDate, 10),
            TopProducts = await GetTopProductsAsync(request.FromDate, request.ToDate, 10),
            
            // Aging Analysis
            ReceivablesAging = receivablesTask.Result,
            PayablesAging = payablesTask.Result,
            
            // GST Summary
            GstSummary = gstTask.Result,
            
            // Inventory
            InventorySummary = inventoryTask.Result,
            LowStockItems = await GetLowStockItemsAsync(10)
        };
    }
}
```

---

## **7. API Endpoints**

```csharp
// API/Controllers/ReportsController.cs
[ApiController]
[Route("api/v1/reports")]
[Authorize]
public class ReportsController : ControllerBase
{
    // Financial Statements
    [HttpGet("balance-sheet")]
    public async Task<ActionResult<BalanceSheetDto>> GetBalanceSheet(
        [FromQuery] DateOnly asOnDate,
        [FromQuery] bool comparative = false)
    {
        return Ok(await _mediator.Send(new GetBalanceSheetQuery 
        { 
            AsOnDate = asOnDate, 
            Comparative = comparative 
        }));
    }
    
    [HttpGet("profit-loss")]
    public async Task<ActionResult<ProfitLossDto>> GetProfitLoss(
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate,
        [FromQuery] bool comparative = false)
    {
        return Ok(await _mediator.Send(new GetProfitLossQuery 
        { 
            FromDate = fromDate, 
            ToDate = toDate, 
            Comparative = comparative 
        }));
    }
    
    [HttpGet("cash-flow")]
    public async Task<ActionResult<CashFlowStatementDto>> GetCashFlow(
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate,
        [FromQuery] CashFlowMethod method = CashFlowMethod.Indirect)
    {
        return Ok(await _mediator.Send(new GetCashFlowStatementQuery 
        { 
            FromDate = fromDate, 
            ToDate = toDate, 
            Method = method 
        }));
    }
    
    [HttpGet("trial-balance")]
    public async Task<ActionResult<TrialBalanceDto>> GetTrialBalance(
        [FromQuery] DateOnly asOnDate,
        [FromQuery] int? groupLevel = null)
    {
        return Ok(await _mediator.Send(new GetTrialBalanceQuery 
        { 
            AsOnDate = asOnDate, 
            GroupLevel = groupLevel 
        }));
    }
    
    // Dashboard
    [HttpGet("dashboard")]
    public async Task<ActionResult<DashboardDto>> GetDashboard(
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate)
    {
        return Ok(await _mediator.Send(new GetDashboardQuery 
        { 
            FromDate = fromDate, 
            ToDate = toDate 
        }));
    }
    
    [HttpGet("dashboard/kpis")]
    public async Task<ActionResult<KPICardsDto>> GetKpis()
    {
        // Real-time KPIs
        return Ok(await _mediator.Send(new GetRealtimeKpisQuery()));
    }
    
    // Custom Reports
    [HttpGet("definitions")]
    public async Task<ActionResult<List<ReportDefinitionDto>>> GetReportDefinitions(
        [FromQuery] ReportCategory? category = null)
    {
        return Ok(await _mediator.Send(new GetReportDefinitionsQuery { Category = category }));
    }
    
    [HttpPost("execute/{reportId}")]
    public async Task<ActionResult<ReportResult>> ExecuteReport(
        Guid reportId,
        [FromBody] ReportExecuteRequest request)
    {
        return Ok(await _mediator.Send(new ExecuteReportCommand 
        { 
            ReportId = reportId, 
            Parameters = request.Parameters,
            Options = request.Options
        }));
    }
    
    [HttpGet("export/{reportId}")]
    public async Task<IActionResult> ExportReport(
        Guid reportId,
        [FromQuery] ExportFormat format,
        [FromQuery] Dictionary<string, string> parameters)
    {
        var result = await _mediator.Send(new ExportReportCommand 
        { 
            ReportId = reportId, 
            Format = format, 
            Parameters = parameters 
        });
        
        return File(result.FileBytes, result.ContentType, result.FileName);
    }
    
    // Scheduled Reports
    [HttpPost("schedule")]
    public async Task<ActionResult<ScheduledReportId>> CreateSchedule(
        [FromBody] CreateScheduledReportCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
}
```

---

## **8. Frontend - Dashboard**

```typescript
// src/app/(dashboard)/reports/dashboard/page.tsx
'use client';

import { useQuery } from '@tanstack/react-query';
import { Card, CardHeader, CardContent } from '@/components/ui/card';
import { 
  LineChart, Line, BarChart, Bar, PieChart, Pie, Cell,
  XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer 
} from 'recharts';
import { ArrowUpIcon, ArrowDownIcon } from 'lucide-react';

const COLORS = ['#0088FE', '#00C49F', '#FFBB28', '#FF8042', '#8884D8'];

export default function DashboardPage() {
  const { data: dashboard, isLoading } = useQuery({
    queryKey: ['dashboard', { period: 'current-month' }],
    queryFn: () => {
      const today = new Date();
      const fromDate = new Date(today.getFullYear(), today.getMonth(), 1);
      return fetch(`/api/v1/reports/dashboard?fromDate=${fromDate.toISOString()}&toDate=${today.toISOString()}`)
        .then(r => r.json());
    },
    refetchInterval: 60000  // Refresh every minute
  });
  
  if (isLoading) return <div>Loading...</div>;
  
  return (
    <div className="p-6 space-y-6">
      <h1 className="text-2xl font-bold">Dashboard</h1>
      
      {/* KPI Cards */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        <KPICard 
          title="Total Sales"
          value={dashboard.kpis.totalSales}
          change={dashboard.kpis.salesGrowth}
          format="currency"
        />
        <KPICard 
          title="Gross Profit"
          value={dashboard.kpis.grossProfit}
          change={null}
          format="currency"
        />
        <KPICard 
          title="Receivables"
          value={dashboard.kpis.receivablesOutstanding}
          change={null}
          format="currency"
          alert={dashboard.kpis.receivablesOutstanding > 1000000}
        />
        <KPICard 
          title="Cash Balance"
          value={dashboard.kpis.cashInHand}
          change={null}
          format="currency"
        />
      </div>
      
      {/* Sales Trend Chart */}
      <Card>
        <CardHeader>
          <h2 className="text-lg font-medium">Sales Trend</h2>
        </CardHeader>
        <CardContent>
          <ResponsiveContainer width="100%" height={300}>
            <LineChart data={dashboard.salesTrend}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="date" />
              <YAxis tickFormatter={(v) => `₹${(v/100000).toFixed(1)}L`} />
              <Tooltip formatter={(v) => `₹${v.toLocaleString('en-IN')}`} />
              <Line type="monotone" dataKey="sales" stroke="#0088FE" strokeWidth={2} />
              <Line type="monotone" dataKey="collection" stroke="#00C49F" strokeWidth={2} />
            </LineChart>
          </ResponsiveContainer>
        </CardContent>
      </Card>
      
      {/* Two Column Layout */}
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        {/* Top Customers */}
        <Card>
          <CardHeader>
            <h2 className="text-lg font-medium">Top 10 Customers</h2>
          </CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={300}>
              <BarChart data={dashboard.topCustomers} layout="vertical">
                <CartesianGrid strokeDasharray="3 3" />
                <XAxis type="number" tickFormatter={(v) => `₹${(v/100000).toFixed(0)}L`} />
                <YAxis type="category" dataKey="name" width={150} />
                <Tooltip formatter={(v) => `₹${v.toLocaleString('en-IN')}`} />
                <Bar dataKey="amount" fill="#0088FE" />
              </BarChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>
        
        {/* Receivables Aging */}
        <Card>
          <CardHeader>
            <h2 className="text-lg font-medium">Receivables Aging</h2>
          </CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={300}>
              <PieChart>
                <Pie
                  data={dashboard.receivablesAging.buckets}
                  dataKey="amount"
                  nameKey="bucket"
                  cx="50%"
                  cy="50%"
                  outerRadius={100}
                  label={({ bucket, percent }) => `${bucket}: ${(percent * 100).toFixed(0)}%`}
                >
                  {dashboard.receivablesAging.buckets.map((entry: any, index: number) => (
                    <Cell key={entry.bucket} fill={COLORS[index % COLORS.length]} />
                  ))}
                </Pie>
                <Tooltip formatter={(v) => `₹${v.toLocaleString('en-IN')}`} />
              </PieChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>
      </div>
      
      {/* GST Summary */}
      <Card>
        <CardHeader>
          <h2 className="text-lg font-medium">GST Summary - Current Month</h2>
        </CardHeader>
        <CardContent>
          <div className="grid grid-cols-3 gap-8 text-center">
            <div>
              <p className="text-gray-500">Output GST (Sales)</p>
              <p className="text-2xl font-bold text-red-600">
                ₹{dashboard.gstSummary.outputGst.toLocaleString('en-IN')}
              </p>
            </div>
            <div>
              <p className="text-gray-500">Input GST (Purchase)</p>
              <p className="text-2xl font-bold text-green-600">
                ₹{dashboard.gstSummary.inputGst.toLocaleString('en-IN')}
              </p>
            </div>
            <div>
              <p className="text-gray-500">Net Payable</p>
              <p className="text-2xl font-bold">
                ₹{dashboard.gstSummary.netPayable.toLocaleString('en-IN')}
              </p>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}

function KPICard({ title, value, change, format, alert = false }: {
  title: string;
  value: number;
  change: number | null;
  format: 'currency' | 'number' | 'percent';
  alert?: boolean;
}) {
  const formattedValue = format === 'currency' 
    ? `₹${value.toLocaleString('en-IN')}`
    : value.toLocaleString('en-IN');
  
  return (
    <Card className={alert ? 'border-red-500' : ''}>
      <CardContent className="pt-6">
        <p className="text-sm text-gray-500">{title}</p>
        <p className="text-2xl font-bold">{formattedValue}</p>
        {change !== null && (
          <div className={`flex items-center text-sm ${change >= 0 ? 'text-green-600' : 'text-red-600'}`}>
            {change >= 0 ? <ArrowUpIcon className="w-4 h-4" /> : <ArrowDownIcon className="w-4 h-4" />}
            <span>{Math.abs(change).toFixed(1)}% vs last month</span>
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

---

## **9. Materialized Views for Performance**

```sql
-- Pre-computed views for dashboard performance

-- Sales summary by day (refreshed every hour)
CREATE MATERIALIZED VIEW mv_sales_daily AS
SELECT 
    s.tenant_id,
    date_trunc('day', s.invoice_date) AS sales_date,
    COUNT(*) AS invoice_count,
    SUM(s.total_amount) AS total_sales,
    SUM(s.total_gst) AS total_gst,
    SUM(s.net_amount) AS net_sales
FROM sales_invoices s
WHERE s.status = 'Posted'
GROUP BY s.tenant_id, date_trunc('day', s.invoice_date)
WITH DATA;

CREATE UNIQUE INDEX ON mv_sales_daily (tenant_id, sales_date);

-- Receivables aging (refreshed every hour)
CREATE MATERIALIZED VIEW mv_receivables_aging AS
SELECT 
    l.tenant_id,
    l.id AS ledger_id,
    l.name AS party_name,
    l.closing_balance AS outstanding,
    CASE 
        WHEN l.closing_balance <= 0 THEN 'Clear'
        WHEN CURRENT_DATE - MAX(v.voucher_date) <= 30 THEN '0-30 Days'
        WHEN CURRENT_DATE - MAX(v.voucher_date) <= 60 THEN '31-60 Days'
        WHEN CURRENT_DATE - MAX(v.voucher_date) <= 90 THEN '61-90 Days'
        ELSE 'Over 90 Days'
    END AS aging_bucket
FROM ledgers l
LEFT JOIN voucher_lines vl ON l.id = vl.ledger_id
LEFT JOIN vouchers v ON vl.voucher_id = v.id
WHERE l.account_group_id IN (SELECT id FROM account_groups WHERE name = 'Sundry Debtors')
  AND l.closing_balance > 0
GROUP BY l.tenant_id, l.id, l.name, l.closing_balance
WITH DATA;

-- Refresh job (Hangfire)
-- _scheduler.AddOrUpdate("refresh-mv-sales", 
--     () => _dbContext.Database.ExecuteSqlRaw("REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_daily"),
--     Cron.Hourly);
```

---

## **10. Database Schema**

```sql
-- Reporting Module Tables

CREATE TABLE report_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),  -- NULL for system reports
    
    report_code VARCHAR(50) NOT NULL,
    report_name VARCHAR(200) NOT NULL,
    description TEXT,
    
    category VARCHAR(30) NOT NULL,
    type VARCHAR(20) NOT NULL,
    
    base_query TEXT NOT NULL,
    columns JSONB NOT NULL,
    parameters JSONB,
    filters JSONB,
    
    group_by_columns JSONB,
    sort_orders JSONB,
    chart_config JSONB,
    
    is_system_report BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE scheduled_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    report_definition_id UUID NOT NULL REFERENCES report_definitions(id),
    
    name VARCHAR(200) NOT NULL,
    schedule_type VARCHAR(20) NOT NULL,
    cron_expression VARCHAR(100) NOT NULL,
    
    format VARCHAR(10) NOT NULL,
    recipients JSONB NOT NULL,
    parameter_values JSONB,
    
    last_run_at TIMESTAMPTZ,
    next_run_at TIMESTAMPTZ,
    
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE report_execution_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    report_definition_id UUID NOT NULL REFERENCES report_definitions(id),
    
    executed_by UUID REFERENCES users(id),
    executed_at TIMESTAMPTZ DEFAULT NOW(),
    
    parameters JSONB,
    execution_time_ms INT,
    row_count INT,
    
    status VARCHAR(20),
    error_message TEXT
);
```

---

## **11. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| Balance Sheet | Generate Schedule III format in < 3s |
| P&L Statement | Support comparative (YoY) view |
| Dashboard | Load all KPIs in < 2s |
| Dashboard Charts | Real-time updates every minute |
| Trial Balance | Support multi-level grouping |
| Report Export | PDF/Excel in < 5s for 10,000 rows |
| Scheduled Reports | Email delivery within 5 min of schedule |

---

## **12. Definition of Done**

- [ ] Balance Sheet (Schedule III format)
- [ ] Profit & Loss Statement (Schedule III format)
- [ ] Cash Flow Statement (Indirect method)
- [ ] Trial Balance with grouping
- [ ] Interactive dashboard with KPIs
- [ ] Sales analysis report
- [ ] Receivables aging report
- [ ] Payables aging report
- [ ] GST summary report
- [ ] Report export (PDF, Excel)
- [ ] Scheduled report delivery
- [ ] Materialized views for performance
- [ ] Dashboard widget customization

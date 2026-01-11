# **Phase 10: Advanced Features & Scale**
**Duration:** 12 Weeks  
**Team:** Backend (3), Frontend (2), DevOps (1), QA (1)  
**Dependencies:** All prior phases  

---

## **1. Overview**

Phase 10 focuses on advanced features for enterprise scale:
- Multi-branch and consolidation
- AI-powered features
- Advanced security
- Performance optimization
- Integration marketplace

---

## **2. Feature Categories**

### **2.1 Multi-Branch & Consolidation**
- Multiple branches under one company
- Branch-level P&L and Balance Sheet
- Inter-branch transactions
- Consolidated financial statements

### **2.2 AI-Powered Features**
- Smart voucher entry suggestions
- Anomaly detection in transactions
- Cash flow prediction
- Invoice data extraction (OCR)

### **2.3 Advanced Security**
- Role-based access control (RBAC)
- Field-level permissions
- Audit logging with tamper detection
- Data encryption at rest

### **2.4 Integration Marketplace**
- Third-party app integrations
- Webhook system
- API rate limiting
- OAuth 2.0 for third-party apps

---

## **3. Multi-Branch Architecture**

```csharp
// Domain/Entities/Branch.cs
public class Branch : AggregateRoot<BranchId>
{
    public TenantId TenantId { get; private set; }
    
    public string BranchCode { get; private set; }
    public string BranchName { get; private set; }
    public string? Gstin { get; private set; }  // If separate registration
    public Address Address { get; private set; }
    
    public BranchId? ParentBranchId { get; private set; }  // For hierarchy
    public bool IsHeadOffice { get; private set; }
    
    // Accounting configuration
    public LedgerId InterBranchLedgerId { get; private set; }
    
    public bool IsActive { get; private set; }
}

// Domain/Entities/InterBranchTransaction.cs
public class InterBranchTransaction : AggregateRoot<InterBranchTransactionId>
{
    public TenantId TenantId { get; private set; }
    
    public BranchId SourceBranchId { get; private set; }
    public BranchId TargetBranchId { get; private set; }
    
    public string TransactionNumber { get; private set; }
    public DateOnly TransactionDate { get; private set; }
    public InterBranchTransactionType Type { get; private set; }
    
    public Money Amount { get; private set; }
    public string Narration { get; private set; }
    
    // Linked vouchers (one in each branch)
    public VoucherId SourceVoucherId { get; private set; }
    public VoucherId TargetVoucherId { get; private set; }
    
    public InterBranchStatus Status { get; private set; }
    public DateTime? ConfirmedAt { get; private set; }
    public UserId? ConfirmedBy { get; private set; }
}

public enum InterBranchTransactionType
{
    FundTransfer = 1,
    StockTransfer = 2,
    ExpenseRecharge = 3,
    SalesTransfer = 4
}

// Application/Commands/CreateInterBranchTransferCommand.cs
public class CreateInterBranchTransferHandler : IRequestHandler<CreateInterBranchTransferCommand, InterBranchTransactionId>
{
    public async Task<InterBranchTransactionId> Handle(CreateInterBranchTransferCommand request, CancellationToken ct)
    {
        // Create voucher in source branch
        var sourceVoucher = new Voucher(
            VoucherType.Journal,
            request.TransactionDate,
            request.Narration);
        
        // Dr. Inter-Branch Account (Source → Target)
        sourceVoucher.AddLine(request.TargetBranch.InterBranchLedgerId, request.Amount, Money.Zero);
        // Cr. Bank/Cash
        sourceVoucher.AddLine(request.SourceLedgerId, Money.Zero, request.Amount);
        
        // Create voucher in target branch
        var targetVoucher = new Voucher(
            VoucherType.Journal,
            request.TransactionDate,
            request.Narration);
        
        // Dr. Bank/Cash
        targetVoucher.AddLine(request.TargetLedgerId, request.Amount, Money.Zero);
        // Cr. Inter-Branch Account (Target → Source)
        targetVoucher.AddLine(request.SourceBranch.InterBranchLedgerId, Money.Zero, request.Amount);
        
        // Create inter-branch transaction record
        var interBranchTxn = new InterBranchTransaction(
            request.SourceBranchId,
            request.TargetBranchId,
            InterBranchTransactionType.FundTransfer,
            request.Amount,
            sourceVoucher.Id,
            targetVoucher.Id);
        
        await _repository.AddAsync(interBranchTxn);
        
        return interBranchTxn.Id;
    }
}
```

---

## **4. Consolidated Financial Statements**

```csharp
// Application/Queries/GetConsolidatedBalanceSheetQuery.cs
public class GetConsolidatedBalanceSheetHandler : IRequestHandler<GetConsolidatedBalanceSheetQuery, ConsolidatedBalanceSheetDto>
{
    public async Task<ConsolidatedBalanceSheetDto> Handle(GetConsolidatedBalanceSheetQuery request, CancellationToken ct)
    {
        var branches = await _branchRepository.GetActiveBranchesAsync(request.TenantId);
        
        var branchBalanceSheets = new List<BalanceSheetDto>();
        
        // Get balance sheet for each branch
        foreach (var branch in branches)
        {
            var bs = await _mediator.Send(new GetBalanceSheetQuery
            {
                BranchId = branch.Id,
                AsOnDate = request.AsOnDate
            }, ct);
            
            branchBalanceSheets.Add(bs);
        }
        
        // Consolidate
        var consolidated = new ConsolidatedBalanceSheetDto
        {
            AsOnDate = request.AsOnDate,
            Branches = branches.Select(b => b.BranchName).ToList()
        };
        
        // Sum all line items
        consolidated.EquityAndLiabilities = ConsolidateSection(
            branchBalanceSheets.Select(b => b.EquityAndLiabilities));
        consolidated.Assets = ConsolidateSection(
            branchBalanceSheets.Select(b => b.Assets));
        
        // Eliminate inter-branch balances
        var interBranchBalance = await GetInterBranchBalanceAsync(request.TenantId, request.AsOnDate);
        
        // Reduce both assets and liabilities by inter-branch amount
        consolidated.EquityAndLiabilities.CurrentLiabilities.OtherCurrentLiabilities -= interBranchBalance;
        consolidated.Assets.CurrentAssets.ShortTermLoansAndAdvances -= interBranchBalance;
        
        return consolidated;
    }
    
    private async Task<decimal> GetInterBranchBalanceAsync(TenantId tenantId, DateOnly asOnDate)
    {
        // Get all inter-branch ledger balances
        var interBranchLedgers = await _ledgerRepository.GetInterBranchLedgersAsync(tenantId);
        
        // Sum of all debit balances (or credit balances - should net to zero ideally)
        return interBranchLedgers
            .Where(l => l.GetClosingBalance(asOnDate).IsDebit)
            .Sum(l => l.GetClosingBalance(asOnDate).Amount);
    }
}
```

---

## **5. AI-Powered Features**

```csharp
// Infrastructure/AI/VoucherSuggestionService.cs
public class VoucherSuggestionService
{
    private readonly IVoucherRepository _voucherRepository;
    private readonly IMLModelService _mlService;
    
    public async Task<VoucherSuggestion> GetSuggestionsAsync(
        string narration, 
        Money amount,
        VoucherType type)
    {
        // Get historical patterns
        var historicalVouchers = await _voucherRepository.GetSimilarVouchersAsync(
            narration, 
            amount, 
            type, 
            limit: 100);
        
        if (!historicalVouchers.Any())
        {
            return VoucherSuggestion.NoSuggestion();
        }
        
        // Use ML model to predict ledger accounts
        var features = new VoucherFeatures
        {
            NarrationEmbedding = await _mlService.GetTextEmbeddingAsync(narration),
            Amount = amount.Amount,
            VoucherType = type,
            DayOfWeek = DateTime.Today.DayOfWeek,
            DayOfMonth = DateTime.Today.Day
        };
        
        var predictions = await _mlService.PredictLedgersAsync(features);
        
        return new VoucherSuggestion
        {
            SuggestedLines = predictions
                .Where(p => p.Confidence > 0.7)
                .Select(p => new SuggestedLine
                {
                    LedgerId = p.LedgerId,
                    LedgerName = p.LedgerName,
                    IsDebit = p.IsDebit,
                    Amount = p.Amount,
                    Confidence = p.Confidence
                })
                .ToList(),
            BasedOnVouchers = historicalVouchers.Take(3).ToList()
        };
    }
}

// Infrastructure/AI/AnomalyDetectionService.cs
public class AnomalyDetectionService
{
    public async Task<List<AnomalyAlert>> DetectAnomaliesAsync(TenantId tenantId, DateOnly date)
    {
        var alerts = new List<AnomalyAlert>();
        
        // 1. Unusual transaction amounts
        var unusualAmounts = await DetectUnusualAmountsAsync(tenantId, date);
        alerts.AddRange(unusualAmounts);
        
        // 2. Unusual timing (weekend, after hours)
        var unusualTiming = await DetectUnusualTimingAsync(tenantId, date);
        alerts.AddRange(unusualTiming);
        
        // 3. New party with large transaction
        var newPartyAlerts = await DetectNewPartyLargeTransactionsAsync(tenantId, date);
        alerts.AddRange(newPartyAlerts);
        
        // 4. Duplicate invoices
        var duplicates = await DetectDuplicateInvoicesAsync(tenantId, date);
        alerts.AddRange(duplicates);
        
        // 5. Round amount transactions (potential manipulation)
        var roundAmounts = await DetectRoundAmountPatternsAsync(tenantId, date);
        alerts.AddRange(roundAmounts);
        
        return alerts;
    }
    
    private async Task<List<AnomalyAlert>> DetectUnusualAmountsAsync(TenantId tenantId, DateOnly date)
    {
        // Get today's vouchers
        var todayVouchers = await _voucherRepository.GetVouchersAsync(tenantId, date, date);
        
        var alerts = new List<AnomalyAlert>();
        
        foreach (var voucher in todayVouchers)
        {
            foreach (var line in voucher.Lines)
            {
                // Get historical stats for this ledger
                var stats = await GetLedgerStatisticsAsync(line.LedgerId);
                
                var amount = line.Debit.Amount > 0 ? line.Debit.Amount : line.Credit.Amount;
                var zScore = (amount - stats.Mean) / stats.StdDev;
                
                if (Math.Abs(zScore) > 3)  // 3 standard deviations
                {
                    alerts.Add(new AnomalyAlert
                    {
                        Type = AnomalyType.UnusualAmount,
                        Severity = Math.Abs(zScore) > 5 ? AlertSeverity.High : AlertSeverity.Medium,
                        VoucherId = voucher.Id,
                        LedgerId = line.LedgerId,
                        Message = $"Transaction amount ₹{amount:N0} is {Math.Abs(zScore):N1} standard deviations from mean",
                        DetectedAt = DateTime.UtcNow
                    });
                }
            }
        }
        
        return alerts;
    }
}

// Infrastructure/AI/CashFlowPredictionService.cs
public class CashFlowPredictionService
{
    private readonly ITimeSeriesModelService _timeSeriesService;
    
    public async Task<CashFlowForecast> PredictCashFlowAsync(
        TenantId tenantId, 
        int daysAhead = 30)
    {
        // Get historical cash flow data
        var historicalData = await GetHistoricalCashFlowAsync(tenantId, days: 365);
        
        // Get scheduled payments (known future outflows)
        var scheduledPayments = await GetScheduledPaymentsAsync(tenantId, daysAhead);
        
        // Get expected collections (based on receivables aging)
        var expectedCollections = await PredictCollectionsAsync(tenantId, daysAhead);
        
        // Use ARIMA or Prophet model for base prediction
        var basePrediction = await _timeSeriesService.ForecastAsync(historicalData, daysAhead);
        
        // Combine with known future transactions
        var forecast = new CashFlowForecast
        {
            StartDate = DateOnly.FromDateTime(DateTime.Today),
            CurrentBalance = await GetCurrentCashBalanceAsync(tenantId),
            DailyPredictions = new List<DailyCashFlowPrediction>()
        };
        
        var runningBalance = forecast.CurrentBalance;
        
        for (int i = 1; i <= daysAhead; i++)
        {
            var date = DateOnly.FromDateTime(DateTime.Today.AddDays(i));
            
            var scheduledOutflow = scheduledPayments
                .Where(p => p.DueDate == date)
                .Sum(p => p.Amount);
            
            var expectedInflow = expectedCollections
                .Where(c => c.ExpectedDate == date)
                .Sum(c => c.Amount * c.Probability);
            
            var predictedNetFlow = basePrediction[i - 1] + expectedInflow - scheduledOutflow;
            runningBalance += predictedNetFlow;
            
            forecast.DailyPredictions.Add(new DailyCashFlowPrediction
            {
                Date = date,
                PredictedInflow = basePrediction[i - 1] > 0 ? basePrediction[i - 1] : 0 + expectedInflow,
                PredictedOutflow = basePrediction[i - 1] < 0 ? Math.Abs(basePrediction[i - 1]) : 0 + scheduledOutflow,
                PredictedBalance = runningBalance,
                Confidence = CalculateConfidence(i)  // Decreases with time
            });
        }
        
        // Detect potential cash crunch
        forecast.CashCrunchWarning = forecast.DailyPredictions
            .Any(p => p.PredictedBalance < 0);
        
        forecast.LowestBalanceDate = forecast.DailyPredictions
            .OrderBy(p => p.PredictedBalance)
            .First().Date;
        
        return forecast;
    }
}
```

---

## **6. Advanced Security - RBAC**

```csharp
// Domain/Entities/Role.cs
public class Role : AggregateRoot<RoleId>
{
    public TenantId TenantId { get; private set; }
    
    public string RoleName { get; private set; }
    public string Description { get; private set; }
    
    public bool IsSystemRole { get; private set; }  // Admin, Auditor, etc.
    
    private readonly List<Permission> _permissions = new();
    public IReadOnlyCollection<Permission> Permissions => _permissions.AsReadOnly();
    
    public void AddPermission(Permission permission)
    {
        if (!_permissions.Contains(permission))
        {
            _permissions.Add(permission);
        }
    }
    
    public bool HasPermission(string resource, string action)
    {
        return _permissions.Any(p => 
            p.Resource == resource && 
            (p.Action == action || p.Action == "*"));
    }
}

// Domain/Entities/Permission.cs
public class Permission : ValueObject
{
    public string Resource { get; private set; }  // "Vouchers", "Ledgers", "Reports"
    public string Action { get; private set; }    // "Create", "Read", "Update", "Delete", "*"
    public string? Scope { get; private set; }    // "Own", "Branch", "All"
    
    // Field-level restrictions
    public List<string>? AllowedFields { get; private set; }
    public List<string>? DeniedFields { get; private set; }
    
    // Condition-based access
    public string? Condition { get; private set; }  // JSON expression
}

// Infrastructure/Authorization/PermissionHandler.cs
public class PermissionHandler : AuthorizationHandler<PermissionRequirement>
{
    private readonly IRoleRepository _roleRepository;
    private readonly ITenantContext _tenantContext;
    
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        PermissionRequirement requirement)
    {
        var userId = context.User.GetUserId();
        var userRoles = await _roleRepository.GetUserRolesAsync(userId, _tenantContext.TenantId);
        
        foreach (var role in userRoles)
        {
            if (role.HasPermission(requirement.Resource, requirement.Action))
            {
                // Check scope
                var permission = role.Permissions.First(p => 
                    p.Resource == requirement.Resource && 
                    p.Action == requirement.Action);
                
                if (CheckScope(permission.Scope, context, requirement))
                {
                    context.Succeed(requirement);
                    return;
                }
            }
        }
        
        context.Fail();
    }
    
    private bool CheckScope(string? scope, AuthorizationHandlerContext context, PermissionRequirement requirement)
    {
        if (scope == null || scope == "All") return true;
        
        if (scope == "Own")
        {
            // Check if resource belongs to current user
            return requirement.ResourceOwnerId == context.User.GetUserId();
        }
        
        if (scope == "Branch")
        {
            // Check if resource belongs to user's branch
            var userBranchId = context.User.GetBranchId();
            return requirement.ResourceBranchId == userBranchId;
        }
        
        return false;
    }
}

// Usage in Controller
[HttpPost]
[Authorize(Policy = "Vouchers.Create")]
public async Task<ActionResult> CreateVoucher([FromBody] CreateVoucherCommand command)
{
    return Ok(await _mediator.Send(command));
}

// Field-level filtering in Query Handler
public class GetVoucherHandler : IRequestHandler<GetVoucherQuery, VoucherDto>
{
    private readonly IFieldPermissionService _fieldPermissions;
    
    public async Task<VoucherDto> Handle(GetVoucherQuery request, CancellationToken ct)
    {
        var voucher = await _repository.GetByIdAsync(request.Id);
        var dto = _mapper.Map<VoucherDto>(voucher);
        
        // Apply field-level filtering
        var allowedFields = await _fieldPermissions.GetAllowedFieldsAsync(
            _currentUser.UserId, 
            "Voucher");
        
        return _fieldPermissions.FilterFields(dto, allowedFields);
    }
}
```

---

## **7. Webhook System**

```csharp
// Domain/Entities/WebhookSubscription.cs
public class WebhookSubscription : AggregateRoot<WebhookSubscriptionId>
{
    public TenantId TenantId { get; private set; }
    
    public string Name { get; private set; }
    public string TargetUrl { get; private set; }
    public string Secret { get; private set; }  // For HMAC signature
    
    public List<string> Events { get; private set; }  // ["voucher.created", "invoice.posted"]
    
    public bool IsActive { get; private set; }
    public int FailureCount { get; private set; }
    public DateTime? LastSuccessAt { get; private set; }
    public DateTime? LastFailureAt { get; private set; }
    
    public void RecordSuccess()
    {
        FailureCount = 0;
        LastSuccessAt = DateTime.UtcNow;
    }
    
    public void RecordFailure()
    {
        FailureCount++;
        LastFailureAt = DateTime.UtcNow;
        
        if (FailureCount >= 10)
        {
            IsActive = false;
            AddDomainEvent(new WebhookDisabledEvent(Id, "Too many failures"));
        }
    }
}

// Infrastructure/Webhooks/WebhookDispatcher.cs
public class WebhookDispatcher : IWebhookDispatcher
{
    private readonly IWebhookSubscriptionRepository _subscriptionRepo;
    private readonly IBackgroundJobClient _jobClient;
    
    public async Task DispatchAsync<TEvent>(TEvent @event) where TEvent : IDomainEvent
    {
        var eventName = GetEventName<TEvent>();
        var tenantId = GetTenantId(@event);
        
        var subscriptions = await _subscriptionRepo.GetActiveSubscriptionsAsync(tenantId, eventName);
        
        foreach (var subscription in subscriptions)
        {
            // Queue webhook delivery
            _jobClient.Enqueue<WebhookDeliveryJob>(job => 
                job.DeliverAsync(subscription.Id, eventName, @event));
        }
    }
}

// Infrastructure/Webhooks/WebhookDeliveryJob.cs
public class WebhookDeliveryJob
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IWebhookSubscriptionRepository _subscriptionRepo;
    
    public async Task DeliverAsync(WebhookSubscriptionId subscriptionId, string eventName, object payload)
    {
        var subscription = await _subscriptionRepo.GetByIdAsync(subscriptionId);
        
        var webhookPayload = new WebhookPayload
        {
            Id = Guid.NewGuid().ToString(),
            Event = eventName,
            Timestamp = DateTime.UtcNow,
            Data = payload
        };
        
        var json = JsonSerializer.Serialize(webhookPayload);
        var signature = ComputeHmacSignature(json, subscription.Secret);
        
        var client = _httpClientFactory.CreateClient("webhook");
        var request = new HttpRequestMessage(HttpMethod.Post, subscription.TargetUrl)
        {
            Content = new StringContent(json, Encoding.UTF8, "application/json")
        };
        
        request.Headers.Add("X-Webhook-Signature", signature);
        request.Headers.Add("X-Webhook-Event", eventName);
        
        try
        {
            var response = await client.SendAsync(request);
            
            if (response.IsSuccessStatusCode)
            {
                subscription.RecordSuccess();
            }
            else
            {
                subscription.RecordFailure();
            }
            
            await _subscriptionRepo.UpdateAsync(subscription);
        }
        catch (Exception ex)
        {
            subscription.RecordFailure();
            await _subscriptionRepo.UpdateAsync(subscription);
            
            // Retry with exponential backoff
            throw new WebhookDeliveryException(ex);
        }
    }
    
    private string ComputeHmacSignature(string payload, string secret)
    {
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(payload));
        return $"sha256={Convert.ToBase64String(hash)}";
    }
}
```

---

## **8. API Rate Limiting**

```csharp
// Infrastructure/RateLimiting/RateLimitingMiddleware.cs
public class RateLimitingMiddleware
{
    private readonly IDistributedCache _cache;
    
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var clientId = GetClientIdentifier(context);
        var endpoint = context.Request.Path.Value;
        
        var rateLimitConfig = GetRateLimitConfig(endpoint);
        
        var key = $"ratelimit:{clientId}:{endpoint}:{GetCurrentWindow(rateLimitConfig.Window)}";
        var currentCount = await _cache.GetAsync<int>(key);
        
        if (currentCount >= rateLimitConfig.Limit)
        {
            context.Response.StatusCode = 429;
            context.Response.Headers.Add("Retry-After", rateLimitConfig.Window.TotalSeconds.ToString());
            await context.Response.WriteAsJsonAsync(new 
            {
                error = "Rate limit exceeded",
                retryAfter = rateLimitConfig.Window.TotalSeconds
            });
            return;
        }
        
        // Increment counter
        await _cache.SetAsync(key, currentCount + 1, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = rateLimitConfig.Window
        });
        
        // Add rate limit headers
        context.Response.Headers.Add("X-RateLimit-Limit", rateLimitConfig.Limit.ToString());
        context.Response.Headers.Add("X-RateLimit-Remaining", (rateLimitConfig.Limit - currentCount - 1).ToString());
        
        await next(context);
    }
}

// Configuration
services.AddRateLimiting(options =>
{
    options.AddPolicy("standard", new RateLimitPolicy
    {
        Limit = 1000,
        Window = TimeSpan.FromHours(1)
    });
    
    options.AddPolicy("reports", new RateLimitPolicy
    {
        Limit = 100,
        Window = TimeSpan.FromHours(1)
    });
    
    options.AddPolicy("sync", new RateLimitPolicy
    {
        Limit = 60,
        Window = TimeSpan.FromMinutes(1)
    });
});
```

---

## **9. Performance Optimization**

```csharp
// Infrastructure/Caching/DistributedCacheService.cs
public class DistributedCacheService : ICacheService
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<DistributedCacheService> _logger;
    
    public async Task<T?> GetOrSetAsync<T>(
        string key, 
        Func<Task<T>> factory, 
        TimeSpan? expiration = null)
    {
        var cached = await _cache.GetStringAsync(key);
        
        if (cached != null)
        {
            _logger.LogDebug("Cache hit for {Key}", key);
            return JsonSerializer.Deserialize<T>(cached);
        }
        
        _logger.LogDebug("Cache miss for {Key}", key);
        var value = await factory();
        
        if (value != null)
        {
            await _cache.SetStringAsync(key, JsonSerializer.Serialize(value), new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = expiration ?? TimeSpan.FromMinutes(10)
            });
        }
        
        return value;
    }
    
    public async Task InvalidatePatternAsync(string pattern)
    {
        // For Redis, use SCAN to find matching keys
        // For other caches, maintain a key registry
        _logger.LogInformation("Invalidating cache pattern: {Pattern}", pattern);
    }
}

// Cache invalidation on domain events
public class VoucherCreatedEventHandler : INotificationHandler<VoucherCreatedEvent>
{
    private readonly ICacheService _cache;
    
    public async Task Handle(VoucherCreatedEvent notification, CancellationToken ct)
    {
        // Invalidate affected caches
        await _cache.InvalidatePatternAsync($"trial-balance:{notification.TenantId}:*");
        await _cache.InvalidatePatternAsync($"ledger-statement:{notification.TenantId}:{notification.AffectedLedgerIds}:*");
        await _cache.InvalidatePatternAsync($"dashboard:{notification.TenantId}:*");
    }
}
```

---

## **10. Database Sharding Strategy**

```sql
-- For very large tenants, consider horizontal sharding

-- Shard key: tenant_id
-- Sharding strategy: Hash-based distribution

-- Example: Citus extension for PostgreSQL
CREATE EXTENSION IF NOT EXISTS citus;

-- Distribute tables across shards
SELECT create_distributed_table('vouchers', 'tenant_id');
SELECT create_distributed_table('voucher_lines', 'tenant_id');
SELECT create_distributed_table('ledgers', 'tenant_id');

-- Co-locate related tables for efficient JOINs
SELECT create_distributed_table('voucher_lines', 'tenant_id', 
    colocate_with => 'vouchers');

-- Reference tables (replicated to all shards)
SELECT create_reference_table('account_groups');
SELECT create_reference_table('gst_rates');
```

---

## **11. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| Multi-Branch | Support 50+ branches per tenant |
| Consolidation | Generate consolidated statements in < 30s |
| AI Suggestions | 80%+ accuracy on trained data |
| Anomaly Detection | < 5% false positive rate |
| RBAC | Support 100+ custom roles |
| Webhooks | 99% delivery success rate |
| Rate Limiting | No performance degradation |
| Response Time | P95 < 200ms for standard APIs |

---

## **12. Definition of Done**

- [ ] Multi-branch configuration
- [ ] Inter-branch transactions
- [ ] Consolidated Balance Sheet
- [ ] Consolidated P&L
- [ ] AI voucher suggestions (ML model trained)
- [ ] Anomaly detection alerts
- [ ] Cash flow prediction dashboard
- [ ] RBAC with field-level permissions
- [ ] Audit log with tamper detection
- [ ] Webhook system with retry
- [ ] API rate limiting
- [ ] Redis caching layer
- [ ] Database query optimization
- [ ] Load testing (1000 concurrent users)
- [ ] Security audit completed

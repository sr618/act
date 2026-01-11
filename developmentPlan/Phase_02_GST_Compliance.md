# **Phase 2: GST & Compliance Module**
**Duration:** 8 Weeks  
**Team:** Backend (2), Frontend (2), QA (1)  
**Dependencies:** Phase 1 (Core Accounting) Complete  

---

## **1. Overview**

The GST & Compliance Module is the **compliance shield** of AuditFlow ERP. It handles:
- E-Invoice generation via IRP
- E-Way Bill creation
- GSTR-1, GSTR-3B return preparation
- GSTR-2B reconciliation (ITC matching)
- TDS compliance
- Non-disableable Audit Trail enforcement

This module is **plug-and-play** - tenants can enable/disable it based on subscription.

---

## **2. Module Architecture**

```
┌─────────────────────────────────────────────────────────────────┐
│                     GST COMPLIANCE MODULE                        │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  E-Invoice  │  │   E-Way     │  │   GSTR      │             │
│  │   Service   │  │   Bill      │  │  Returns    │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│  ┌──────┴────────────────┴────────────────┴──────┐             │
│  │              GST RULES ENGINE                  │             │
│  │  (Configurable tax rates, validations, rules) │             │
│  └───────────────────────┬───────────────────────┘             │
│                          │                                      │
│  ┌───────────────────────┴───────────────────────┐             │
│  │           GSP INTEGRATION LAYER               │             │
│  │     (ClearTax / Sandbox / Custom API)         │             │
│  └───────────────────────────────────────────────┘             │
├─────────────────────────────────────────────────────────────────┤
│  EVENT BUS: Subscribes to VoucherPostedEvent from Core          │
│             Publishes: EInvoiceGeneratedEvent, GSTFiledEvent    │
└─────────────────────────────────────────────────────────────────┘
```

---

## **3. Domain Model**

### **3.1 GST Configuration**

```csharp
// Domain/Entities/GstConfiguration.cs
public class GstConfiguration : AggregateRoot<GstConfigurationId>
{
    public TenantId TenantId { get; private set; }
    
    // Primary GSTIN
    public Gstin PrimaryGstin { get; private set; }
    public string LegalName { get; private set; }
    public string TradeName { get; private set; }
    public string StateCode { get; private set; }
    public GstRegistrationType RegistrationType { get; private set; }
    
    // Thresholds (auto-updated based on turnover)
    public decimal AnnualTurnover { get; private set; }
    public bool IsEInvoiceMandatory => AnnualTurnover >= 50000000; // 5 Cr
    public bool Is30DayRuleApplicable => AnnualTurnover >= 100000000; // 10 Cr
    public int HsnDigitsRequired => AnnualTurnover >= 50000000 ? 6 : 4;
    
    // GSP Credentials (Encrypted)
    public GspCredentials? GspCredentials { get; private set; }
    
    // Multiple GSTINs (for multi-state businesses)
    private readonly List<AdditionalGstin> _additionalGstins = new();
    public IReadOnlyCollection<AdditionalGstin> AdditionalGstins => _additionalGstins.AsReadOnly();
    
    // E-Invoice Settings
    public EInvoiceSettings EInvoiceSettings { get; private set; }
}

public record Gstin
{
    public string Value { get; }
    
    public Gstin(string value)
    {
        if (!IsValid(value))
            throw new DomainException($"Invalid GSTIN: {value}");
        Value = value.ToUpperInvariant();
    }
    
    public string StateCode => Value[..2];
    public string Pan => Value[2..12];
    public string EntityCode => Value[12..13];
    public string CheckDigit => Value[14..15];
    
    public static bool IsValid(string gstin)
    {
        if (string.IsNullOrEmpty(gstin) || gstin.Length != 15)
            return false;
        
        // Format: 22AAAAA0000A1Z5
        var pattern = @"^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}Z[0-9A-Z]{1}$";
        if (!Regex.IsMatch(gstin, pattern))
            return false;
        
        // Validate checksum
        return ValidateChecksum(gstin);
    }
    
    private static bool ValidateChecksum(string gstin)
    {
        // GSTIN checksum validation logic
        var chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
        int sum = 0;
        for (int i = 0; i < 14; i++)
        {
            int idx = chars.IndexOf(gstin[i]);
            int factor = (i % 2 == 0) ? 1 : 2;
            int product = idx * factor;
            sum += (product / 36) + (product % 36);
        }
        int checkDigit = (36 - (sum % 36)) % 36;
        return chars[checkDigit] == gstin[14];
    }
}

public enum GstRegistrationType
{
    Regular = 1,
    Composition = 2,
    InputServiceDistributor = 3,
    CasualTaxablePerson = 4,
    NonResidentTaxablePerson = 5,
    UINHolder = 6,
    SEZUnit = 7,
    SEZDeveloper = 8
}
```

### **3.2 E-Invoice Entity**

```csharp
// Domain/Entities/EInvoice.cs
public class EInvoice : AggregateRoot<EInvoiceId>
{
    public TenantId TenantId { get; private set; }
    public Guid SourceVoucherId { get; private set; }
    public string SourceDocumentNumber { get; private set; }
    
    // IRP Response
    public string? Irn { get; private set; }  // Invoice Reference Number
    public string? AckNumber { get; private set; }
    public DateTime? AckDate { get; private set; }
    public string? SignedQrCode { get; private set; }
    public string? SignedInvoice { get; private set; }
    
    // Invoice Details
    public EInvoiceType InvoiceType { get; private set; }
    public DateOnly InvoiceDate { get; private set; }
    public string InvoiceNumber { get; private set; }
    
    // Supplier & Buyer
    public GstParty Supplier { get; private set; }
    public GstParty Buyer { get; private set; }
    public GstParty? ShipTo { get; private set; }
    public GstParty? DispatchFrom { get; private set; }
    
    // Values
    public decimal TotalValue { get; private set; }
    public decimal TaxableValue { get; private set; }
    public decimal CgstAmount { get; private set; }
    public decimal SgstAmount { get; private set; }
    public decimal IgstAmount { get; private set; }
    public decimal CessAmount { get; private set; }
    public decimal TotalInvoiceValue { get; private set; }
    
    // Line Items
    private readonly List<EInvoiceItem> _items = new();
    public IReadOnlyCollection<EInvoiceItem> Items => _items.AsReadOnly();
    
    // Status
    public EInvoiceStatus Status { get; private set; }
    public string? ErrorMessage { get; private set; }
    public int RetryCount { get; private set; }
    
    // Cancellation
    public bool IsCancelled { get; private set; }
    public string? CancelReason { get; private set; }
    public DateTime? CancelledAt { get; private set; }
    
    public void MarkAsGenerated(IrpResponse response)
    {
        Irn = response.Irn;
        AckNumber = response.AckNo;
        AckDate = response.AckDt;
        SignedQrCode = response.SignedQRCode;
        SignedInvoice = response.SignedInvoice;
        Status = EInvoiceStatus.Generated;
        
        AddDomainEvent(new EInvoiceGeneratedEvent(this));
    }
    
    public Result Validate()
    {
        var errors = new List<string>();
        
        // Check 30-day rule
        var daysSinceInvoice = (DateTime.Today - InvoiceDate.ToDateTime(TimeOnly.MinValue)).Days;
        if (daysSinceInvoice > 30)
        {
            errors.Add("E-Invoice cannot be generated for invoices older than 30 days (As per Notification No. 12/2024)");
        }
        
        // Validate GSTIN
        if (!Buyer.Gstin.HasValue && InvoiceType == EInvoiceType.B2B)
        {
            errors.Add("Buyer GSTIN is mandatory for B2B invoices");
        }
        
        // Validate HSN codes
        foreach (var item in _items)
        {
            if (item.HsnCode.Length < 4)
            {
                errors.Add($"Item '{item.ProductName}' has invalid HSN code (minimum 4 digits required)");
            }
        }
        
        return errors.Any() ? Result.Failure(errors) : Result.Success();
    }
}

public enum EInvoiceStatus
{
    Pending = 1,
    Queued = 2,
    Generated = 3,
    Failed = 4,
    Cancelled = 5
}

// Domain/Entities/EInvoiceItem.cs
public class EInvoiceItem : Entity<EInvoiceItemId>
{
    public EInvoiceId EInvoiceId { get; private set; }
    public int SlNo { get; private set; }
    public string ProductName { get; private set; }
    public string Description { get; private set; }
    public string HsnCode { get; private set; }
    public string Uom { get; private set; }  // Unit of Measure
    public decimal Quantity { get; private set; }
    public decimal UnitPrice { get; private set; }
    public decimal Discount { get; private set; }
    public decimal TaxableValue { get; private set; }
    public decimal GstRate { get; private set; }
    public decimal CgstAmount { get; private set; }
    public decimal SgstAmount { get; private set; }
    public decimal IgstAmount { get; private set; }
    public decimal CessRate { get; private set; }
    public decimal CessAmount { get; private set; }
    public decimal TotalItemValue { get; private set; }
}
```

### **3.3 GSTR Reconciliation**

```csharp
// Domain/Entities/Gstr2bReconciliation.cs
public class Gstr2bReconciliation : AggregateRoot<Gstr2bReconciliationId>
{
    public TenantId TenantId { get; private set; }
    public string ReturnPeriod { get; private set; }  // "012026" for Jan 2026
    public DateTime FetchedAt { get; private set; }
    
    // Summary
    public int TotalInvoicesFromPortal { get; private set; }
    public int MatchedCount { get; private set; }
    public int MismatchedCount { get; private set; }
    public int MissingInBooksCount { get; private set; }
    public int MissingInPortalCount { get; private set; }
    
    public decimal TotalItcAvailable { get; private set; }
    public decimal TotalItcMatched { get; private set; }
    public decimal ItcAtRisk { get; private set; }
    
    // Detail Lines
    private readonly List<Gstr2bLine> _lines = new();
    public IReadOnlyCollection<Gstr2bLine> Lines => _lines.AsReadOnly();
    
    public void RunReconciliation(List<PurchaseVoucher> purchases)
    {
        foreach (var line in _lines)
        {
            var match = FindMatch(line, purchases);
            if (match == null)
            {
                line.SetStatus(ReconciliationStatus.MissingInBooks);
            }
            else if (IsExactMatch(line, match))
            {
                line.SetStatus(ReconciliationStatus.Matched, match.VoucherId);
            }
            else
            {
                line.SetStatus(ReconciliationStatus.Mismatch, match.VoucherId);
                line.SetMismatchDetails(GetMismatchDetails(line, match));
            }
        }
        
        // Find invoices in books but not in portal
        foreach (var purchase in purchases)
        {
            if (!_lines.Any(l => l.MatchedVoucherId == purchase.VoucherId))
            {
                var missingLine = Gstr2bLine.CreateMissingInPortal(purchase);
                _lines.Add(missingLine);
            }
        }
        
        UpdateSummary();
    }
}

public class Gstr2bLine : Entity<Gstr2bLineId>
{
    // From GST Portal
    public string VendorGstin { get; private set; }
    public string VendorName { get; private set; }
    public string InvoiceNumber { get; private set; }
    public DateOnly InvoiceDate { get; private set; }
    public decimal TaxableValue { get; private set; }
    public decimal IgstAmount { get; private set; }
    public decimal CgstAmount { get; private set; }
    public decimal SgstAmount { get; private set; }
    public decimal CessAmount { get; private set; }
    public decimal TotalTax => IgstAmount + CgstAmount + SgstAmount + CessAmount;
    
    // Reconciliation Result
    public ReconciliationStatus Status { get; private set; }
    public Guid? MatchedVoucherId { get; private set; }
    public string? MismatchDetails { get; private set; }
    
    // Vendor Filing Status
    public FilingStatus VendorFilingStatus { get; private set; }
    public bool IsDefaultingVendor => VendorFilingStatus == FilingStatus.NotFiled;
    
    // Action taken
    public ReconciliationAction? ActionTaken { get; private set; }
    public string? ActionNote { get; private set; }
}

public enum ReconciliationStatus
{
    Pending = 1,
    Matched = 2,
    Mismatch = 3,
    MissingInBooks = 4,
    MissingInPortal = 5
}

public enum ReconciliationAction
{
    None = 0,
    Accepted = 1,
    Disputed = 2,
    PaymentBlocked = 3,
    CommunicationSent = 4
}
```

### **3.4 TDS Management**

```csharp
// Domain/Entities/TdsTransaction.cs
public class TdsTransaction : AggregateRoot<TdsTransactionId>
{
    public TenantId TenantId { get; private set; }
    public Guid VoucherId { get; private set; }
    public LedgerId PartyLedgerId { get; private set; }
    
    // TDS Section
    public TdsSection Section { get; private set; }
    public string SectionCode { get; private set; }  // "194C", "194J", etc.
    public string NatureOfPayment { get; private set; }
    
    // Party Details
    public string PartyName { get; private set; }
    public string Pan { get; private set; }
    public PartyType PartyType { get; private set; }  // Individual, Company, etc.
    
    // Transaction Details
    public DateOnly TransactionDate { get; private set; }
    public decimal GrossAmount { get; private set; }
    public decimal TdsRate { get; private set; }
    public decimal TdsAmount { get; private set; }
    public decimal NetPayable { get; private set; }
    
    // Deposit Details
    public bool IsDeposited { get; private set; }
    public string? ChallanNumber { get; private set; }
    public DateOnly? DepositDate { get; private set; }
    public string? BsrCode { get; private set; }
    
    // Certificate
    public bool IsCertificateIssued { get; private set; }
    public string? CertificateNumber { get; private set; }  // Form 16A number
    
    public static decimal CalculateTdsRate(TdsSection section, PartyType partyType, bool hasPan)
    {
        if (!hasPan)
            return 20.0m;  // Higher rate for non-PAN
        
        return section switch
        {
            TdsSection.Section194C when partyType == PartyType.Individual => 1.0m,
            TdsSection.Section194C => 2.0m,
            TdsSection.Section194H => 5.0m,
            TdsSection.Section194I => 10.0m,
            TdsSection.Section194J when partyType == PartyType.Individual => 10.0m,
            TdsSection.Section194J => 10.0m,  // For technical services: 2%
            TdsSection.Section194Q => 0.1m,
            _ => 10.0m
        };
    }
}

public enum TdsSection
{
    Section194C,  // Contractors
    Section194H,  // Commission
    Section194I,  // Rent
    Section194J,  // Professional/Technical
    Section194Q,  // Purchase of Goods
    Section195,   // NRI Payments
}
```

---

## **4. GSP Integration Layer**

### **4.1 GSP Interface (Abstraction)**

```csharp
// Infrastructure/Gsp/IGspClient.cs
public interface IGspClient
{
    // E-Invoice
    Task<Result<IrpResponse>> GenerateEInvoiceAsync(EInvoiceRequest request, CancellationToken ct);
    Task<Result<IrpResponse>> CancelEInvoiceAsync(string irn, string reason, CancellationToken ct);
    Task<Result<EInvoiceDetails>> GetEInvoiceAsync(string irn, CancellationToken ct);
    
    // E-Way Bill
    Task<Result<EwbResponse>> GenerateEwayBillAsync(EwayBillRequest request, CancellationToken ct);
    Task<Result<EwbResponse>> UpdateTransporterAsync(string ewbNo, TransporterDetails transporter, CancellationToken ct);
    Task<Result<EwbResponse>> CancelEwayBillAsync(string ewbNo, string reason, CancellationToken ct);
    
    // GSTR
    Task<Result<Gstr2bData>> FetchGstr2bAsync(string gstin, string returnPeriod, CancellationToken ct);
    Task<Result<GstrSaveResponse>> SaveGstr1Async(string gstin, string returnPeriod, Gstr1Data data, CancellationToken ct);
    Task<Result<GstrFileResponse>> FileGstr1Async(string gstin, string returnPeriod, string otp, CancellationToken ct);
}

// Infrastructure/Gsp/ClearTax/ClearTaxGspClient.cs
public class ClearTaxGspClient : IGspClient
{
    private readonly HttpClient _httpClient;
    private readonly ClearTaxSettings _settings;
    private readonly ILogger<ClearTaxGspClient> _logger;
    
    public ClearTaxGspClient(HttpClient httpClient, IOptions<ClearTaxSettings> settings, ILogger<ClearTaxGspClient> logger)
    {
        _httpClient = httpClient;
        _settings = settings.Value;
        _logger = logger;
        
        _httpClient.BaseAddress = new Uri(_settings.BaseUrl);
        _httpClient.DefaultRequestHeaders.Add("x-cleartax-auth-token", _settings.AuthToken);
    }
    
    public async Task<Result<IrpResponse>> GenerateEInvoiceAsync(EInvoiceRequest request, CancellationToken ct)
    {
        try
        {
            var payload = MapToIrpPayload(request);
            var content = new StringContent(JsonSerializer.Serialize(payload), Encoding.UTF8, "application/json");
            
            var response = await _httpClient.PostAsync("/integration/v2/einvoice/generate", content, ct);
            
            if (!response.IsSuccessStatusCode)
            {
                var error = await response.Content.ReadAsStringAsync(ct);
                _logger.LogError("E-Invoice generation failed: {Error}", error);
                return Result.Failure<IrpResponse>($"GSP Error: {error}");
            }
            
            var result = await response.Content.ReadFromJsonAsync<ClearTaxEInvoiceResponse>(ct);
            return Result.Success(MapToIrpResponse(result));
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Network error during E-Invoice generation");
            return Result.Failure<IrpResponse>("Network error. Request queued for retry.");
        }
    }
    
    private IrpPayload MapToIrpPayload(EInvoiceRequest request)
    {
        return new IrpPayload
        {
            Version = "1.1",
            TranDtls = new TransactionDetails
            {
                TaxSch = "GST",
                SupTyp = request.SupplyType,
                RegRev = request.IsReverseCharge ? "Y" : "N"
            },
            DocDtls = new DocumentDetails
            {
                Typ = request.InvoiceType.ToIrpCode(),
                No = request.InvoiceNumber,
                Dt = request.InvoiceDate.ToString("dd/MM/yyyy")
            },
            SellerDtls = MapParty(request.Supplier),
            BuyerDtls = MapParty(request.Buyer),
            ItemList = request.Items.Select(MapItem).ToList(),
            ValDtls = new ValueDetails
            {
                AssVal = request.TaxableValue,
                CgstVal = request.CgstAmount,
                SgstVal = request.SgstAmount,
                IgstVal = request.IgstAmount,
                CesVal = request.CessAmount,
                TotInvVal = request.TotalInvoiceValue
            }
        };
    }
}
```

### **4.2 Async Queue for API Resilience**

```csharp
// Application/BackgroundJobs/EInvoiceGenerationJob.cs
public class EInvoiceGenerationJob
{
    private readonly IGspClient _gspClient;
    private readonly IEInvoiceRepository _repository;
    private readonly IEventBus _eventBus;
    private readonly ILogger<EInvoiceGenerationJob> _logger;
    
    [AutomaticRetry(Attempts = 5, DelaysInSeconds = new[] { 60, 300, 900, 3600, 7200 })]
    public async Task GenerateAsync(Guid eInvoiceId)
    {
        var eInvoice = await _repository.GetByIdAsync(EInvoiceId.From(eInvoiceId));
        
        if (eInvoice == null || eInvoice.Status == EInvoiceStatus.Generated)
            return;
        
        // Validate before calling API
        var validation = eInvoice.Validate();
        if (validation.IsFailure)
        {
            eInvoice.MarkAsFailed(string.Join(", ", validation.Errors));
            await _repository.UpdateAsync(eInvoice);
            return;
        }
        
        var request = MapToRequest(eInvoice);
        var result = await _gspClient.GenerateEInvoiceAsync(request, CancellationToken.None);
        
        if (result.IsSuccess)
        {
            eInvoice.MarkAsGenerated(result.Value);
            await _eventBus.PublishAsync(new EInvoiceGeneratedEvent(eInvoice));
        }
        else
        {
            eInvoice.IncrementRetryCount();
            eInvoice.SetErrorMessage(result.Error);
            
            if (eInvoice.RetryCount >= 5)
            {
                eInvoice.MarkAsFailed(result.Error);
            }
        }
        
        await _repository.UpdateAsync(eInvoice);
    }
}
```

---

## **5. Application Layer**

### **5.1 Commands**

```csharp
// Application/Commands/GenerateEInvoice/GenerateEInvoiceCommand.cs
public record GenerateEInvoiceCommand : IRequest<Result<EInvoiceId>>
{
    public Guid VoucherId { get; init; }
}

public class GenerateEInvoiceCommandHandler : IRequestHandler<GenerateEInvoiceCommand, Result<EInvoiceId>>
{
    private readonly IVoucherRepository _voucherRepo;
    private readonly IEInvoiceRepository _eInvoiceRepo;
    private readonly IGstConfigRepository _gstConfigRepo;
    private readonly IBackgroundJobClient _jobClient;
    
    public async Task<Result<EInvoiceId>> Handle(GenerateEInvoiceCommand request, CancellationToken ct)
    {
        // 1. Get voucher
        var voucher = await _voucherRepo.GetByIdAsync(VoucherId.From(request.VoucherId), ct);
        if (voucher == null)
            return Result.Failure<EInvoiceId>("Voucher not found");
        
        if (!voucher.IsPosted)
            return Result.Failure<EInvoiceId>("Voucher must be posted before generating E-Invoice");
        
        // 2. Check if E-Invoice already exists
        var existing = await _eInvoiceRepo.GetByVoucherIdAsync(voucher.Id, ct);
        if (existing != null && existing.Status == EInvoiceStatus.Generated)
            return Result.Failure<EInvoiceId>("E-Invoice already generated for this voucher");
        
        // 3. Get GST config
        var gstConfig = await _gstConfigRepo.GetAsync(ct);
        if (gstConfig == null)
            return Result.Failure<EInvoiceId>("GST configuration not set up");
        
        // 4. Validate 30-day rule
        var daysSinceVoucher = (DateOnly.FromDateTime(DateTime.Today) - voucher.VoucherDate).Days;
        if (daysSinceVoucher > 30 && gstConfig.Is30DayRuleApplicable)
        {
            return Result.Failure<EInvoiceId>(
                "Cannot generate E-Invoice: Invoice is older than 30 days. " +
                "As per Notification No. 12/2024-Central Tax, for taxpayers with turnover > ₹10 Cr, " +
                "e-invoice cannot be reported after 30 days from invoice date."
            );
        }
        
        // 5. Create E-Invoice entity
        var eInvoice = EInvoice.Create(
            tenantId: gstConfig.TenantId,
            sourceVoucherId: voucher.Id.Value,
            sourceDocumentNumber: voucher.VoucherNumber,
            invoiceType: MapVoucherTypeToEInvoiceType(voucher.Type),
            invoiceDate: voucher.VoucherDate,
            invoiceNumber: voucher.VoucherNumber,
            supplier: gstConfig.ToGstParty(),
            buyer: voucher.GetBuyerDetails()
        );
        
        // Add items from voucher
        foreach (var item in voucher.GetInvoiceItems())
        {
            eInvoice.AddItem(item);
        }
        
        // 6. Validate
        var validation = eInvoice.Validate();
        if (validation.IsFailure)
            return Result.Failure<EInvoiceId>(validation.Errors);
        
        // 7. Save and queue for generation
        await _eInvoiceRepo.AddAsync(eInvoice, ct);
        
        // Queue background job
        _jobClient.Enqueue<EInvoiceGenerationJob>(j => j.GenerateAsync(eInvoice.Id.Value));
        
        return Result.Success(eInvoice.Id);
    }
}

// Application/Commands/ReconcileGstr2b/ReconcileGstr2bCommand.cs
public record ReconcileGstr2bCommand : IRequest<Result<Gstr2bReconciliationId>>
{
    public string ReturnPeriod { get; init; } = string.Empty;  // "012026"
}

public class ReconcileGstr2bCommandHandler : IRequestHandler<ReconcileGstr2bCommand, Result<Gstr2bReconciliationId>>
{
    private readonly IGspClient _gspClient;
    private readonly IVoucherRepository _voucherRepo;
    private readonly IGstConfigRepository _gstConfigRepo;
    private readonly IGstr2bRepository _gstr2bRepo;
    
    public async Task<Result<Gstr2bReconciliationId>> Handle(ReconcileGstr2bCommand request, CancellationToken ct)
    {
        var gstConfig = await _gstConfigRepo.GetAsync(ct);
        
        // 1. Fetch GSTR-2B from portal
        var gstr2bResult = await _gspClient.FetchGstr2bAsync(
            gstConfig.PrimaryGstin.Value, 
            request.ReturnPeriod, 
            ct
        );
        
        if (gstr2bResult.IsFailure)
            return Result.Failure<Gstr2bReconciliationId>($"Failed to fetch GSTR-2B: {gstr2bResult.Error}");
        
        // 2. Get purchase vouchers for the period
        var (startDate, endDate) = ParseReturnPeriod(request.ReturnPeriod);
        var purchases = await _voucherRepo.GetPurchaseVouchersAsync(startDate, endDate, ct);
        
        // 3. Create reconciliation
        var reconciliation = Gstr2bReconciliation.Create(
            gstConfig.TenantId,
            request.ReturnPeriod,
            gstr2bResult.Value.Invoices
        );
        
        // 4. Run matching algorithm
        reconciliation.RunReconciliation(purchases);
        
        // 5. Save
        await _gstr2bRepo.AddAsync(reconciliation, ct);
        
        return Result.Success(reconciliation.Id);
    }
}
```

---

## **6. API Endpoints**

```csharp
// API/Controllers/GstController.cs
[ApiController]
[Route("api/v1/gst")]
[Authorize]
[RequireModule(ModuleType.GSTCompliance)]
public class GstController : ControllerBase
{
    private readonly ISender _mediator;
    
    // E-Invoice Endpoints
    [HttpPost("einvoice/generate")]
    [RequirePermission(Permissions.GST.GenerateEInvoice)]
    public async Task<ActionResult<EInvoiceId>> GenerateEInvoice([FromBody] GenerateEInvoiceRequest request)
    {
        var command = new GenerateEInvoiceCommand { VoucherId = request.VoucherId };
        var result = await _mediator.Send(command);
        
        if (result.IsFailure)
            return BadRequest(new { Errors = result.Errors });
        
        return Accepted(new { EInvoiceId = result.Value, Message = "E-Invoice generation queued" });
    }
    
    [HttpGet("einvoice/{id:guid}")]
    public async Task<ActionResult<EInvoiceDto>> GetEInvoice(Guid id)
    {
        var query = new GetEInvoiceQuery { Id = id };
        return Ok(await _mediator.Send(query));
    }
    
    [HttpGet("einvoice/{id:guid}/pdf")]
    public async Task<IActionResult> GetEInvoicePdf(Guid id)
    {
        var query = new GetEInvoicePdfQuery { Id = id };
        var pdf = await _mediator.Send(query);
        return File(pdf, "application/pdf", $"EInvoice_{id}.pdf");
    }
    
    // GSTR-2B Reconciliation
    [HttpPost("gstr2b/fetch")]
    [RequirePermission(Permissions.GST.View)]
    public async Task<ActionResult> FetchGstr2b([FromBody] FetchGstr2bRequest request)
    {
        var command = new ReconcileGstr2bCommand { ReturnPeriod = request.ReturnPeriod };
        var result = await _mediator.Send(command);
        
        if (result.IsFailure)
            return BadRequest(result.Errors);
        
        return Ok(new { ReconciliationId = result.Value });
    }
    
    [HttpGet("gstr2b/reconciliation/{id:guid}")]
    public async Task<ActionResult<Gstr2bReconciliationDto>> GetReconciliation(Guid id)
    {
        return Ok(await _mediator.Send(new GetGstr2bReconciliationQuery { Id = id }));
    }
    
    [HttpPost("gstr2b/reconciliation/{id:guid}/action")]
    public async Task<ActionResult> TakeAction(Guid id, [FromBody] ReconciliationActionRequest request)
    {
        var command = new TakeReconciliationActionCommand
        {
            ReconciliationId = id,
            LineId = request.LineId,
            Action = request.Action,
            Note = request.Note
        };
        
        return Ok(await _mediator.Send(command));
    }
    
    // GSTR-1
    [HttpGet("gstr1/prepare")]
    public async Task<ActionResult<Gstr1PreparationDto>> PrepareGstr1([FromQuery] string returnPeriod)
    {
        return Ok(await _mediator.Send(new PrepareGstr1Query { ReturnPeriod = returnPeriod }));
    }
    
    [HttpPost("gstr1/save")]
    [RequirePermission(Permissions.GST.File)]
    public async Task<ActionResult> SaveGstr1([FromBody] SaveGstr1Request request)
    {
        var command = new SaveGstr1Command { ReturnPeriod = request.ReturnPeriod };
        return Ok(await _mediator.Send(command));
    }
    
    // Reports
    [HttpGet("reports/hsn-summary")]
    public async Task<ActionResult<HsnSummaryDto>> GetHsnSummary(
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate)
    {
        return Ok(await _mediator.Send(new GetHsnSummaryQuery { FromDate = fromDate, ToDate = toDate }));
    }
    
    [HttpGet("reports/tax-summary")]
    public async Task<ActionResult<GstTaxSummaryDto>> GetTaxSummary(
        [FromQuery] string returnPeriod)
    {
        return Ok(await _mediator.Send(new GetGstTaxSummaryQuery { ReturnPeriod = returnPeriod }));
    }
}

// API/Controllers/TdsController.cs
[ApiController]
[Route("api/v1/tds")]
[Authorize]
[RequireModule(ModuleType.GSTCompliance)]
public class TdsController : ControllerBase
{
    [HttpGet("transactions")]
    public async Task<ActionResult<List<TdsTransactionDto>>> GetTransactions(
        [FromQuery] string quarter,  // "Q1", "Q2", etc.
        [FromQuery] string financialYear)  // "2025-26"
    {
        return Ok(await _mediator.Send(new GetTdsTransactionsQuery { Quarter = quarter, FinancialYear = financialYear }));
    }
    
    [HttpPost("challan")]
    public async Task<ActionResult> RecordChallan([FromBody] RecordTdsChallanRequest request)
    {
        var command = new RecordTdsChallanCommand
        {
            TransactionIds = request.TransactionIds,
            ChallanNumber = request.ChallanNumber,
            BsrCode = request.BsrCode,
            DepositDate = request.DepositDate
        };
        
        return Ok(await _mediator.Send(command));
    }
    
    [HttpGet("form16a/{partyId:guid}")]
    public async Task<IActionResult> GenerateForm16A(Guid partyId, [FromQuery] string financialYear)
    {
        var pdf = await _mediator.Send(new GenerateForm16AQuery { PartyId = partyId, FinancialYear = financialYear });
        return File(pdf, "application/pdf", $"Form16A_{partyId}.pdf");
    }
}
```

---

## **7. Event Handlers (Integration with Core)**

```csharp
// Application/EventHandlers/VoucherPostedEventHandler.cs
public class VoucherPostedEventHandler : INotificationHandler<VoucherPostedEvent>
{
    private readonly IGstConfigRepository _gstConfigRepo;
    private readonly IEInvoiceRepository _eInvoiceRepo;
    private readonly IBackgroundJobClient _jobClient;
    
    public async Task Handle(VoucherPostedEvent notification, CancellationToken ct)
    {
        // Only process sales vouchers
        if (!IsSalesVoucher(notification.Type))
            return;
        
        var gstConfig = await _gstConfigRepo.GetAsync(ct);
        
        // Check if E-Invoice is mandatory for this business
        if (!gstConfig.IsEInvoiceMandatory)
            return;
        
        // Check if it's a B2B transaction
        if (!notification.HasGstinBuyer())
            return;
        
        // Auto-queue E-Invoice generation
        _jobClient.Enqueue<EInvoiceGenerationJob>(j => 
            j.GenerateFromVoucherAsync(notification.VoucherId)
        );
    }
    
    private bool IsSalesVoucher(VoucherType type) => 
        type is VoucherType.Sales or VoucherType.DebitNote;
}
```

---

## **8. Frontend Components**

### **8.1 GSTR-2B Reconciliation Screen**

```typescript
// src/app/(dashboard)/gst/gstr2b/page.tsx
'use client';

import { useState } from 'react';
import { useQuery, useMutation } from '@tanstack/react-query';
import { DataGrid } from '@/components/ui/data-grid';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Select } from '@/components/ui/select';

const statusColors = {
  Matched: 'bg-green-100 text-green-800',
  Mismatch: 'bg-yellow-100 text-yellow-800',
  MissingInBooks: 'bg-red-100 text-red-800',
  MissingInPortal: 'bg-orange-100 text-orange-800',
};

export default function Gstr2bReconciliationPage() {
  const [returnPeriod, setReturnPeriod] = useState('012026');
  const [filter, setFilter] = useState<string>('all');
  
  const { data: reconciliation, isLoading, refetch } = useQuery({
    queryKey: ['gstr2b-reconciliation', returnPeriod],
    queryFn: () => fetch(`/api/v1/gst/gstr2b/reconciliation?period=${returnPeriod}`).then(r => r.json()),
  });
  
  const fetchMutation = useMutation({
    mutationFn: () => fetch('/api/v1/gst/gstr2b/fetch', {
      method: 'POST',
      body: JSON.stringify({ returnPeriod }),
    }).then(r => r.json()),
    onSuccess: () => refetch(),
  });
  
  const columns = [
    { field: 'vendorGstin', headerName: 'Vendor GSTIN', width: 180 },
    { field: 'vendorName', headerName: 'Vendor Name', width: 200 },
    { field: 'invoiceNumber', headerName: 'Invoice No.', width: 150 },
    { field: 'invoiceDate', headerName: 'Date', width: 100 },
    { 
      field: 'totalTax', 
      headerName: 'Tax Amount', 
      width: 120,
      valueFormatter: (value: number) => `₹${value.toLocaleString('en-IN')}`
    },
    {
      field: 'status',
      headerName: 'Status',
      width: 150,
      renderCell: (params: any) => (
        <Badge className={statusColors[params.value as keyof typeof statusColors]}>
          {params.value}
        </Badge>
      )
    },
    {
      field: 'isDefaultingVendor',
      headerName: 'Vendor Filing',
      width: 120,
      renderCell: (params: any) => (
        params.value 
          ? <Badge className="bg-red-100 text-red-800">⚠️ Not Filed</Badge>
          : <Badge className="bg-green-100 text-green-800">✓ Filed</Badge>
      )
    },
    {
      field: 'actions',
      headerName: 'Actions',
      width: 200,
      renderCell: (params: any) => (
        <div className="flex gap-2">
          <Button size="sm" variant="outline" onClick={() => handleAction(params.row.id, 'PaymentBlocked')}>
            Block Payment
          </Button>
          <Button size="sm" variant="outline" onClick={() => sendVendorEmail(params.row)}>
            Send Email
          </Button>
        </div>
      )
    }
  ];
  
  return (
    <div className="p-6 space-y-6">
      {/* Header */}
      <div className="flex justify-between items-center">
        <div>
          <h1 className="text-2xl font-bold">GSTR-2B Reconciliation</h1>
          <p className="text-gray-500">Match portal data with your purchase register</p>
        </div>
        
        <div className="flex gap-4">
          <Select value={returnPeriod} onValueChange={setReturnPeriod}>
            {/* Period options */}
          </Select>
          <Button onClick={() => fetchMutation.mutate()} loading={fetchMutation.isPending}>
            Fetch from Portal
          </Button>
        </div>
      </div>
      
      {/* Summary Cards */}
      {reconciliation && (
        <div className="grid grid-cols-5 gap-4">
          <SummaryCard title="Total Invoices" value={reconciliation.totalInvoicesFromPortal} />
          <SummaryCard title="Matched" value={reconciliation.matchedCount} className="text-green-600" />
          <SummaryCard title="Mismatched" value={reconciliation.mismatchedCount} className="text-yellow-600" />
          <SummaryCard title="Missing in Books" value={reconciliation.missingInBooksCount} className="text-red-600" />
          <SummaryCard title="ITC at Risk" value={`₹${reconciliation.itcAtRisk.toLocaleString('en-IN')}`} className="text-red-600 font-bold" />
        </div>
      )}
      
      {/* Filter Tabs */}
      <div className="flex gap-2 border-b">
        {['all', 'Matched', 'Mismatch', 'MissingInBooks', 'MissingInPortal'].map(status => (
          <button
            key={status}
            onClick={() => setFilter(status)}
            className={`px-4 py-2 ${filter === status ? 'border-b-2 border-blue-500 font-medium' : ''}`}
          >
            {status === 'all' ? 'All' : status}
          </button>
        ))}
      </div>
      
      {/* Data Grid */}
      <DataGrid
        rows={reconciliation?.lines.filter(l => filter === 'all' || l.status === filter) || []}
        columns={columns}
        loading={isLoading}
        pageSize={25}
      />
    </div>
  );
}
```

---

## **9. Database Schema**

```sql
-- GST Module Tables

CREATE TABLE gst_configurations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL UNIQUE REFERENCES tenants(id),
    primary_gstin VARCHAR(15) NOT NULL,
    legal_name VARCHAR(200) NOT NULL,
    trade_name VARCHAR(200),
    state_code VARCHAR(2) NOT NULL,
    registration_type VARCHAR(30) NOT NULL,
    annual_turnover DECIMAL(18,2) DEFAULT 0,
    gsp_credentials BYTEA,  -- Encrypted
    einvoice_settings JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE e_invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    source_voucher_id UUID NOT NULL,
    source_document_number VARCHAR(50) NOT NULL,
    
    irn VARCHAR(64),
    ack_number VARCHAR(50),
    ack_date TIMESTAMPTZ,
    signed_qr_code TEXT,
    signed_invoice TEXT,
    
    invoice_type VARCHAR(20) NOT NULL,
    invoice_date DATE NOT NULL,
    invoice_number VARCHAR(50) NOT NULL,
    
    supplier_details JSONB NOT NULL,
    buyer_details JSONB NOT NULL,
    ship_to_details JSONB,
    
    total_value DECIMAL(18,2) NOT NULL,
    taxable_value DECIMAL(18,2) NOT NULL,
    cgst_amount DECIMAL(18,2) DEFAULT 0,
    sgst_amount DECIMAL(18,2) DEFAULT 0,
    igst_amount DECIMAL(18,2) DEFAULT 0,
    cess_amount DECIMAL(18,2) DEFAULT 0,
    total_invoice_value DECIMAL(18,2) NOT NULL,
    
    status VARCHAR(20) NOT NULL DEFAULT 'Pending',
    error_message TEXT,
    retry_count INT DEFAULT 0,
    
    is_cancelled BOOLEAN DEFAULT FALSE,
    cancel_reason TEXT,
    cancelled_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE e_invoice_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    e_invoice_id UUID NOT NULL REFERENCES e_invoices(id) ON DELETE CASCADE,
    sl_no INT NOT NULL,
    product_name VARCHAR(300) NOT NULL,
    hsn_code VARCHAR(8) NOT NULL,
    uom VARCHAR(10),
    quantity DECIMAL(18,4),
    unit_price DECIMAL(18,4),
    discount DECIMAL(18,2) DEFAULT 0,
    taxable_value DECIMAL(18,2) NOT NULL,
    gst_rate DECIMAL(5,2) NOT NULL,
    cgst_amount DECIMAL(18,2) DEFAULT 0,
    sgst_amount DECIMAL(18,2) DEFAULT 0,
    igst_amount DECIMAL(18,2) DEFAULT 0,
    cess_amount DECIMAL(18,2) DEFAULT 0,
    total_item_value DECIMAL(18,2) NOT NULL
);

CREATE TABLE gstr2b_reconciliations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    return_period VARCHAR(6) NOT NULL,  -- "012026"
    fetched_at TIMESTAMPTZ NOT NULL,
    
    total_invoices_from_portal INT DEFAULT 0,
    matched_count INT DEFAULT 0,
    mismatched_count INT DEFAULT 0,
    missing_in_books_count INT DEFAULT 0,
    missing_in_portal_count INT DEFAULT 0,
    
    total_itc_available DECIMAL(18,2) DEFAULT 0,
    total_itc_matched DECIMAL(18,2) DEFAULT 0,
    itc_at_risk DECIMAL(18,2) DEFAULT 0,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_reconciliation_per_period UNIQUE (tenant_id, return_period)
);

CREATE TABLE gstr2b_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reconciliation_id UUID NOT NULL REFERENCES gstr2b_reconciliations(id) ON DELETE CASCADE,
    
    vendor_gstin VARCHAR(15) NOT NULL,
    vendor_name VARCHAR(200),
    invoice_number VARCHAR(50) NOT NULL,
    invoice_date DATE NOT NULL,
    taxable_value DECIMAL(18,2) NOT NULL,
    igst_amount DECIMAL(18,2) DEFAULT 0,
    cgst_amount DECIMAL(18,2) DEFAULT 0,
    sgst_amount DECIMAL(18,2) DEFAULT 0,
    cess_amount DECIMAL(18,2) DEFAULT 0,
    
    status VARCHAR(30) NOT NULL DEFAULT 'Pending',
    matched_voucher_id UUID,
    mismatch_details TEXT,
    
    vendor_filing_status VARCHAR(20),
    
    action_taken VARCHAR(30),
    action_note TEXT,
    action_taken_at TIMESTAMPTZ,
    action_taken_by UUID REFERENCES users(id)
);

CREATE TABLE tds_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    voucher_id UUID NOT NULL,
    party_ledger_id UUID NOT NULL REFERENCES ledgers(id),
    
    section_code VARCHAR(10) NOT NULL,
    nature_of_payment VARCHAR(100),
    
    party_name VARCHAR(200) NOT NULL,
    pan VARCHAR(10),
    party_type VARCHAR(20) NOT NULL,
    
    transaction_date DATE NOT NULL,
    gross_amount DECIMAL(18,2) NOT NULL,
    tds_rate DECIMAL(5,2) NOT NULL,
    tds_amount DECIMAL(18,2) NOT NULL,
    net_payable DECIMAL(18,2) NOT NULL,
    
    is_deposited BOOLEAN DEFAULT FALSE,
    challan_number VARCHAR(50),
    deposit_date DATE,
    bsr_code VARCHAR(10),
    
    is_certificate_issued BOOLEAN DEFAULT FALSE,
    certificate_number VARCHAR(50),
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_einvoices_tenant_status ON e_invoices(tenant_id, status);
CREATE INDEX idx_einvoices_date ON e_invoices(tenant_id, invoice_date);
CREATE INDEX idx_gstr2b_lines_status ON gstr2b_lines(reconciliation_id, status);
CREATE INDEX idx_tds_tenant_date ON tds_transactions(tenant_id, transaction_date);
```

---

## **10. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| E-Invoice Generation | Generate IRN within 3 seconds (including API call) |
| 30-Day Hard Stop | Block E-Invoice for invoices > 30 days old (no bypass) |
| GSTR-2B Fetch | Download 1000+ invoices in < 10 seconds |
| Reconciliation | Match 1000 invoices in < 5 seconds |
| Defaulting Vendor Detection | Flag vendors who haven't filed GST |
| HSN Validation | Enforce 4/6 digit based on turnover |
| GSTIN Validation | Validate checksum on every entry |

---

## **11. Definition of Done**

- [ ] E-Invoice generation with IRN + QR
- [ ] E-Invoice PDF with signed QR code
- [ ] E-Invoice cancellation
- [ ] GSTR-2B fetch and reconciliation
- [ ] GSTR-1 JSON preparation
- [ ] TDS calculation and tracking
- [ ] Form 16A generation
- [ ] All validations (GSTIN, HSN, 30-day rule)
- [ ] Async queue for API resilience
- [ ] GSP integration tests passing
- [ ] Performance benchmarks met

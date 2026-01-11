# **Phase 1: Core Accounting Engine**
**Duration:** 12 Weeks  
**Team:** Backend (3), Frontend (2), QA (1), DBA (1)  
**Dependencies:** Phase 0 Complete  

---

## **1. Overview**

The Core Accounting Engine is the **heart of AuditFlow ERP**. Every other module depends on it. This module MUST be:
- **Rock-solid:** Zero tolerance for calculation errors
- **Fast:** Sub-500ms voucher entry
- **Extensible:** Other modules plug into it via events
- **Compliant:** Built-in audit trail, double-entry enforcement

---

## **2. Domain Model**

### **2.1 Core Entities**

```csharp
// Domain/Entities/FinancialYear.cs
public class FinancialYear : AggregateRoot<FinancialYearId>
{
    public TenantId TenantId { get; private set; }
    public string Name { get; private set; }  // "2025-26"
    public DateOnly StartDate { get; private set; }  // April 1
    public DateOnly EndDate { get; private set; }    // March 31
    public bool IsClosed { get; private set; }
    public bool IsLocked { get; private set; }
    public DateTime? ClosedAt { get; private set; }
    public UserId? ClosedBy { get; private set; }
    
    // Books can be locked for periods (e.g., after GST filing)
    private readonly List<LockedPeriod> _lockedPeriods = new();
    public IReadOnlyCollection<LockedPeriod> LockedPeriods => _lockedPeriods.AsReadOnly();
    
    public void LockPeriod(DateOnly from, DateOnly to, string reason)
    {
        if (_lockedPeriods.Any(p => p.Overlaps(from, to)))
            throw new DomainException("Period already locked");
            
        _lockedPeriods.Add(new LockedPeriod(from, to, reason));
        AddDomainEvent(new PeriodLockedEvent(Id, from, to));
    }
}

// Domain/Entities/AccountGroup.cs
public class AccountGroup : AggregateRoot<AccountGroupId>
{
    public TenantId TenantId { get; private set; }
    public string Name { get; private set; }
    public string Code { get; private set; }
    public AccountGroupId? ParentGroupId { get; private set; }
    public AccountNature Nature { get; private set; }  // Assets, Liabilities, Income, Expense
    public AccountType Type { get; private set; }      // Primary or Subgroup
    public int Level { get; private set; }
    public bool IsSystemGroup { get; private set; }    // Cannot be deleted
    public bool IsActive { get; private set; }
    
    private readonly List<AccountGroup> _childGroups = new();
    public IReadOnlyCollection<AccountGroup> ChildGroups => _childGroups.AsReadOnly();
}

public enum AccountNature
{
    Assets = 1,
    Liabilities = 2,
    Income = 3,
    Expense = 4
}

// Domain/Entities/Ledger.cs
public class Ledger : AggregateRoot<LedgerId>
{
    public TenantId TenantId { get; private set; }
    public string Name { get; private set; }
    public string Code { get; private set; }
    public string? Alias { get; private set; }
    public AccountGroupId GroupId { get; private set; }
    public LedgerType Type { get; private set; }
    
    // Opening Balance
    public Money OpeningBalance { get; private set; }
    public BalanceType OpeningBalanceType { get; private set; }  // Dr or Cr
    
    // Current Balance (Computed)
    public Money CurrentBalance { get; private set; }
    public BalanceType CurrentBalanceType { get; private set; }
    
    // GST Configuration (if applicable)
    public GstLedgerConfig? GstConfig { get; private set; }
    
    // Bank Configuration (if Bank Account)
    public BankAccountConfig? BankConfig { get; private set; }
    
    // Party Configuration (if Debtor/Creditor)
    public PartyConfig? PartyConfig { get; private set; }
    
    public bool IsActive { get; private set; }
    public bool IsSystemLedger { get; private set; }
    
    // Credit Limit for parties
    public Money? CreditLimit { get; private set; }
    public int? CreditDays { get; private set; }
    
    public void UpdateBalance(Money amount, BalanceType type)
    {
        // Logic to update running balance
        if (type == CurrentBalanceType)
        {
            CurrentBalance = CurrentBalance.Add(amount);
        }
        else
        {
            if (amount.Amount > CurrentBalance.Amount)
            {
                CurrentBalance = new Money(amount.Amount - CurrentBalance.Amount, amount.Currency);
                CurrentBalanceType = type;
            }
            else
            {
                CurrentBalance = new Money(CurrentBalance.Amount - amount.Amount, amount.Currency);
            }
        }
    }
}

public enum LedgerType
{
    General = 1,
    BankAccount = 2,
    CashAccount = 3,
    SundryDebtor = 4,
    SundryCreditor = 5,
    DutiesAndTaxes = 6,
    FixedAsset = 7,
    StockInHand = 8
}

// Domain/ValueObjects/GstLedgerConfig.cs
public record GstLedgerConfig
{
    public string? Gstin { get; init; }
    public string? StateCode { get; init; }
    public GstRegistrationType RegistrationType { get; init; }
    public bool IsCompositionDealer { get; init; }
    public string? Pan { get; init; }
}

// Domain/ValueObjects/PartyConfig.cs
public record PartyConfig
{
    public string? ContactPerson { get; init; }
    public string? Phone { get; init; }
    public string? Email { get; init; }
    public Address? BillingAddress { get; init; }
    public Address? ShippingAddress { get; init; }
    public string? Pan { get; init; }
    public TdsConfig? TdsConfig { get; init; }
}
```

### **2.2 Voucher System (The Core)**

```csharp
// Domain/Entities/Voucher.cs
public class Voucher : AggregateRoot<VoucherId>
{
    public TenantId TenantId { get; private set; }
    public FinancialYearId FinancialYearId { get; private set; }
    
    // Voucher Identification
    public VoucherType Type { get; private set; }
    public string VoucherNumber { get; private set; }
    public string? ReferenceNumber { get; private set; }
    public DateOnly VoucherDate { get; private set; }
    
    // Narration
    public string? Narration { get; private set; }
    
    // Status
    public VoucherStatus Status { get; private set; }
    public bool IsPosted { get; private set; }
    public DateTime? PostedAt { get; private set; }
    public UserId? PostedBy { get; private set; }
    
    // Voucher Lines (Dr and Cr entries)
    private readonly List<VoucherLine> _lines = new();
    public IReadOnlyCollection<VoucherLine> Lines => _lines.AsReadOnly();
    
    // Totals (Computed)
    public Money TotalDebit => new Money(_lines.Where(l => l.Type == BalanceType.Debit).Sum(l => l.Amount.Amount), Currency.INR);
    public Money TotalCredit => new Money(_lines.Where(l => l.Type == BalanceType.Credit).Sum(l => l.Amount.Amount), Currency.INR);
    
    // GST Details (if applicable)
    public GstVoucherDetails? GstDetails { get; private set; }
    
    // Source Module (which module created this voucher)
    public string SourceModule { get; private set; } = "CoreAccounting";
    public string? SourceDocumentId { get; private set; }
    
    public void AddLine(LedgerId ledgerId, Money amount, BalanceType type, string? particulars = null)
    {
        var line = new VoucherLine(
            id: VoucherLineId.Create(),
            voucherId: Id,
            ledgerId: ledgerId,
            amount: amount,
            type: type,
            particulars: particulars,
            lineNumber: _lines.Count + 1
        );
        
        _lines.Add(line);
    }
    
    public Result Validate()
    {
        var errors = new List<string>();
        
        // Rule 1: Must have at least 2 lines
        if (_lines.Count < 2)
            errors.Add("Voucher must have at least 2 entries");
        
        // Rule 2: Debit must equal Credit (Double-Entry)
        if (TotalDebit.Amount != TotalCredit.Amount)
            errors.Add($"Debit ({TotalDebit}) does not equal Credit ({TotalCredit})");
        
        // Rule 3: All amounts must be positive
        if (_lines.Any(l => l.Amount.Amount <= 0))
            errors.Add("All amounts must be positive");
        
        // Rule 4: Date must be within financial year
        // (Validated in application layer with FY context)
        
        return errors.Any() 
            ? Result.Failure(errors) 
            : Result.Success();
    }
    
    public void Post()
    {
        var validation = Validate();
        if (validation.IsFailure)
            throw new DomainException($"Cannot post invalid voucher: {string.Join(", ", validation.Errors)}");
        
        IsPosted = true;
        PostedAt = DateTime.UtcNow;
        Status = VoucherStatus.Posted;
        
        // Raise domain event for ledger balance updates
        AddDomainEvent(new VoucherPostedEvent(this));
    }
}

// Domain/Entities/VoucherLine.cs
public class VoucherLine : Entity<VoucherLineId>
{
    public VoucherId VoucherId { get; private set; }
    public LedgerId LedgerId { get; private set; }
    public Money Amount { get; private set; }
    public BalanceType Type { get; private set; }  // Debit or Credit
    public string? Particulars { get; private set; }
    public int LineNumber { get; private set; }
    
    // Cost Center (optional)
    public CostCenterId? CostCenterId { get; private set; }
    
    // Bill Reference (for party accounts - tracking)
    public string? BillReference { get; private set; }
    public BillType? BillType { get; private set; }
    
    // Inventory Reference (set by Inventory module)
    public Guid? InventoryTransactionId { get; private set; }
}

public enum VoucherType
{
    // Standard Vouchers
    Journal = 1,
    Payment = 2,
    Receipt = 3,
    Contra = 4,
    
    // Sales Vouchers
    Sales = 10,
    SalesReturn = 11,
    CreditNote = 12,
    
    // Purchase Vouchers
    Purchase = 20,
    PurchaseReturn = 21,
    DebitNote = 22,
    
    // Inventory Vouchers (Created by Inventory Module)
    StockTransfer = 30,
    StockJournal = 31,
    Manufacturing = 32,
    
    // Opening Vouchers
    OpeningBalance = 90
}

public enum VoucherStatus
{
    Draft = 1,
    Pending = 2,      // Awaiting approval
    Posted = 3,       // Finalized
    Cancelled = 4,    // Voided
    Reversed = 5      // Reversed by another voucher
}
```

### **2.3 Value Objects**

```csharp
// Domain/ValueObjects/Money.cs
public record Money
{
    public decimal Amount { get; }
    public Currency Currency { get; }
    
    public Money(decimal amount, Currency currency)
    {
        Amount = Math.Round(amount, 2, MidpointRounding.AwayFromZero);
        Currency = currency;
    }
    
    public static Money INR(decimal amount) => new(amount, Currency.INR);
    public static Money Zero => new(0, Currency.INR);
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException("Cannot add different currencies");
        return new Money(Amount + other.Amount, Currency);
    }
    
    public Money Subtract(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException("Cannot subtract different currencies");
        return new Money(Amount - other.Amount, Currency);
    }
    
    public Money Multiply(decimal factor) => new(Amount * factor, Currency);
    
    public override string ToString() => $"{Currency.Symbol}{Amount:N2}";
}

// Domain/ValueObjects/VoucherNumber.cs
public record VoucherNumber
{
    public string Prefix { get; }
    public int Number { get; }
    public string Suffix { get; }
    
    public VoucherNumber(string prefix, int number, string suffix = "")
    {
        Prefix = prefix;
        Number = number;
        Suffix = suffix;
    }
    
    public override string ToString() => $"{Prefix}{Number:D6}{Suffix}";
    
    public static VoucherNumber Parse(string value)
    {
        // Parse logic
        throw new NotImplementedException();
    }
}
```

---

## **3. Application Layer (CQRS)**

### **3.1 Commands**

```csharp
// Application/Commands/CreateVoucher/CreateVoucherCommand.cs
public record CreateVoucherCommand : IRequest<Result<VoucherId>>
{
    public VoucherType Type { get; init; }
    public DateOnly VoucherDate { get; init; }
    public string? ReferenceNumber { get; init; }
    public string? Narration { get; init; }
    public List<VoucherLineDto> Lines { get; init; } = new();
    public bool PostImmediately { get; init; } = true;
}

public record VoucherLineDto
{
    public Guid LedgerId { get; init; }
    public decimal Amount { get; init; }
    public BalanceType Type { get; init; }
    public string? Particulars { get; init; }
    public Guid? CostCenterId { get; init; }
}

// Application/Commands/CreateVoucher/CreateVoucherCommandHandler.cs
public class CreateVoucherCommandHandler : IRequestHandler<CreateVoucherCommand, Result<VoucherId>>
{
    private readonly IVoucherRepository _voucherRepo;
    private readonly ILedgerRepository _ledgerRepo;
    private readonly IFinancialYearRepository _fyRepo;
    private readonly IVoucherNumberGenerator _numberGenerator;
    private readonly IUnitOfWork _unitOfWork;
    
    public async Task<Result<VoucherId>> Handle(CreateVoucherCommand request, CancellationToken ct)
    {
        // 1. Validate Financial Year
        var fy = await _fyRepo.GetActiveAsync(ct);
        if (fy == null)
            return Result.Failure<VoucherId>("No active financial year");
        
        if (!fy.IsDateWithinYear(request.VoucherDate))
            return Result.Failure<VoucherId>("Voucher date is outside financial year");
        
        // 2. Check if period is locked
        if (fy.IsPeriodLocked(request.VoucherDate))
            return Result.Failure<VoucherId>("Period is locked for this date");
        
        // 3. Validate all ledgers exist and are active
        var ledgerIds = request.Lines.Select(l => LedgerId.From(l.LedgerId)).ToList();
        var ledgers = await _ledgerRepo.GetByIdsAsync(ledgerIds, ct);
        
        if (ledgers.Count != ledgerIds.Count)
            return Result.Failure<VoucherId>("One or more ledgers not found");
        
        if (ledgers.Any(l => !l.IsActive))
            return Result.Failure<VoucherId>("One or more ledgers are inactive");
        
        // 4. Generate voucher number
        var voucherNumber = await _numberGenerator.GenerateAsync(request.Type, fy.Id, ct);
        
        // 5. Create Voucher
        var voucher = Voucher.Create(
            type: request.Type,
            voucherNumber: voucherNumber,
            voucherDate: request.VoucherDate,
            financialYearId: fy.Id,
            referenceNumber: request.ReferenceNumber,
            narration: request.Narration
        );
        
        // 6. Add Lines
        foreach (var line in request.Lines)
        {
            voucher.AddLine(
                ledgerId: LedgerId.From(line.LedgerId),
                amount: Money.INR(line.Amount),
                type: line.Type,
                particulars: line.Particulars
            );
        }
        
        // 7. Validate (Double-Entry Check)
        var validation = voucher.Validate();
        if (validation.IsFailure)
            return Result.Failure<VoucherId>(validation.Errors);
        
        // 8. Post if requested
        if (request.PostImmediately)
        {
            voucher.Post();
        }
        
        // 9. Save
        await _voucherRepo.AddAsync(voucher, ct);
        await _unitOfWork.SaveChangesAsync(ct);
        
        return Result.Success(voucher.Id);
    }
}

// Application/Commands/CreateLedger/CreateLedgerCommand.cs
public record CreateLedgerCommand : IRequest<Result<LedgerId>>
{
    public string Name { get; init; } = string.Empty;
    public string? Alias { get; init; }
    public Guid GroupId { get; init; }
    public decimal OpeningBalance { get; init; }
    public BalanceType OpeningBalanceType { get; init; }
    
    // Party Details (optional)
    public PartyDetailsDto? PartyDetails { get; init; }
    
    // GST Details (optional)
    public GstDetailsDto? GstDetails { get; init; }
    
    // Bank Details (optional)
    public BankDetailsDto? BankDetails { get; init; }
}

public record PartyDetailsDto
{
    public string? ContactPerson { get; init; }
    public string? Phone { get; init; }
    public string? Email { get; init; }
    public string? Pan { get; init; }
    public AddressDto? BillingAddress { get; init; }
    public AddressDto? ShippingAddress { get; init; }
    public int? CreditDays { get; init; }
    public decimal? CreditLimit { get; init; }
}
```

### **3.2 Queries**

```csharp
// Application/Queries/GetLedgerStatement/GetLedgerStatementQuery.cs
public record GetLedgerStatementQuery : IRequest<LedgerStatementDto>
{
    public Guid LedgerId { get; init; }
    public DateOnly FromDate { get; init; }
    public DateOnly ToDate { get; init; }
}

public record LedgerStatementDto
{
    public Guid LedgerId { get; init; }
    public string LedgerName { get; init; } = string.Empty;
    public decimal OpeningBalance { get; init; }
    public BalanceType OpeningBalanceType { get; init; }
    public List<LedgerStatementLineDto> Transactions { get; init; } = new();
    public decimal ClosingBalance { get; init; }
    public BalanceType ClosingBalanceType { get; init; }
}

public record LedgerStatementLineDto
{
    public DateOnly Date { get; init; }
    public string VoucherNumber { get; init; } = string.Empty;
    public VoucherType VoucherType { get; init; }
    public string Particulars { get; init; } = string.Empty;
    public decimal? Debit { get; init; }
    public decimal? Credit { get; init; }
    public decimal RunningBalance { get; init; }
    public BalanceType BalanceType { get; init; }
}

// Application/Queries/GetTrialBalance/GetTrialBalanceQuery.cs
public record GetTrialBalanceQuery : IRequest<TrialBalanceDto>
{
    public DateOnly AsOnDate { get; init; }
    public bool ShowZeroBalance { get; init; } = false;
}

public record TrialBalanceDto
{
    public DateOnly AsOnDate { get; init; }
    public List<TrialBalanceLineDto> Lines { get; init; } = new();
    public decimal TotalDebit => Lines.Sum(l => l.DebitBalance);
    public decimal TotalCredit => Lines.Sum(l => l.CreditBalance);
    public bool IsBalanced => TotalDebit == TotalCredit;
}

// Application/Queries/GetDayBook/GetDayBookQuery.cs
public record GetDayBookQuery : IRequest<DayBookDto>
{
    public DateOnly Date { get; init; }
    public VoucherType? FilterByType { get; init; }
}
```

---

## **4. Infrastructure Layer**

### **4.1 Database Schema**

```sql
-- Core Accounting Tables
CREATE TABLE financial_years (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(20) NOT NULL,  -- "2025-26"
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_closed BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    closed_at TIMESTAMPTZ,
    closed_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_fy_per_tenant UNIQUE (tenant_id, name)
);

CREATE TABLE account_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20),
    parent_group_id UUID REFERENCES account_groups(id),
    nature VARCHAR(20) NOT NULL CHECK (nature IN ('Assets', 'Liabilities', 'Income', 'Expense')),
    level INT NOT NULL DEFAULT 1,
    is_system_group BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_group_name_per_tenant UNIQUE (tenant_id, name)
);

CREATE TABLE ledgers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(200) NOT NULL,
    code VARCHAR(50),
    alias VARCHAR(100),
    group_id UUID NOT NULL REFERENCES account_groups(id),
    ledger_type VARCHAR(30) NOT NULL,
    
    -- Opening Balance
    opening_balance DECIMAL(18,2) DEFAULT 0,
    opening_balance_type VARCHAR(6) CHECK (opening_balance_type IN ('Debit', 'Credit')),
    
    -- Current Balance (Updated by triggers)
    current_balance DECIMAL(18,2) DEFAULT 0,
    current_balance_type VARCHAR(6) CHECK (current_balance_type IN ('Debit', 'Credit')),
    
    -- GST Config (JSONB for flexibility)
    gst_config JSONB,
    
    -- Bank Config
    bank_config JSONB,
    
    -- Party Config
    party_config JSONB,
    
    credit_limit DECIMAL(18,2),
    credit_days INT,
    
    is_active BOOLEAN DEFAULT TRUE,
    is_system_ledger BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ,
    
    CONSTRAINT unique_ledger_name_per_tenant UNIQUE (tenant_id, name)
);

CREATE TABLE vouchers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    financial_year_id UUID NOT NULL REFERENCES financial_years(id),
    
    voucher_type VARCHAR(30) NOT NULL,
    voucher_number VARCHAR(50) NOT NULL,
    reference_number VARCHAR(100),
    voucher_date DATE NOT NULL,
    
    narration TEXT,
    
    status VARCHAR(20) NOT NULL DEFAULT 'Draft',
    is_posted BOOLEAN DEFAULT FALSE,
    posted_at TIMESTAMPTZ,
    posted_by UUID REFERENCES users(id),
    
    -- Source Module Tracking
    source_module VARCHAR(50) DEFAULT 'CoreAccounting',
    source_document_id UUID,
    
    -- GST Details
    gst_details JSONB,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    created_by UUID NOT NULL REFERENCES users(id),
    updated_at TIMESTAMPTZ,
    updated_by UUID REFERENCES users(id),
    
    CONSTRAINT unique_voucher_number UNIQUE (tenant_id, financial_year_id, voucher_type, voucher_number)
);

CREATE TABLE voucher_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    voucher_id UUID NOT NULL REFERENCES vouchers(id) ON DELETE CASCADE,
    ledger_id UUID NOT NULL REFERENCES ledgers(id),
    
    amount DECIMAL(18,2) NOT NULL CHECK (amount > 0),
    balance_type VARCHAR(6) NOT NULL CHECK (balance_type IN ('Debit', 'Credit')),
    particulars TEXT,
    line_number INT NOT NULL,
    
    -- Cost Center
    cost_center_id UUID,
    
    -- Bill Reference (for tracking)
    bill_reference VARCHAR(100),
    bill_type VARCHAR(20),
    
    -- Link to Inventory Transaction (if applicable)
    inventory_transaction_id UUID,
    
    CONSTRAINT unique_line_per_voucher UNIQUE (voucher_id, line_number)
);

-- Indexes for Performance
CREATE INDEX idx_vouchers_tenant_date ON vouchers(tenant_id, voucher_date);
CREATE INDEX idx_vouchers_type ON vouchers(tenant_id, voucher_type);
CREATE INDEX idx_voucher_lines_ledger ON voucher_lines(ledger_id);
CREATE INDEX idx_ledgers_group ON ledgers(group_id);

-- Trigger for Updating Ledger Balances
CREATE OR REPLACE FUNCTION update_ledger_balance()
RETURNS TRIGGER AS $$
BEGIN
    -- Update ledger current balance when voucher is posted
    IF NEW.is_posted = TRUE AND OLD.is_posted = FALSE THEN
        -- Update each ledger's balance
        UPDATE ledgers l
        SET current_balance = (
            SELECT COALESCE(SUM(
                CASE 
                    WHEN vl.balance_type = 'Debit' THEN vl.amount
                    ELSE -vl.amount
                END
            ), 0) + l.opening_balance
            FROM voucher_lines vl
            JOIN vouchers v ON vl.voucher_id = v.id
            WHERE vl.ledger_id = l.id
            AND v.is_posted = TRUE
        ),
        updated_at = NOW()
        WHERE l.id IN (
            SELECT ledger_id FROM voucher_lines WHERE voucher_id = NEW.id
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_ledger_balance
AFTER UPDATE ON vouchers
FOR EACH ROW
EXECUTE FUNCTION update_ledger_balance();
```

### **4.2 Repository Implementation**

```csharp
// Infrastructure/Repositories/VoucherRepository.cs
public class VoucherRepository : IVoucherRepository
{
    private readonly AccountingDbContext _context;
    
    public async Task<Voucher?> GetByIdAsync(VoucherId id, CancellationToken ct)
    {
        return await _context.Vouchers
            .Include(v => v.Lines)
            .FirstOrDefaultAsync(v => v.Id == id, ct);
    }
    
    public async Task<List<Voucher>> GetByDateRangeAsync(
        DateOnly from, 
        DateOnly to, 
        CancellationToken ct)
    {
        return await _context.Vouchers
            .Include(v => v.Lines)
            .Where(v => v.VoucherDate >= from && v.VoucherDate <= to)
            .OrderBy(v => v.VoucherDate)
            .ThenBy(v => v.VoucherNumber)
            .ToListAsync(ct);
    }
    
    public async Task AddAsync(Voucher voucher, CancellationToken ct)
    {
        await _context.Vouchers.AddAsync(voucher, ct);
    }
}

// Infrastructure/Repositories/LedgerRepository.cs (with Dapper for complex queries)
public class LedgerRepository : ILedgerRepository
{
    private readonly AccountingDbContext _context;
    private readonly IDbConnection _connection;
    
    public async Task<LedgerStatementDto> GetStatementAsync(
        LedgerId ledgerId,
        DateOnly from,
        DateOnly to,
        CancellationToken ct)
    {
        const string sql = @"
            WITH opening AS (
                SELECT 
                    COALESCE(SUM(CASE WHEN vl.balance_type = 'Debit' THEN vl.amount ELSE -vl.amount END), 0) 
                    + l.opening_balance AS balance
                FROM ledgers l
                LEFT JOIN voucher_lines vl ON vl.ledger_id = l.id
                LEFT JOIN vouchers v ON vl.voucher_id = v.id AND v.is_posted = TRUE
                WHERE l.id = @LedgerId
                AND (v.voucher_date < @FromDate OR v.id IS NULL)
                GROUP BY l.id, l.opening_balance
            ),
            transactions AS (
                SELECT 
                    v.voucher_date,
                    v.voucher_number,
                    v.voucher_type,
                    COALESCE(vl.particulars, v.narration) AS particulars,
                    CASE WHEN vl.balance_type = 'Debit' THEN vl.amount END AS debit,
                    CASE WHEN vl.balance_type = 'Credit' THEN vl.amount END AS credit,
                    SUM(CASE WHEN vl.balance_type = 'Debit' THEN vl.amount ELSE -vl.amount END) 
                        OVER (ORDER BY v.voucher_date, v.voucher_number) AS running_total
                FROM voucher_lines vl
                JOIN vouchers v ON vl.voucher_id = v.id
                WHERE vl.ledger_id = @LedgerId
                AND v.is_posted = TRUE
                AND v.voucher_date BETWEEN @FromDate AND @ToDate
                ORDER BY v.voucher_date, v.voucher_number
            )
            SELECT * FROM opening, transactions";
        
        // Execute and map results
        var result = await _connection.QueryAsync<dynamic>(sql, new { LedgerId = ledgerId.Value, FromDate = from, ToDate = to });
        
        // Map to DTO
        return MapToStatement(result);
    }
}
```

---

## **5. API Endpoints**

### **5.1 Voucher API**

```csharp
// API/Controllers/VouchersController.cs
[ApiController]
[Route("api/v1/accounting/vouchers")]
[Authorize]
public class VouchersController : ControllerBase
{
    private readonly ISender _mediator;
    
    [HttpPost]
    [RequirePermission(Permissions.Voucher.Create)]
    public async Task<ActionResult<VoucherId>> Create([FromBody] CreateVoucherCommand command)
    {
        var result = await _mediator.Send(command);
        
        if (result.IsFailure)
            return BadRequest(result.Errors);
        
        return CreatedAtAction(nameof(GetById), new { id = result.Value }, result.Value);
    }
    
    [HttpGet("{id:guid}")]
    [RequirePermission(Permissions.Voucher.View)]
    public async Task<ActionResult<VoucherDto>> GetById(Guid id)
    {
        var query = new GetVoucherByIdQuery { VoucherId = id };
        var result = await _mediator.Send(query);
        
        if (result == null)
            return NotFound();
        
        return Ok(result);
    }
    
    [HttpGet]
    [RequirePermission(Permissions.Voucher.View)]
    public async Task<ActionResult<PagedResult<VoucherSummaryDto>>> GetAll(
        [FromQuery] DateOnly? fromDate,
        [FromQuery] DateOnly? toDate,
        [FromQuery] VoucherType? type,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 50)
    {
        var query = new GetVouchersQuery
        {
            FromDate = fromDate ?? DateOnly.FromDateTime(DateTime.Now.AddMonths(-1)),
            ToDate = toDate ?? DateOnly.FromDateTime(DateTime.Now),
            Type = type,
            Page = page,
            PageSize = pageSize
        };
        
        return Ok(await _mediator.Send(query));
    }
    
    [HttpPost("{id:guid}/post")]
    [RequirePermission(Permissions.Voucher.Approve)]
    public async Task<ActionResult> Post(Guid id)
    {
        var command = new PostVoucherCommand { VoucherId = id };
        var result = await _mediator.Send(command);
        
        if (result.IsFailure)
            return BadRequest(result.Errors);
        
        return Ok();
    }
    
    [HttpPost("{id:guid}/reverse")]
    [RequirePermission(Permissions.Voucher.Delete)]
    public async Task<ActionResult<VoucherId>> Reverse(Guid id, [FromBody] ReverseVoucherRequest request)
    {
        var command = new ReverseVoucherCommand 
        { 
            VoucherId = id,
            ReversalDate = request.ReversalDate,
            Reason = request.Reason
        };
        
        var result = await _mediator.Send(command);
        
        if (result.IsFailure)
            return BadRequest(result.Errors);
        
        return Ok(result.Value);  // Returns the reversal voucher ID
    }
}

// API/Controllers/LedgersController.cs
[ApiController]
[Route("api/v1/accounting/ledgers")]
[Authorize]
public class LedgersController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<List<LedgerDto>>> GetAll(
        [FromQuery] string? search,
        [FromQuery] Guid? groupId,
        [FromQuery] LedgerType? type)
    {
        var query = new GetLedgersQuery { Search = search, GroupId = groupId, Type = type };
        return Ok(await _mediator.Send(query));
    }
    
    [HttpGet("{id:guid}/statement")]
    public async Task<ActionResult<LedgerStatementDto>> GetStatement(
        Guid id,
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate)
    {
        var query = new GetLedgerStatementQuery 
        { 
            LedgerId = id, 
            FromDate = fromDate, 
            ToDate = toDate 
        };
        return Ok(await _mediator.Send(query));
    }
    
    [HttpGet("search")]
    public async Task<ActionResult<List<LedgerSearchResultDto>>> Search(
        [FromQuery] string q,
        [FromQuery] int limit = 10)
    {
        // Fast search for voucher entry autocomplete
        var query = new SearchLedgersQuery { SearchTerm = q, Limit = limit };
        return Ok(await _mediator.Send(query));
    }
}

// API/Controllers/ReportsController.cs
[ApiController]
[Route("api/v1/accounting/reports")]
[Authorize]
[RequirePermission(Permissions.Reports.View)]
public class ReportsController : ControllerBase
{
    [HttpGet("trial-balance")]
    public async Task<ActionResult<TrialBalanceDto>> GetTrialBalance([FromQuery] DateOnly asOnDate)
    {
        return Ok(await _mediator.Send(new GetTrialBalanceQuery { AsOnDate = asOnDate }));
    }
    
    [HttpGet("profit-loss")]
    public async Task<ActionResult<ProfitLossDto>> GetProfitLoss(
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate)
    {
        return Ok(await _mediator.Send(new GetProfitLossQuery { FromDate = fromDate, ToDate = toDate }));
    }
    
    [HttpGet("balance-sheet")]
    public async Task<ActionResult<BalanceSheetDto>> GetBalanceSheet([FromQuery] DateOnly asOnDate)
    {
        return Ok(await _mediator.Send(new GetBalanceSheetQuery { AsOnDate = asOnDate }));
    }
    
    [HttpGet("day-book")]
    public async Task<ActionResult<DayBookDto>> GetDayBook([FromQuery] DateOnly date)
    {
        return Ok(await _mediator.Send(new GetDayBookQuery { Date = date }));
    }
    
    [HttpGet("cash-book")]
    public async Task<ActionResult<CashBookDto>> GetCashBook(
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate)
    {
        return Ok(await _mediator.Send(new GetCashBookQuery { FromDate = fromDate, ToDate = toDate }));
    }
}
```

---

## **6. Frontend Components**

### **6.1 Voucher Entry Screen (Keyboard-First)**

```typescript
// src/app/(dashboard)/vouchers/new/page.tsx
'use client';

import { useState, useRef, useCallback } from 'react';
import { useForm, useFieldArray } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useHotkeys } from 'react-hotkeys-hook';
import { LedgerSearch } from '@/components/accounting/ledger-search';
import { MoneyInput } from '@/components/ui/money-input';

const voucherSchema = z.object({
  type: z.enum(['Journal', 'Payment', 'Receipt', 'Contra']),
  date: z.string(),
  referenceNumber: z.string().optional(),
  narration: z.string().optional(),
  lines: z.array(z.object({
    ledgerId: z.string().uuid(),
    ledgerName: z.string(),
    amount: z.number().positive(),
    type: z.enum(['Debit', 'Credit']),
    particulars: z.string().optional(),
  })).min(2),
});

type VoucherForm = z.infer<typeof voucherSchema>;

export default function NewVoucherPage() {
  const formRef = useRef<HTMLFormElement>(null);
  const [activeLineIndex, setActiveLineIndex] = useState(0);
  
  const { register, control, handleSubmit, watch, setValue, formState } = useForm<VoucherForm>({
    resolver: zodResolver(voucherSchema),
    defaultValues: {
      type: 'Journal',
      date: new Date().toISOString().split('T')[0],
      lines: [
        { ledgerId: '', ledgerName: '', amount: 0, type: 'Debit' },
        { ledgerId: '', ledgerName: '', amount: 0, type: 'Credit' },
      ],
    },
  });
  
  const { fields, append, remove } = useFieldArray({ control, name: 'lines' });
  
  // Keyboard shortcuts (Tally-like)
  useHotkeys('f5', () => setValue('type', 'Payment'), { enableOnFormTags: true });
  useHotkeys('f6', () => setValue('type', 'Receipt'), { enableOnFormTags: true });
  useHotkeys('f7', () => setValue('type', 'Journal'), { enableOnFormTags: true });
  useHotkeys('f8', () => setValue('type', 'Contra'), { enableOnFormTags: true });
  useHotkeys('ctrl+enter', () => handleSubmit(onSubmit)(), { enableOnFormTags: true });
  useHotkeys('ctrl+n', () => append({ ledgerId: '', ledgerName: '', amount: 0, type: 'Debit' }), { enableOnFormTags: true });
  
  // Calculate totals
  const lines = watch('lines');
  const totalDebit = lines.filter(l => l.type === 'Debit').reduce((sum, l) => sum + (l.amount || 0), 0);
  const totalCredit = lines.filter(l => l.type === 'Credit').reduce((sum, l) => sum + (l.amount || 0), 0);
  const isBalanced = totalDebit === totalCredit;
  
  const onSubmit = async (data: VoucherForm) => {
    // Submit to API
    const response = await fetch('/api/v1/accounting/vouchers', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        type: data.type,
        voucherDate: data.date,
        referenceNumber: data.referenceNumber,
        narration: data.narration,
        lines: data.lines.map(l => ({
          ledgerId: l.ledgerId,
          amount: l.amount,
          type: l.type,
          particulars: l.particulars,
        })),
        postImmediately: true,
      }),
    });
    
    if (response.ok) {
      // Success - redirect or show message
    }
  };
  
  return (
    <div className="max-w-5xl mx-auto p-6">
      {/* Shortcut hints */}
      <div className="flex gap-4 mb-4 text-sm text-gray-500">
        <span>F5: Payment</span>
        <span>F6: Receipt</span>
        <span>F7: Journal</span>
        <span>F8: Contra</span>
        <span>Ctrl+Enter: Save</span>
      </div>
      
      <form ref={formRef} onSubmit={handleSubmit(onSubmit)} className="space-y-6">
        {/* Header */}
        <div className="grid grid-cols-4 gap-4">
          <div>
            <label className="block text-sm font-medium">Voucher Type</label>
            <select {...register('type')} className="mt-1 block w-full rounded-md border-gray-300">
              <option value="Journal">Journal (F7)</option>
              <option value="Payment">Payment (F5)</option>
              <option value="Receipt">Receipt (F6)</option>
              <option value="Contra">Contra (F8)</option>
            </select>
          </div>
          <div>
            <label className="block text-sm font-medium">Date</label>
            <input type="date" {...register('date')} className="mt-1 block w-full rounded-md border-gray-300" />
          </div>
          <div>
            <label className="block text-sm font-medium">Reference No.</label>
            <input type="text" {...register('referenceNumber')} className="mt-1 block w-full rounded-md border-gray-300" />
          </div>
        </div>
        
        {/* Voucher Lines Table */}
        <div className="border rounded-lg overflow-hidden">
          <table className="min-w-full divide-y divide-gray-200">
            <thead className="bg-gray-50">
              <tr>
                <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Particulars</th>
                <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase w-32">Debit (₹)</th>
                <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase w-32">Credit (₹)</th>
                <th className="w-10"></th>
              </tr>
            </thead>
            <tbody className="bg-white divide-y divide-gray-200">
              {fields.map((field, index) => (
                <VoucherLineRow
                  key={field.id}
                  index={index}
                  control={control}
                  register={register}
                  setValue={setValue}
                  onRemove={() => remove(index)}
                  isActive={activeLineIndex === index}
                  onFocus={() => setActiveLineIndex(index)}
                />
              ))}
            </tbody>
            <tfoot className="bg-gray-50">
              <tr>
                <td className="px-4 py-3 font-medium">Total</td>
                <td className={`px-4 py-3 font-bold ${!isBalanced ? 'text-red-600' : 'text-green-600'}`}>
                  ₹{totalDebit.toLocaleString('en-IN', { minimumFractionDigits: 2 })}
                </td>
                <td className={`px-4 py-3 font-bold ${!isBalanced ? 'text-red-600' : 'text-green-600'}`}>
                  ₹{totalCredit.toLocaleString('en-IN', { minimumFractionDigits: 2 })}
                </td>
                <td></td>
              </tr>
              {!isBalanced && (
                <tr>
                  <td colSpan={4} className="px-4 py-2 text-red-600 text-sm">
                    ⚠️ Debit and Credit totals must match. Difference: ₹{Math.abs(totalDebit - totalCredit).toLocaleString('en-IN')}
                  </td>
                </tr>
              )}
            </tfoot>
          </table>
        </div>
        
        {/* Narration */}
        <div>
          <label className="block text-sm font-medium">Narration</label>
          <textarea {...register('narration')} rows={2} className="mt-1 block w-full rounded-md border-gray-300" />
        </div>
        
        {/* Actions */}
        <div className="flex justify-end gap-4">
          <button type="button" className="px-4 py-2 border rounded-md">Cancel (Esc)</button>
          <button 
            type="submit" 
            disabled={!isBalanced}
            className="px-4 py-2 bg-blue-600 text-white rounded-md disabled:opacity-50"
          >
            Save & Post (Ctrl+Enter)
          </button>
        </div>
      </form>
    </div>
  );
}

// Ledger Search Component with Keyboard Navigation
function LedgerSearch({ value, onChange, onSelect }: LedgerSearchProps) {
  const [isOpen, setIsOpen] = useState(false);
  const [search, setSearch] = useState(value);
  const [selectedIndex, setSelectedIndex] = useState(0);
  
  const { data: results } = useQuery({
    queryKey: ['ledger-search', search],
    queryFn: () => fetch(`/api/v1/accounting/ledgers/search?q=${search}&limit=10`).then(r => r.json()),
    enabled: search.length >= 2,
  });
  
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'ArrowDown') {
      e.preventDefault();
      setSelectedIndex(prev => Math.min(prev + 1, (results?.length || 1) - 1));
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      setSelectedIndex(prev => Math.max(prev - 1, 0));
    } else if (e.key === 'Enter' && results?.[selectedIndex]) {
      e.preventDefault();
      onSelect(results[selectedIndex]);
      setIsOpen(false);
    } else if (e.key === 'Escape') {
      setIsOpen(false);
    }
  };
  
  return (
    <div className="relative">
      <input
        type="text"
        value={search}
        onChange={(e) => { setSearch(e.target.value); setIsOpen(true); }}
        onKeyDown={handleKeyDown}
        onFocus={() => setIsOpen(true)}
        className="w-full px-3 py-2 border rounded-md"
        placeholder="Type ledger name..."
      />
      {isOpen && results?.length > 0 && (
        <ul className="absolute z-10 w-full mt-1 bg-white border rounded-md shadow-lg max-h-60 overflow-auto">
          {results.map((ledger: any, index: number) => (
            <li
              key={ledger.id}
              className={`px-3 py-2 cursor-pointer ${index === selectedIndex ? 'bg-blue-100' : 'hover:bg-gray-100'}`}
              onClick={() => { onSelect(ledger); setIsOpen(false); }}
            >
              <div className="font-medium">{ledger.name}</div>
              <div className="text-sm text-gray-500">{ledger.groupName}</div>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## **7. Domain Events (For Module Communication)**

```csharp
// Domain/Events/VoucherPostedEvent.cs
public record VoucherPostedEvent : DomainEvent
{
    public Guid VoucherId { get; init; }
    public Guid TenantId { get; init; }
    public VoucherType Type { get; init; }
    public DateOnly VoucherDate { get; init; }
    public string VoucherNumber { get; init; }
    public decimal TotalAmount { get; init; }
    public string SourceModule { get; init; }
    public List<VoucherLineEventData> Lines { get; init; }
}

public record VoucherLineEventData
{
    public Guid LedgerId { get; init; }
    public string LedgerName { get; init; }
    public decimal Amount { get; init; }
    public BalanceType Type { get; init; }
}

// Integration Events (Published to Message Bus for other modules)
public record LedgerBalanceChangedIntegrationEvent : IntegrationEvent
{
    public Guid TenantId { get; init; }
    public Guid LedgerId { get; init; }
    public string LedgerName { get; init; }
    public decimal NewBalance { get; init; }
    public BalanceType BalanceType { get; init; }
}

// GST Module subscribes to this
public record TaxableVoucherCreatedIntegrationEvent : IntegrationEvent
{
    public Guid TenantId { get; init; }
    public Guid VoucherId { get; init; }
    public VoucherType Type { get; init; }
    public DateOnly VoucherDate { get; init; }
    public GstTransactionData GstData { get; init; }
}

// Inventory Module subscribes to this
public record StockVoucherCreatedIntegrationEvent : IntegrationEvent
{
    public Guid TenantId { get; init; }
    public Guid VoucherId { get; init; }
    public VoucherType Type { get; init; }
    public List<StockLineData> StockLines { get; init; }
}
```

---

## **8. Testing Strategy**

### **8.1 Unit Tests**

```csharp
// Tests/Domain/VoucherTests.cs
public class VoucherTests
{
    [Fact]
    public void Voucher_WithBalancedEntries_ShouldValidateSuccessfully()
    {
        // Arrange
        var voucher = Voucher.Create(VoucherType.Journal, "JV001", DateOnly.FromDateTime(DateTime.Now), FinancialYearId.Create());
        voucher.AddLine(LedgerId.Create(), Money.INR(1000), BalanceType.Debit);
        voucher.AddLine(LedgerId.Create(), Money.INR(1000), BalanceType.Credit);
        
        // Act
        var result = voucher.Validate();
        
        // Assert
        result.IsSuccess.Should().BeTrue();
    }
    
    [Fact]
    public void Voucher_WithUnbalancedEntries_ShouldFail()
    {
        // Arrange
        var voucher = Voucher.Create(VoucherType.Journal, "JV001", DateOnly.FromDateTime(DateTime.Now), FinancialYearId.Create());
        voucher.AddLine(LedgerId.Create(), Money.INR(1000), BalanceType.Debit);
        voucher.AddLine(LedgerId.Create(), Money.INR(500), BalanceType.Credit);
        
        // Act
        var result = voucher.Validate();
        
        // Assert
        result.IsFailure.Should().BeTrue();
        result.Errors.Should().Contain(e => e.Contains("Debit") && e.Contains("Credit"));
    }
    
    [Fact]
    public void Voucher_Post_ShouldRaiseDomainEvent()
    {
        // Arrange
        var voucher = CreateValidVoucher();
        
        // Act
        voucher.Post();
        
        // Assert
        voucher.DomainEvents.Should().ContainSingle(e => e is VoucherPostedEvent);
    }
}

// Tests/Application/CreateVoucherCommandHandlerTests.cs
public class CreateVoucherCommandHandlerTests
{
    [Fact]
    public async Task Handle_WithLockedPeriod_ShouldReturnFailure()
    {
        // Arrange
        var mockFyRepo = new Mock<IFinancialYearRepository>();
        var fy = CreateFinancialYearWithLockedPeriod(DateOnly.Parse("2025-04-01"), DateOnly.Parse("2025-04-30"));
        mockFyRepo.Setup(r => r.GetActiveAsync(It.IsAny<CancellationToken>())).ReturnsAsync(fy);
        
        var handler = new CreateVoucherCommandHandler(/* ... */);
        var command = new CreateVoucherCommand { VoucherDate = DateOnly.Parse("2025-04-15"), /* ... */ };
        
        // Act
        var result = await handler.Handle(command, CancellationToken.None);
        
        // Assert
        result.IsFailure.Should().BeTrue();
        result.Errors.Should().Contain("Period is locked");
    }
}
```

---

## **9. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| Voucher Entry | Create voucher in <500ms with keyboard-only navigation |
| Double-Entry Validation | Prevent saving unbalanced vouchers |
| Ledger Statements | Generate statement for 10,000 transactions in <2s |
| Trial Balance | Calculate TB for 1,000 ledgers in <1s |
| Period Locking | Prevent any changes to locked periods |
| Audit Trail | All changes captured in immutable log |
| Multi-Currency | Support INR with 2 decimal precision |

---

## **10. Definition of Done**

- [ ] All CRUD operations for Ledgers, Groups, Vouchers
- [ ] Double-entry validation enforced at domain level
- [ ] Voucher number auto-generation by type
- [ ] Financial Year and Period management
- [ ] All standard reports (TB, P&L, BS, Day Book, Cash Book)
- [ ] Keyboard-first UI with Tally-like shortcuts
- [ ] API documentation complete
- [ ] Unit test coverage >85%
- [ ] Integration tests for critical paths
- [ ] Performance benchmarks met
- [ ] Security audit passed

---

## **11. Handoff to Other Modules**

The Core Accounting Engine exposes these **integration points** for other modules:

| Integration Point | Consumer Module | Purpose |
|-------------------|-----------------|---------|
| `CreateVoucherFromModule()` | All | Modules can create accounting entries |
| `VoucherPostedEvent` | GST, Inventory | React when voucher is finalized |
| `LedgerBalanceChangedEvent` | Banking, Dashboard | Update reconciliation, show live balance |
| `GetLedgerByType()` | GST, TDS | Find tax ledgers |
| `LockPeriod()` | GST | Lock period after filing |

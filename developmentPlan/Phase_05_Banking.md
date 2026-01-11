# **Phase 5: Banking Module**
**Duration:** 6 Weeks  
**Team:** Backend (2), Frontend (1), QA (1)  
**Dependencies:** Phase 1 (Core Accounting)  

---

## **1. Overview**

The Banking Module provides:
- Bank account management
- Bank statement import and reconciliation
- Payment processing (via payment gateways)
- Cash flow tracking
- Multi-bank support

---

## **2. Key Features**

### **2.1 Bank Account Management**
- Create and manage multiple bank accounts
- Link bank accounts to ledgers
- Track opening balances
- Support for different account types (Current, Savings, OD, CC)

### **2.2 Bank Reconciliation ("Tinder-Style")**
- Import bank statements (CSV, PDF, API)
- Auto-match transactions with vouchers
- Swipe-style UI for quick confirmation
- Create vouchers directly from unmatched bank entries

### **2.3 Payment Gateway Integration**
- Razorpay, PayU, CCAvenue integration
- Payment links generation
- Auto-reconciliation of online payments
- Refund processing

---

## **3. Domain Model**

```csharp
// Domain/Entities/BankAccount.cs
public class BankAccount : AggregateRoot<BankAccountId>
{
    public TenantId TenantId { get; private set; }
    public LedgerId LedgerId { get; private set; }  // Linked accounting ledger
    
    public string AccountName { get; private set; }
    public string AccountNumber { get; private set; }
    public string BankName { get; private set; }
    public string? BranchName { get; private set; }
    public string IfscCode { get; private set; }
    public BankAccountType AccountType { get; private set; }
    
    // Opening Balance
    public Money OpeningBalance { get; private set; }
    public DateOnly OpeningBalanceDate { get; private set; }
    
    // Current Balance (updated by reconciliation)
    public Money BookBalance { get; private set; }
    public Money BankBalance { get; private set; }
    public Money UnreconciledDifference => BankBalance.Subtract(BookBalance);
    
    // API Integration
    public bool IsApiEnabled { get; private set; }
    public string? AggregatorType { get; private set; }  // "Setu", "Razorpay"
    public string? AggregatorAccountId { get; private set; }
    public DateTime? LastSyncAt { get; private set; }
    
    public bool IsActive { get; private set; }
}

// Domain/Entities/BankStatement.cs
public class BankStatement : AggregateRoot<BankStatementId>
{
    public TenantId TenantId { get; private set; }
    public BankAccountId BankAccountId { get; private set; }
    
    public DateOnly StatementDate { get; private set; }
    public DateOnly FromDate { get; private set; }
    public DateOnly ToDate { get; private set; }
    
    public Money OpeningBalance { get; private set; }
    public Money ClosingBalance { get; private set; }
    
    public string ImportSource { get; private set; }  // "CSV", "API", "PDF"
    public DateTime ImportedAt { get; private set; }
    
    private readonly List<BankStatementEntry> _entries = new();
    public IReadOnlyCollection<BankStatementEntry> Entries => _entries.AsReadOnly();
    
    // Stats
    public int TotalEntries => _entries.Count;
    public int MatchedEntries => _entries.Count(e => e.Status == ReconciliationStatus.Matched);
    public int UnmatchedEntries => _entries.Count(e => e.Status == ReconciliationStatus.Unmatched);
}

// Domain/Entities/BankStatementEntry.cs
public class BankStatementEntry : Entity<BankStatementEntryId>
{
    public BankStatementId StatementId { get; private set; }
    
    public DateOnly TransactionDate { get; private set; }
    public DateOnly? ValueDate { get; private set; }
    public string Description { get; private set; }
    public string? Reference { get; private set; }
    public string? ChequeNumber { get; private set; }
    
    public Money? DebitAmount { get; private set; }
    public Money? CreditAmount { get; private set; }
    public Money Balance { get; private set; }
    
    // Reconciliation
    public ReconciliationStatus Status { get; private set; }
    public Guid? MatchedVoucherId { get; private set; }
    public string? MatchConfidence { get; private set; }  // "Exact", "Fuzzy", "Manual"
    public DateTime? ReconciledAt { get; private set; }
    public UserId? ReconciledBy { get; private set; }
    
    // For unmatched entries - create voucher
    public Guid? CreatedVoucherId { get; private set; }
    
    public void MarkAsMatched(Guid voucherId, string confidence)
    {
        Status = ReconciliationStatus.Matched;
        MatchedVoucherId = voucherId;
        MatchConfidence = confidence;
        ReconciledAt = DateTime.UtcNow;
    }
}

public enum ReconciliationStatus
{
    Unmatched = 1,
    SuggestedMatch = 2,
    Matched = 3,
    Ignored = 4,
    CreatedNew = 5
}
```

---

## **4. Bank Reconciliation Engine**

```csharp
// Domain/Services/BankReconciliationService.cs
public class BankReconciliationService
{
    public List<ReconciliationSuggestion> GenerateSuggestions(
        List<BankStatementEntry> bankEntries,
        List<Voucher> bookEntries)
    {
        var suggestions = new List<ReconciliationSuggestion>();
        
        foreach (var bankEntry in bankEntries.Where(e => e.Status == ReconciliationStatus.Unmatched))
        {
            var matches = FindMatches(bankEntry, bookEntries);
            
            if (matches.Any())
            {
                suggestions.Add(new ReconciliationSuggestion
                {
                    BankEntryId = bankEntry.Id,
                    Matches = matches.OrderByDescending(m => m.Score).ToList()
                });
            }
        }
        
        return suggestions;
    }
    
    private List<MatchResult> FindMatches(BankStatementEntry bankEntry, List<Voucher> vouchers)
    {
        var results = new List<MatchResult>();
        var amount = bankEntry.DebitAmount ?? bankEntry.CreditAmount;
        
        foreach (var voucher in vouchers)
        {
            var voucherAmount = voucher.TotalDebit;
            
            // Rule 1: Exact Amount + Date Match
            if (voucherAmount.Amount == amount.Amount && voucher.VoucherDate == bankEntry.TransactionDate)
            {
                results.Add(new MatchResult
                {
                    VoucherId = voucher.Id,
                    Score = 100,
                    MatchType = "ExactMatch"
                });
                continue;
            }
            
            // Rule 2: Exact Amount + Date within 3 days
            var dateDiff = Math.Abs((voucher.VoucherDate.ToDateTime(TimeOnly.MinValue) - 
                                     bankEntry.TransactionDate.ToDateTime(TimeOnly.MinValue)).Days);
            if (voucherAmount.Amount == amount.Amount && dateDiff <= 3)
            {
                results.Add(new MatchResult
                {
                    VoucherId = voucher.Id,
                    Score = 90 - (dateDiff * 5),
                    MatchType = "FuzzyDate"
                });
                continue;
            }
            
            // Rule 3: Reference/Cheque number match
            if (!string.IsNullOrEmpty(bankEntry.ChequeNumber) && 
                voucher.ReferenceNumber?.Contains(bankEntry.ChequeNumber) == true)
            {
                results.Add(new MatchResult
                {
                    VoucherId = voucher.Id,
                    Score = 85,
                    MatchType = "ReferenceMatch"
                });
            }
            
            // Rule 4: Description contains party name
            // ... ML-based matching can be added here
        }
        
        return results;
    }
}

public record ReconciliationSuggestion
{
    public BankStatementEntryId BankEntryId { get; init; }
    public List<MatchResult> Matches { get; init; }
}

public record MatchResult
{
    public VoucherId VoucherId { get; init; }
    public int Score { get; init; }  // 0-100
    public string MatchType { get; init; }
}
```

---

## **5. API Endpoints**

```csharp
// API/Controllers/BankingController.cs
[ApiController]
[Route("api/v1/banking")]
[Authorize]
[RequireModule(ModuleType.Banking)]
public class BankingController : ControllerBase
{
    // Bank Accounts
    [HttpGet("accounts")]
    public async Task<ActionResult<List<BankAccountDto>>> GetAccounts()
    {
        return Ok(await _mediator.Send(new GetBankAccountsQuery()));
    }
    
    [HttpPost("accounts")]
    public async Task<ActionResult<BankAccountId>> CreateAccount([FromBody] CreateBankAccountCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    // Statement Import
    [HttpPost("accounts/{accountId}/import-statement")]
    public async Task<ActionResult<BankStatementId>> ImportStatement(
        Guid accountId,
        [FromForm] IFormFile file)
    {
        var command = new ImportBankStatementCommand
        {
            BankAccountId = accountId,
            File = file
        };
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("accounts/{accountId}/sync")]
    public async Task<ActionResult> SyncFromBank(Guid accountId)
    {
        // Fetch from bank aggregator API
        return Ok(await _mediator.Send(new SyncBankAccountCommand { AccountId = accountId }));
    }
    
    // Reconciliation
    [HttpGet("statements/{statementId}/entries")]
    public async Task<ActionResult<List<BankStatementEntryDto>>> GetStatementEntries(Guid statementId)
    {
        return Ok(await _mediator.Send(new GetStatementEntriesQuery { StatementId = statementId }));
    }
    
    [HttpGet("statements/{statementId}/suggestions")]
    public async Task<ActionResult<List<ReconciliationSuggestionDto>>> GetSuggestions(Guid statementId)
    {
        return Ok(await _mediator.Send(new GetReconciliationSuggestionsQuery { StatementId = statementId }));
    }
    
    [HttpPost("entries/{entryId}/match")]
    public async Task<ActionResult> MatchEntry(Guid entryId, [FromBody] MatchEntryRequest request)
    {
        var command = new MatchBankEntryCommand
        {
            BankEntryId = entryId,
            VoucherId = request.VoucherId
        };
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("entries/{entryId}/create-voucher")]
    public async Task<ActionResult<VoucherId>> CreateVoucherFromEntry(
        Guid entryId, 
        [FromBody] CreateVoucherFromBankEntryRequest request)
    {
        var command = new CreateVoucherFromBankEntryCommand
        {
            BankEntryId = entryId,
            CounterpartyLedgerId = request.LedgerId,
            VoucherType = request.VoucherType,
            Narration = request.Narration
        };
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("entries/{entryId}/ignore")]
    public async Task<ActionResult> IgnoreEntry(Guid entryId)
    {
        return Ok(await _mediator.Send(new IgnoreBankEntryCommand { EntryId = entryId }));
    }
    
    // Reports
    [HttpGet("accounts/{accountId}/reconciliation-report")]
    public async Task<ActionResult<BankReconciliationReportDto>> GetReconciliationReport(
        Guid accountId,
        [FromQuery] DateOnly asOnDate)
    {
        return Ok(await _mediator.Send(new GetBankReconciliationReportQuery 
        { 
            AccountId = accountId, 
            AsOnDate = asOnDate 
        }));
    }
    
    [HttpGet("cash-flow")]
    public async Task<ActionResult<CashFlowDto>> GetCashFlow(
        [FromQuery] DateOnly fromDate,
        [FromQuery] DateOnly toDate)
    {
        return Ok(await _mediator.Send(new GetCashFlowQuery { FromDate = fromDate, ToDate = toDate }));
    }
}
```

---

## **6. Frontend - Tinder-Style Reconciliation**

```typescript
// src/app/(dashboard)/banking/reconciliation/page.tsx
'use client';

import { useState } from 'react';
import { useQuery, useMutation } from '@tanstack/react-query';
import { motion, AnimatePresence } from 'framer-motion';
import { CheckIcon, XIcon, PlusIcon } from 'lucide-react';

export default function BankReconciliationPage() {
  const [currentIndex, setCurrentIndex] = useState(0);
  
  const { data: entries } = useQuery({
    queryKey: ['bank-entries-unmatched'],
    queryFn: () => fetch('/api/v1/banking/statements/current/entries?status=unmatched').then(r => r.json())
  });
  
  const matchMutation = useMutation({
    mutationFn: ({ entryId, voucherId }: { entryId: string; voucherId: string }) =>
      fetch(`/api/v1/banking/entries/${entryId}/match`, {
        method: 'POST',
        body: JSON.stringify({ voucherId })
      }),
    onSuccess: () => setCurrentIndex(prev => prev + 1)
  });
  
  const ignoreMutation = useMutation({
    mutationFn: (entryId: string) =>
      fetch(`/api/v1/banking/entries/${entryId}/ignore`, { method: 'POST' }),
    onSuccess: () => setCurrentIndex(prev => prev + 1)
  });
  
  const currentEntry = entries?.[currentIndex];
  const suggestion = currentEntry?.suggestions?.[0];
  
  if (!currentEntry) {
    return (
      <div className="flex items-center justify-center h-96">
        <div className="text-center">
          <CheckIcon className="w-16 h-16 text-green-500 mx-auto mb-4" />
          <h2 className="text-2xl font-bold">All Done!</h2>
          <p className="text-gray-500">All bank entries have been reconciled.</p>
        </div>
      </div>
    );
  }
  
  return (
    <div className="max-w-2xl mx-auto p-6">
      <div className="mb-8">
        <h1 className="text-2xl font-bold">Bank Reconciliation</h1>
        <p className="text-gray-500">{entries.length - currentIndex} entries remaining</p>
      </div>
      
      <AnimatePresence mode="wait">
        <motion.div
          key={currentEntry.id}
          initial={{ opacity: 0, x: 100 }}
          animate={{ opacity: 1, x: 0 }}
          exit={{ opacity: 0, x: -100 }}
          className="bg-white rounded-xl shadow-lg p-6"
        >
          {/* Bank Entry Card */}
          <div className="border-b pb-4 mb-4">
            <div className="flex justify-between items-start">
              <div>
                <p className="text-sm text-gray-500">{currentEntry.transactionDate}</p>
                <p className="font-medium">{currentEntry.description}</p>
                {currentEntry.reference && (
                  <p className="text-sm text-gray-500">Ref: {currentEntry.reference}</p>
                )}
              </div>
              <div className="text-right">
                {currentEntry.debitAmount && (
                  <p className="text-red-600 font-bold">
                    -₹{currentEntry.debitAmount.toLocaleString('en-IN')}
                  </p>
                )}
                {currentEntry.creditAmount && (
                  <p className="text-green-600 font-bold">
                    +₹{currentEntry.creditAmount.toLocaleString('en-IN')}
                  </p>
                )}
              </div>
            </div>
          </div>
          
          {/* Suggested Match */}
          {suggestion && (
            <div className="bg-green-50 border border-green-200 rounded-lg p-4 mb-4">
              <div className="flex items-center justify-between mb-2">
                <span className="text-sm font-medium text-green-800">
                  Suggested Match ({suggestion.score}% confidence)
                </span>
                <span className="text-xs bg-green-200 text-green-800 px-2 py-1 rounded">
                  {suggestion.matchType}
                </span>
              </div>
              <div className="flex justify-between">
                <div>
                  <p className="font-medium">{suggestion.voucher.voucherNumber}</p>
                  <p className="text-sm text-gray-600">{suggestion.voucher.narration}</p>
                </div>
                <p className="font-bold">₹{suggestion.voucher.amount.toLocaleString('en-IN')}</p>
              </div>
            </div>
          )}
          
          {/* Actions */}
          <div className="flex justify-center gap-4 mt-6">
            <button
              onClick={() => ignoreMutation.mutate(currentEntry.id)}
              className="flex items-center gap-2 px-6 py-3 border rounded-full hover:bg-gray-50"
            >
              <XIcon className="w-5 h-5 text-red-500" />
              Skip
            </button>
            
            <button
              onClick={() => {/* Open create voucher modal */}}
              className="flex items-center gap-2 px-6 py-3 border rounded-full hover:bg-gray-50"
            >
              <PlusIcon className="w-5 h-5 text-blue-500" />
              Create New
            </button>
            
            {suggestion && (
              <button
                onClick={() => matchMutation.mutate({ 
                  entryId: currentEntry.id, 
                  voucherId: suggestion.voucher.id 
                })}
                className="flex items-center gap-2 px-6 py-3 bg-green-600 text-white rounded-full hover:bg-green-700"
              >
                <CheckIcon className="w-5 h-5" />
                Match
              </button>
            )}
          </div>
        </motion.div>
      </AnimatePresence>
      
      {/* Progress Bar */}
      <div className="mt-8">
        <div className="h-2 bg-gray-200 rounded-full">
          <div 
            className="h-2 bg-green-500 rounded-full transition-all"
            style={{ width: `${(currentIndex / entries.length) * 100}%` }}
          />
        </div>
      </div>
    </div>
  );
}
```

---

## **7. Database Schema**

```sql
-- Banking Module Tables

CREATE TABLE bank_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    ledger_id UUID NOT NULL REFERENCES ledgers(id),
    
    account_name VARCHAR(100) NOT NULL,
    account_number VARCHAR(30) NOT NULL,
    bank_name VARCHAR(100) NOT NULL,
    branch_name VARCHAR(100),
    ifsc_code VARCHAR(11) NOT NULL,
    account_type VARCHAR(20) NOT NULL,
    
    opening_balance DECIMAL(18,2) DEFAULT 0,
    opening_balance_date DATE,
    
    book_balance DECIMAL(18,2) DEFAULT 0,
    bank_balance DECIMAL(18,2) DEFAULT 0,
    
    is_api_enabled BOOLEAN DEFAULT FALSE,
    aggregator_type VARCHAR(30),
    aggregator_account_id VARCHAR(100),
    last_sync_at TIMESTAMPTZ,
    
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_bank_account UNIQUE (tenant_id, account_number)
);

CREATE TABLE bank_statements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    bank_account_id UUID NOT NULL REFERENCES bank_accounts(id),
    
    statement_date DATE NOT NULL,
    from_date DATE NOT NULL,
    to_date DATE NOT NULL,
    
    opening_balance DECIMAL(18,2) NOT NULL,
    closing_balance DECIMAL(18,2) NOT NULL,
    
    import_source VARCHAR(20) NOT NULL,
    imported_at TIMESTAMPTZ DEFAULT NOW(),
    
    total_entries INT DEFAULT 0,
    matched_entries INT DEFAULT 0,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE bank_statement_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id UUID NOT NULL REFERENCES bank_statements(id) ON DELETE CASCADE,
    
    transaction_date DATE NOT NULL,
    value_date DATE,
    description TEXT NOT NULL,
    reference VARCHAR(100),
    cheque_number VARCHAR(20),
    
    debit_amount DECIMAL(18,2),
    credit_amount DECIMAL(18,2),
    balance DECIMAL(18,2) NOT NULL,
    
    status VARCHAR(20) DEFAULT 'Unmatched',
    matched_voucher_id UUID REFERENCES vouchers(id),
    match_confidence VARCHAR(20),
    reconciled_at TIMESTAMPTZ,
    reconciled_by UUID REFERENCES users(id),
    
    created_voucher_id UUID REFERENCES vouchers(id),
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_bank_entries_status ON bank_statement_entries(statement_id, status);
CREATE INDEX idx_bank_entries_date ON bank_statement_entries(transaction_date);
```

---

## **8. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| CSV Import | Parse and import 1000 entries in < 5s |
| Auto-Match | Match rate > 70% on typical data |
| Manual Match | Match in single click |
| Create Voucher | Create Payment/Receipt from bank entry |
| Reconciliation Report | Generate BRS in < 2s |
| API Sync | Fetch last 30 days in < 10s |

---

## **9. Definition of Done**

- [ ] Bank account CRUD with ledger linking
- [ ] CSV/Excel statement import
- [ ] PDF statement parsing (basic)
- [ ] Auto-matching algorithm
- [ ] Tinder-style reconciliation UI
- [ ] Create voucher from bank entry
- [ ] Bank reconciliation statement report
- [ ] Cash flow report
- [ ] API integration with one aggregator (Setu/Razorpay)
- [ ] Unit tests for matching algorithm

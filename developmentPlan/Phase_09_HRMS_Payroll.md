# **Phase 9: HRMS & Payroll Module**
**Duration:** 10 Weeks  
**Team:** Backend (2), Frontend (2), QA (1)  
**Dependencies:** Phase 1 (Core Accounting)  

---

## **1. Overview**

The HRMS & Payroll Module provides:
- Employee master management
- Attendance tracking
- Leave management
- Indian payroll with statutory compliance
- PF, ESI, PT, TDS calculations
- Form 16 and other statutory reports

---

## **2. Indian Payroll Compliance**

| Component | Applicability | Rate |
|-----------|---------------|------|
| EPF (Employee) | Basic + DA ≤ ₹15,000 | 12% |
| EPF (Employer) | Basic + DA ≤ ₹15,000 | 12% (3.67% EPF + 8.33% EPS) |
| ESI (Employee) | Gross ≤ ₹21,000 | 0.75% |
| ESI (Employer) | Gross ≤ ₹21,000 | 3.25% |
| Professional Tax | State-wise | Varies (max ₹200/month) |
| TDS | As per income tax slabs | Variable |
| LWF | State-wise | Variable |

---

## **3. Domain Model**

```csharp
// Domain/Entities/Employee.cs
public class Employee : AggregateRoot<EmployeeId>
{
    public TenantId TenantId { get; private set; }
    
    // Personal Details
    public string EmployeeCode { get; private set; }
    public string FirstName { get; private set; }
    public string LastName { get; private set; }
    public string FullName => $"{FirstName} {LastName}";
    public DateOnly DateOfBirth { get; private set; }
    public Gender Gender { get; private set; }
    public MaritalStatus MaritalStatus { get; private set; }
    
    // Contact
    public string Email { get; private set; }
    public string Phone { get; private set; }
    public Address PermanentAddress { get; private set; }
    public Address? CurrentAddress { get; private set; }
    
    // Employment
    public DateOnly DateOfJoining { get; private set; }
    public DateOnly? DateOfLeaving { get; private set; }
    public EmploymentType EmploymentType { get; private set; }
    public DepartmentId DepartmentId { get; private set; }
    public DesignationId DesignationId { get; private set; }
    public EmployeeId? ReportingManagerId { get; private set; }
    
    // Statutory
    public string PanNumber { get; private set; }
    public string? AadhaarNumber { get; private set; }
    public string? UanNumber { get; private set; }  // Universal Account Number (EPF)
    public string? EsiNumber { get; private set; }
    
    // Bank
    public string BankAccountNumber { get; private set; }
    public string BankIfscCode { get; private set; }
    public string BankName { get; private set; }
    
    // Salary Structure
    public SalaryStructure CurrentSalaryStructure { get; private set; }
    
    // Status
    public EmployeeStatus Status { get; private set; }
    public bool IsActive => Status == EmployeeStatus.Active;
    
    public void ApplySalaryRevision(SalaryStructure newStructure, DateOnly effectiveFrom)
    {
        if (effectiveFrom <= CurrentSalaryStructure?.EffectiveFrom)
            throw new DomainException("New structure must be effective after current structure");
        
        // Archive current structure
        AddDomainEvent(new SalaryRevisedEvent(Id, CurrentSalaryStructure, newStructure, effectiveFrom));
        
        CurrentSalaryStructure = newStructure;
    }
}

// Domain/Entities/SalaryStructure.cs
public class SalaryStructure : Entity<SalaryStructureId>
{
    public EmployeeId EmployeeId { get; private set; }
    public DateOnly EffectiveFrom { get; private set; }
    public DateOnly? EffectiveTo { get; private set; }
    
    // Earnings
    public Money BasicSalary { get; private set; }
    public Money HouseRentAllowance { get; private set; }
    public Money ConveyanceAllowance { get; private set; }
    public Money MedicalAllowance { get; private set; }
    public Money SpecialAllowance { get; private set; }
    public Money OtherAllowances { get; private set; }
    
    public Money GrossSalary => BasicSalary
        .Add(HouseRentAllowance)
        .Add(ConveyanceAllowance)
        .Add(MedicalAllowance)
        .Add(SpecialAllowance)
        .Add(OtherAllowances);
    
    // Statutory Applicability
    public bool IsEpfApplicable { get; private set; }
    public bool IsEsiApplicable { get; private set; }
    public bool IsPtApplicable { get; private set; }
    
    // For EPF calculation
    public Money EpfBasic => BasicSalary;  // EPF on Basic only (common practice)
    
    public decimal CalculateEmployeeEpf()
    {
        if (!IsEpfApplicable) return 0;
        return Math.Round(EpfBasic.Amount * 0.12m, 0);  // 12% rounded
    }
    
    public decimal CalculateEmployerEpf()
    {
        if (!IsEpfApplicable) return 0;
        var epfAmount = Math.Round(EpfBasic.Amount * 0.0367m, 0);  // 3.67% to EPF
        var epsAmount = Math.Min(Math.Round(EpfBasic.Amount * 0.0833m, 0), 1250);  // 8.33% to EPS, max ₹1250
        return epfAmount + epsAmount;
    }
}

// Domain/Entities/Payroll.cs
public class Payroll : AggregateRoot<PayrollId>
{
    public TenantId TenantId { get; private set; }
    public int Year { get; private set; }
    public int Month { get; private set; }
    public string Period => $"{Year}-{Month:D2}";
    
    public PayrollStatus Status { get; private set; }
    public DateOnly ProcessedDate { get; private set; }
    
    private readonly List<PaySlip> _paySlips = new();
    public IReadOnlyCollection<PaySlip> PaySlips => _paySlips.AsReadOnly();
    
    public Money TotalGross => new Money(_paySlips.Sum(p => p.GrossSalary.Amount));
    public Money TotalDeductions => new Money(_paySlips.Sum(p => p.TotalDeductions.Amount));
    public Money TotalNet => new Money(_paySlips.Sum(p => p.NetSalary.Amount));
    
    public void Process()
    {
        if (Status != PayrollStatus.Draft)
            throw new DomainException("Can only process draft payroll");
        
        Status = PayrollStatus.Processed;
        ProcessedDate = DateOnly.FromDateTime(DateTime.Today);
        
        AddDomainEvent(new PayrollProcessedEvent(Id, _paySlips.Count, TotalNet));
    }
    
    public void GenerateVoucher()
    {
        if (Status != PayrollStatus.Processed)
            throw new DomainException("Payroll must be processed first");
        
        // Creates Journal voucher:
        // Dr. Salary Expense (P&L)
        // Cr. Salary Payable (Liability) - Net amount
        // Cr. EPF Payable (Liability)
        // Cr. ESI Payable (Liability)
        // Cr. PT Payable (Liability)
        // Cr. TDS Payable (Liability)
        
        Status = PayrollStatus.VoucherGenerated;
        AddDomainEvent(new PayrollVoucherGeneratedEvent(Id));
    }
}

// Domain/Entities/PaySlip.cs
public class PaySlip : Entity<PaySlipId>
{
    public PayrollId PayrollId { get; private set; }
    public EmployeeId EmployeeId { get; private set; }
    
    public int PaidDays { get; private set; }
    public int LopDays { get; private set; }
    public int TotalDays { get; private set; }
    
    // Earnings
    public Money BasicSalary { get; private set; }
    public Money HRA { get; private set; }
    public Money Conveyance { get; private set; }
    public Money Medical { get; private set; }
    public Money Special { get; private set; }
    public Money OtherEarnings { get; private set; }
    public Money GrossSalary { get; private set; }
    
    // Deductions
    public Money EmployeeEpf { get; private set; }
    public Money EmployeeEsi { get; private set; }
    public Money ProfessionalTax { get; private set; }
    public Money Tds { get; private set; }
    public Money OtherDeductions { get; private set; }
    public Money TotalDeductions { get; private set; }
    
    // Employer Contributions (not deducted from salary)
    public Money EmployerEpf { get; private set; }
    public Money EmployerEsi { get; private set; }
    
    // Net
    public Money NetSalary { get; private set; }
    
    // CTC Calculation
    public Money MonthlyCTC => GrossSalary.Add(EmployerEpf).Add(EmployerEsi);
}
```

---

## **4. Payroll Calculation Engine**

```csharp
// Domain/Services/PayrollCalculationService.cs
public class PayrollCalculationService
{
    private readonly IPtSlabRepository _ptSlabRepository;
    private readonly ITaxSlabRepository _taxSlabRepository;
    
    public PaySlip CalculatePaySlip(
        Employee employee, 
        SalaryStructure structure, 
        AttendanceSummary attendance,
        TaxDeclaration? taxDeclaration,
        int year, 
        int month)
    {
        var totalDays = DateTime.DaysInMonth(year, month);
        var paidDays = attendance.PresentDays + attendance.PaidLeaveDays + attendance.Holidays;
        var lopDays = attendance.LopDays;
        
        // Pro-rate salary for LOP
        var ratio = (decimal)paidDays / totalDays;
        
        var basic = ProRate(structure.BasicSalary, ratio);
        var hra = ProRate(structure.HouseRentAllowance, ratio);
        var conveyance = ProRate(structure.ConveyanceAllowance, ratio);
        var medical = ProRate(structure.MedicalAllowance, ratio);
        var special = ProRate(structure.SpecialAllowance, ratio);
        var otherEarnings = ProRate(structure.OtherAllowances, ratio);
        
        var gross = basic.Add(hra).Add(conveyance).Add(medical).Add(special).Add(otherEarnings);
        
        // Calculate deductions
        var employeeEpf = CalculateEmployeeEpf(structure, basic);
        var employeeEsi = CalculateEmployeeEsi(structure, gross);
        var pt = CalculateProfessionalTax(employee.CurrentAddress?.State, gross);
        var tds = CalculateTds(employee, structure, taxDeclaration, year, month);
        
        var totalDeductions = employeeEpf.Add(employeeEsi).Add(pt).Add(tds);
        var netSalary = gross.Subtract(totalDeductions);
        
        // Employer contributions
        var employerEpf = CalculateEmployerEpf(structure, basic);
        var employerEsi = CalculateEmployerEsi(structure, gross);
        
        return new PaySlip
        {
            PaidDays = paidDays,
            LopDays = lopDays,
            TotalDays = totalDays,
            BasicSalary = basic,
            HRA = hra,
            Conveyance = conveyance,
            Medical = medical,
            Special = special,
            OtherEarnings = otherEarnings,
            GrossSalary = gross,
            EmployeeEpf = employeeEpf,
            EmployeeEsi = employeeEsi,
            ProfessionalTax = pt,
            Tds = tds,
            TotalDeductions = totalDeductions,
            NetSalary = netSalary,
            EmployerEpf = employerEpf,
            EmployerEsi = employerEsi
        };
    }
    
    private Money CalculateEmployeeEpf(SalaryStructure structure, Money basic)
    {
        if (!structure.IsEpfApplicable) return Money.Zero;
        
        // EPF is 12% of Basic (capped at ₹15,000 for mandatory contribution)
        var epfBase = Math.Min(basic.Amount, 15000);
        return new Money(Math.Round(epfBase * 0.12m, 0));
    }
    
    private Money CalculateEmployerEpf(SalaryStructure structure, Money basic)
    {
        if (!structure.IsEpfApplicable) return Money.Zero;
        
        var epfBase = Math.Min(basic.Amount, 15000);
        
        // 3.67% to EPF Account
        var epf = Math.Round(epfBase * 0.0367m, 0);
        
        // 8.33% to EPS (capped at ₹1250)
        var eps = Math.Min(Math.Round(epfBase * 0.0833m, 0), 1250);
        
        return new Money(epf + eps);
    }
    
    private Money CalculateEmployeeEsi(SalaryStructure structure, Money gross)
    {
        if (!structure.IsEsiApplicable) return Money.Zero;
        if (gross.Amount > 21000) return Money.Zero;  // ESI not applicable above ₹21,000
        
        return new Money(Math.Round(gross.Amount * 0.0075m, 0));  // 0.75%
    }
    
    private Money CalculateEmployerEsi(SalaryStructure structure, Money gross)
    {
        if (!structure.IsEsiApplicable) return Money.Zero;
        if (gross.Amount > 21000) return Money.Zero;
        
        return new Money(Math.Round(gross.Amount * 0.0325m, 0));  // 3.25%
    }
    
    private Money CalculateProfessionalTax(string? state, Money gross)
    {
        if (string.IsNullOrEmpty(state)) return Money.Zero;
        
        var slabs = _ptSlabRepository.GetSlabs(state);
        var applicableSlab = slabs.FirstOrDefault(s => 
            gross.Amount >= s.MinSalary && gross.Amount <= s.MaxSalary);
        
        return new Money(applicableSlab?.TaxAmount ?? 0);
    }
    
    private Money CalculateTds(
        Employee employee, 
        SalaryStructure structure, 
        TaxDeclaration? declaration,
        int year, 
        int month)
    {
        // Calculate annual projected income
        var monthsRemaining = 12 - month + 1;  // Including current month
        var annualGross = structure.GrossSalary.Amount * 12;
        
        // Standard deduction
        var standardDeduction = 50000m;
        
        // Exemptions from declaration
        var exemptions = declaration?.GetTotalExemptions() ?? 0;
        
        // Taxable income
        var taxableIncome = annualGross - standardDeduction - exemptions;
        
        // Calculate tax based on regime
        var regime = declaration?.TaxRegime ?? TaxRegime.New;
        var annualTax = CalculateIncomeTax(taxableIncome, regime);
        
        // Add cess
        var totalTax = annualTax * 1.04m;  // 4% Health & Education Cess
        
        // Monthly TDS
        var monthlyTds = totalTax / monthsRemaining;
        
        return new Money(Math.Round(monthlyTds, 0));
    }
    
    private decimal CalculateIncomeTax(decimal taxableIncome, TaxRegime regime)
    {
        if (regime == TaxRegime.New)
        {
            // New Tax Regime (FY 2024-25)
            if (taxableIncome <= 300000) return 0;
            if (taxableIncome <= 700000) return (taxableIncome - 300000) * 0.05m;
            if (taxableIncome <= 1000000) return 20000 + (taxableIncome - 700000) * 0.10m;
            if (taxableIncome <= 1200000) return 50000 + (taxableIncome - 1000000) * 0.15m;
            if (taxableIncome <= 1500000) return 80000 + (taxableIncome - 1200000) * 0.20m;
            return 140000 + (taxableIncome - 1500000) * 0.30m;
        }
        else
        {
            // Old Tax Regime
            if (taxableIncome <= 250000) return 0;
            if (taxableIncome <= 500000) return (taxableIncome - 250000) * 0.05m;
            if (taxableIncome <= 1000000) return 12500 + (taxableIncome - 500000) * 0.20m;
            return 112500 + (taxableIncome - 1000000) * 0.30m;
        }
    }
}
```

---

## **5. API Endpoints**

```csharp
// API/Controllers/HrmsController.cs
[ApiController]
[Route("api/v1/hrms")]
[Authorize]
[RequireModule(ModuleType.HRMS)]
public class HrmsController : ControllerBase
{
    // Employees
    [HttpGet("employees")]
    public async Task<ActionResult<PagedList<EmployeeDto>>> GetEmployees([FromQuery] EmployeeFilter filter)
    {
        return Ok(await _mediator.Send(new GetEmployeesQuery { Filter = filter }));
    }
    
    [HttpGet("employees/{id}")]
    public async Task<ActionResult<EmployeeDetailDto>> GetEmployee(Guid id)
    {
        return Ok(await _mediator.Send(new GetEmployeeQuery { Id = id }));
    }
    
    [HttpPost("employees")]
    public async Task<ActionResult<EmployeeId>> CreateEmployee([FromBody] CreateEmployeeCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPut("employees/{id}/salary-structure")]
    public async Task<ActionResult> UpdateSalaryStructure(Guid id, [FromBody] UpdateSalaryStructureCommand command)
    {
        command.EmployeeId = id;
        return Ok(await _mediator.Send(command));
    }
    
    // Attendance
    [HttpGet("attendance/{employeeId}/{year}/{month}")]
    public async Task<ActionResult<AttendanceDto>> GetAttendance(Guid employeeId, int year, int month)
    {
        return Ok(await _mediator.Send(new GetAttendanceQuery 
        { 
            EmployeeId = employeeId, 
            Year = year, 
            Month = month 
        }));
    }
    
    [HttpPost("attendance/mark")]
    public async Task<ActionResult> MarkAttendance([FromBody] MarkAttendanceCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("attendance/import")]
    public async Task<ActionResult> ImportAttendance([FromForm] IFormFile file)
    {
        return Ok(await _mediator.Send(new ImportAttendanceCommand { File = file }));
    }
    
    // Leave
    [HttpGet("leave/balance/{employeeId}")]
    public async Task<ActionResult<LeaveBalanceDto>> GetLeaveBalance(Guid employeeId)
    {
        return Ok(await _mediator.Send(new GetLeaveBalanceQuery { EmployeeId = employeeId }));
    }
    
    [HttpPost("leave/apply")]
    public async Task<ActionResult<LeaveRequestId>> ApplyLeave([FromBody] ApplyLeaveCommand command)
    {
        return Ok(await _mediator.Send(command));
    }
    
    [HttpPost("leave/{requestId}/approve")]
    public async Task<ActionResult> ApproveLeave(Guid requestId)
    {
        return Ok(await _mediator.Send(new ApproveLeaveCommand { RequestId = requestId }));
    }
    
    // Payroll
    [HttpGet("payroll/{year}/{month}")]
    public async Task<ActionResult<PayrollDto>> GetPayroll(int year, int month)
    {
        return Ok(await _mediator.Send(new GetPayrollQuery { Year = year, Month = month }));
    }
    
    [HttpPost("payroll/{year}/{month}/process")]
    public async Task<ActionResult> ProcessPayroll(int year, int month)
    {
        return Ok(await _mediator.Send(new ProcessPayrollCommand { Year = year, Month = month }));
    }
    
    [HttpPost("payroll/{payrollId}/generate-voucher")]
    public async Task<ActionResult<VoucherId>> GeneratePayrollVoucher(Guid payrollId)
    {
        return Ok(await _mediator.Send(new GeneratePayrollVoucherCommand { PayrollId = payrollId }));
    }
    
    [HttpGet("payroll/payslip/{employeeId}/{year}/{month}")]
    public async Task<ActionResult<PaySlipDto>> GetPaySlip(Guid employeeId, int year, int month)
    {
        return Ok(await _mediator.Send(new GetPaySlipQuery 
        { 
            EmployeeId = employeeId, 
            Year = year, 
            Month = month 
        }));
    }
    
    [HttpGet("payroll/payslip/{employeeId}/{year}/{month}/pdf")]
    public async Task<IActionResult> DownloadPaySlip(Guid employeeId, int year, int month)
    {
        var result = await _mediator.Send(new GeneratePaySlipPdfQuery 
        { 
            EmployeeId = employeeId, 
            Year = year, 
            Month = month 
        });
        return File(result.FileBytes, "application/pdf", result.FileName);
    }
    
    // Statutory Reports
    [HttpGet("reports/form-16/{employeeId}/{financialYear}")]
    public async Task<ActionResult<Form16Dto>> GetForm16(Guid employeeId, string financialYear)
    {
        return Ok(await _mediator.Send(new GetForm16Query 
        { 
            EmployeeId = employeeId, 
            FinancialYear = financialYear 
        }));
    }
    
    [HttpGet("reports/pf-ecr/{year}/{month}")]
    public async Task<ActionResult<byte[]>> GetPfEcrFile(int year, int month)
    {
        return Ok(await _mediator.Send(new GeneratePfEcrCommand { Year = year, Month = month }));
    }
    
    [HttpGet("reports/esi-return/{year}/{month}")]
    public async Task<ActionResult<EsiReturnDto>> GetEsiReturn(int year, int month)
    {
        return Ok(await _mediator.Send(new GenerateEsiReturnQuery { Year = year, Month = month }));
    }
}
```

---

## **6. Database Schema**

```sql
-- HRMS Module Tables

CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20) NOT NULL,
    parent_id UUID REFERENCES departments(id),
    is_active BOOLEAN DEFAULT TRUE,
    
    CONSTRAINT unique_department UNIQUE (tenant_id, code)
);

CREATE TABLE designations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20) NOT NULL,
    level INT,
    is_active BOOLEAN DEFAULT TRUE,
    
    CONSTRAINT unique_designation UNIQUE (tenant_id, code)
);

CREATE TABLE employees (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    
    employee_code VARCHAR(20) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    date_of_birth DATE NOT NULL,
    gender VARCHAR(10),
    marital_status VARCHAR(20),
    
    email VARCHAR(200),
    phone VARCHAR(20),
    
    -- Address (JSON for simplicity)
    permanent_address JSONB,
    current_address JSONB,
    
    -- Employment
    date_of_joining DATE NOT NULL,
    date_of_leaving DATE,
    employment_type VARCHAR(20),
    department_id UUID REFERENCES departments(id),
    designation_id UUID REFERENCES designations(id),
    reporting_manager_id UUID REFERENCES employees(id),
    
    -- Statutory
    pan_number VARCHAR(10),
    aadhaar_number VARCHAR(12),
    uan_number VARCHAR(12),
    esi_number VARCHAR(17),
    
    -- Bank
    bank_account_number VARCHAR(20),
    bank_ifsc_code VARCHAR(11),
    bank_name VARCHAR(100),
    
    status VARCHAR(20) DEFAULT 'Active',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_employee UNIQUE (tenant_id, employee_code)
);

CREATE TABLE salary_structures (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id UUID NOT NULL REFERENCES employees(id),
    effective_from DATE NOT NULL,
    effective_to DATE,
    
    basic_salary DECIMAL(18,2) NOT NULL,
    hra DECIMAL(18,2) DEFAULT 0,
    conveyance DECIMAL(18,2) DEFAULT 0,
    medical DECIMAL(18,2) DEFAULT 0,
    special_allowance DECIMAL(18,2) DEFAULT 0,
    other_allowances DECIMAL(18,2) DEFAULT 0,
    
    is_epf_applicable BOOLEAN DEFAULT TRUE,
    is_esi_applicable BOOLEAN DEFAULT TRUE,
    is_pt_applicable BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE payrolls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    year INT NOT NULL,
    month INT NOT NULL,
    status VARCHAR(20) DEFAULT 'Draft',
    processed_date DATE,
    voucher_id UUID REFERENCES vouchers(id),
    
    total_gross DECIMAL(18,2),
    total_deductions DECIMAL(18,2),
    total_net DECIMAL(18,2),
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_payroll UNIQUE (tenant_id, year, month)
);

CREATE TABLE pay_slips (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payroll_id UUID NOT NULL REFERENCES payrolls(id),
    employee_id UUID NOT NULL REFERENCES employees(id),
    
    paid_days INT NOT NULL,
    lop_days INT DEFAULT 0,
    total_days INT NOT NULL,
    
    -- Earnings
    basic_salary DECIMAL(18,2),
    hra DECIMAL(18,2),
    conveyance DECIMAL(18,2),
    medical DECIMAL(18,2),
    special_allowance DECIMAL(18,2),
    other_earnings DECIMAL(18,2),
    gross_salary DECIMAL(18,2),
    
    -- Deductions
    employee_epf DECIMAL(18,2),
    employee_esi DECIMAL(18,2),
    professional_tax DECIMAL(18,2),
    tds DECIMAL(18,2),
    other_deductions DECIMAL(18,2),
    total_deductions DECIMAL(18,2),
    
    -- Employer Contributions
    employer_epf DECIMAL(18,2),
    employer_esi DECIMAL(18,2),
    
    net_salary DECIMAL(18,2),
    
    CONSTRAINT unique_payslip UNIQUE (payroll_id, employee_id)
);

-- Indexes
CREATE INDEX idx_employees_department ON employees(tenant_id, department_id);
CREATE INDEX idx_payslips_employee ON pay_slips(employee_id);
```

---

## **7. Frontend - Payroll Processing**

```typescript
// src/app/(dashboard)/hrms/payroll/process/page.tsx
'use client';

import { useState } from 'react';
import { useQuery, useMutation } from '@tanstack/react-query';
import { Card, CardHeader, CardContent } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Progress } from '@/components/ui/progress';

export default function PayrollProcessPage() {
  const [year] = useState(new Date().getFullYear());
  const [month] = useState(new Date().getMonth() + 1);
  
  const { data: payroll, refetch } = useQuery({
    queryKey: ['payroll', year, month],
    queryFn: () => fetch(`/api/v1/hrms/payroll/${year}/${month}`).then(r => r.json())
  });
  
  const processMutation = useMutation({
    mutationFn: () => fetch(`/api/v1/hrms/payroll/${year}/${month}/process`, { method: 'POST' }),
    onSuccess: () => refetch()
  });
  
  const voucherMutation = useMutation({
    mutationFn: () => fetch(`/api/v1/hrms/payroll/${payroll?.id}/generate-voucher`, { method: 'POST' }),
    onSuccess: () => refetch()
  });
  
  return (
    <div className="p-6 space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-2xl font-bold">
          Payroll Processing - {new Date(year, month - 1).toLocaleDateString('en-IN', { month: 'long', year: 'numeric' })}
        </h1>
        <div className="space-x-2">
          {payroll?.status === 'Draft' && (
            <Button onClick={() => processMutation.mutate()} disabled={processMutation.isPending}>
              Process Payroll
            </Button>
          )}
          {payroll?.status === 'Processed' && (
            <Button onClick={() => voucherMutation.mutate()} disabled={voucherMutation.isPending}>
              Generate Voucher
            </Button>
          )}
        </div>
      </div>
      
      {/* Summary Cards */}
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
        <Card>
          <CardContent className="pt-6">
            <p className="text-sm text-gray-500">Total Employees</p>
            <p className="text-2xl font-bold">{payroll?.employeeCount || 0}</p>
          </CardContent>
        </Card>
        <Card>
          <CardContent className="pt-6">
            <p className="text-sm text-gray-500">Total Gross</p>
            <p className="text-2xl font-bold">₹{payroll?.totalGross?.toLocaleString('en-IN') || 0}</p>
          </CardContent>
        </Card>
        <Card>
          <CardContent className="pt-6">
            <p className="text-sm text-gray-500">Total Deductions</p>
            <p className="text-2xl font-bold">₹{payroll?.totalDeductions?.toLocaleString('en-IN') || 0}</p>
          </CardContent>
        </Card>
        <Card>
          <CardContent className="pt-6">
            <p className="text-sm text-gray-500">Net Payable</p>
            <p className="text-2xl font-bold text-green-600">₹{payroll?.totalNet?.toLocaleString('en-IN') || 0}</p>
          </CardContent>
        </Card>
      </div>
      
      {/* Statutory Summary */}
      <Card>
        <CardHeader>
          <h2 className="text-lg font-medium">Statutory Contributions</h2>
        </CardHeader>
        <CardContent>
          <div className="grid grid-cols-2 md:grid-cols-5 gap-4">
            <div className="text-center p-4 bg-gray-50 rounded">
              <p className="text-sm text-gray-500">EPF (Employee)</p>
              <p className="text-xl font-bold">₹{payroll?.totalEmployeeEpf?.toLocaleString('en-IN') || 0}</p>
            </div>
            <div className="text-center p-4 bg-gray-50 rounded">
              <p className="text-sm text-gray-500">EPF (Employer)</p>
              <p className="text-xl font-bold">₹{payroll?.totalEmployerEpf?.toLocaleString('en-IN') || 0}</p>
            </div>
            <div className="text-center p-4 bg-gray-50 rounded">
              <p className="text-sm text-gray-500">ESI (Employee)</p>
              <p className="text-xl font-bold">₹{payroll?.totalEmployeeEsi?.toLocaleString('en-IN') || 0}</p>
            </div>
            <div className="text-center p-4 bg-gray-50 rounded">
              <p className="text-sm text-gray-500">ESI (Employer)</p>
              <p className="text-xl font-bold">₹{payroll?.totalEmployerEsi?.toLocaleString('en-IN') || 0}</p>
            </div>
            <div className="text-center p-4 bg-gray-50 rounded">
              <p className="text-sm text-gray-500">TDS</p>
              <p className="text-xl font-bold">₹{payroll?.totalTds?.toLocaleString('en-IN') || 0}</p>
            </div>
          </div>
        </CardContent>
      </Card>
      
      {/* Employee-wise Details */}
      <Card>
        <CardHeader>
          <h2 className="text-lg font-medium">Employee-wise Breakdown</h2>
        </CardHeader>
        <CardContent>
          <table className="w-full">
            <thead>
              <tr className="text-left text-sm text-gray-500 border-b">
                <th className="pb-2">Employee</th>
                <th className="pb-2 text-right">Gross</th>
                <th className="pb-2 text-right">EPF</th>
                <th className="pb-2 text-right">ESI</th>
                <th className="pb-2 text-right">PT</th>
                <th className="pb-2 text-right">TDS</th>
                <th className="pb-2 text-right">Net</th>
              </tr>
            </thead>
            <tbody>
              {payroll?.paySlips?.map((slip: any) => (
                <tr key={slip.employeeId} className="border-b">
                  <td className="py-3">
                    <p className="font-medium">{slip.employeeName}</p>
                    <p className="text-sm text-gray-500">{slip.employeeCode}</p>
                  </td>
                  <td className="text-right">₹{slip.grossSalary.toLocaleString('en-IN')}</td>
                  <td className="text-right">₹{slip.employeeEpf.toLocaleString('en-IN')}</td>
                  <td className="text-right">₹{slip.employeeEsi.toLocaleString('en-IN')}</td>
                  <td className="text-right">₹{slip.professionalTax.toLocaleString('en-IN')}</td>
                  <td className="text-right">₹{slip.tds.toLocaleString('en-IN')}</td>
                  <td className="text-right font-bold">₹{slip.netSalary.toLocaleString('en-IN')}</td>
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

## **8. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| Employee Master | Store all statutory details (PAN, UAN, ESI) |
| Salary Structure | Support multiple structures per employee |
| Payroll Processing | Process 500 employees in < 30s |
| EPF Calculation | 12% employee + employer contribution |
| ESI Calculation | 0.75% employee + 3.25% employer |
| TDS Calculation | Support both Old and New tax regimes |
| Pay Slip | Generate PDF pay slip |
| Voucher Integration | Create salary Journal voucher |

---

## **9. Definition of Done**

- [ ] Employee CRUD with all statutory fields
- [ ] Department and Designation masters
- [ ] Salary structure configuration
- [ ] Attendance tracking (manual + import)
- [ ] Leave management (apply, approve, balance)
- [ ] Payroll processing engine
- [ ] EPF, ESI, PT, TDS calculations
- [ ] Pay slip generation (PDF)
- [ ] Salary voucher generation
- [ ] PF ECR file generation
- [ ] ESI return data
- [ ] Form 16 Part B generation
- [ ] Integration tests for payroll calculations

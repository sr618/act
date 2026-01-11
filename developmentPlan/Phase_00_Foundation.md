# **Phase 0: Foundation & Infrastructure**
**Duration:** 6 Weeks  
**Team:** DevOps (2), Backend Lead (1), Architect (1)  
**Dependencies:** None (Starting Phase)  

---

## **1. Objectives**

- Set up development, staging, and production environments
- Establish CI/CD pipelines
- Implement Identity & Access Management
- Create shared infrastructure services
- Define coding standards and project structure

---

## **2. Deliverables**

### **Week 1-2: Development Environment Setup**

#### **2.1 Repository Structure**
```
/auditflow-erp
├── /src
│   ├── /Services
│   │   ├── /AuditFlow.Core.Accounting      # Core Accounting Engine
│   │   ├── /AuditFlow.Module.GST           # GST Compliance Module
│   │   ├── /AuditFlow.Module.Inventory     # Inventory Module
│   │   ├── /AuditFlow.Module.Manufacturing # Manufacturing Module
│   │   ├── /AuditFlow.Module.Banking       # Banking Module
│   │   ├── /AuditFlow.Module.HRMS          # HRMS Module
│   │   ├── /AuditFlow.Service.Identity     # Identity Service
│   │   ├── /AuditFlow.Service.Notification # Notification Service
│   │   ├── /AuditFlow.Service.AuditTrail   # Audit Trail Service
│   │   └── /AuditFlow.Service.Gateway      # API Gateway
│   ├── /Shared
│   │   ├── /AuditFlow.Shared.Kernel        # Domain primitives, base classes
│   │   ├── /AuditFlow.Shared.Contracts     # DTOs, Events, Commands
│   │   └── /AuditFlow.Shared.Infrastructure# Common infra code
│   ├── /Web
│   │   └── /auditflow-web                  # Next.js Frontend
│   └── /Mobile
│       └── /auditflow-mobile               # Flutter App
├── /tests
│   ├── /Unit
│   ├── /Integration
│   └── /E2E
├── /infrastructure
│   ├── /docker
│   ├── /kubernetes
│   └── /terraform
├── /docs
└── /tools
```

#### **2.2 Solution Structure (.NET)**
```csharp
// Each microservice follows Clean Architecture
/AuditFlow.Core.Accounting
├── /src
│   ├── /AuditFlow.Core.Accounting.Domain        # Entities, Value Objects, Domain Events
│   ├── /AuditFlow.Core.Accounting.Application   # Use Cases, Commands, Queries, DTOs
│   ├── /AuditFlow.Core.Accounting.Infrastructure# EF Core, External Services
│   └── /AuditFlow.Core.Accounting.API           # Controllers, Middleware
└── /tests
    ├── /AuditFlow.Core.Accounting.Domain.Tests
    ├── /AuditFlow.Core.Accounting.Application.Tests
    └── /AuditFlow.Core.Accounting.Integration.Tests
```

---

### **Week 2-3: CI/CD Pipeline**

#### **2.3 GitHub Actions Workflow**
```yaml
# .github/workflows/build-and-deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Test
        run: dotnet test --no-build --configuration Release --collect:"XPlat Code Coverage"
      
      - name: Publish Code Coverage
        uses: codecov/codecov-action@v3
        
      - name: Build Docker Images
        run: |
          docker build -t auditflow-core:${{ github.sha }} ./src/Services/AuditFlow.Core.Accounting
          
      - name: Push to Registry
        if: github.ref == 'refs/heads/main'
        run: |
          docker push ${{ secrets.REGISTRY }}/auditflow-core:${{ github.sha }}

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: kubectl apply -f ./infrastructure/kubernetes/staging/
```

#### **2.4 Environment Configuration**
```yaml
# infrastructure/docker/docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: auditflow
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: auditflow_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

  seq:
    image: datalust/seq:latest
    ports:
      - "5341:5341"
      - "8081:80"
    environment:
      ACCEPT_EULA: Y

volumes:
  postgres_data:
```

---

### **Week 3-4: Identity & Access Management**

#### **2.5 Identity Service Architecture**
```csharp
// Domain/Entities/User.cs
namespace AuditFlow.Service.Identity.Domain.Entities;

public class User : AggregateRoot<UserId>
{
    public string Email { get; private set; }
    public string PasswordHash { get; private set; }
    public string FirstName { get; private set; }
    public string LastName { get; private set; }
    public TenantId TenantId { get; private set; }
    public bool IsActive { get; private set; }
    public bool MfaEnabled { get; private set; }
    public string? MfaSecret { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? LastLoginAt { get; private set; }
    
    private readonly List<UserRole> _roles = new();
    public IReadOnlyCollection<UserRole> Roles => _roles.AsReadOnly();
    
    private readonly List<RefreshToken> _refreshTokens = new();
    public IReadOnlyCollection<RefreshToken> RefreshTokens => _refreshTokens.AsReadOnly();
}

// Domain/Entities/Tenant.cs
public class Tenant : AggregateRoot<TenantId>
{
    public string Name { get; private set; }
    public string SchemaName { get; private set; }  // For schema-per-tenant isolation
    public string GstinPrimary { get; private set; }
    public TenantSubscription Subscription { get; private set; }
    public TenantSettings Settings { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public bool IsActive { get; private set; }
    
    private readonly List<EnabledModule> _enabledModules = new();
    public IReadOnlyCollection<EnabledModule> EnabledModules => _enabledModules.AsReadOnly();
}

// Domain/Entities/EnabledModule.cs
public class EnabledModule : Entity<EnabledModuleId>
{
    public TenantId TenantId { get; private set; }
    public ModuleType ModuleType { get; private set; }  // GST, Inventory, Manufacturing, etc.
    public DateTime EnabledAt { get; private set; }
    public ModuleSettings Settings { get; private set; }
}

public enum ModuleType
{
    CoreAccounting = 1,  // Always enabled
    GSTCompliance = 2,
    Inventory = 3,
    Manufacturing = 4,
    Banking = 5,
    HRMS = 6,
    POS = 7,
    Reporting = 8
}
```

#### **2.6 JWT Token Configuration**
```csharp
// Infrastructure/Authentication/JwtTokenService.cs
public class JwtTokenService : ITokenService
{
    private readonly JwtSettings _settings;
    
    public TokenResponse GenerateTokens(User user, Tenant tenant)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user.Id.Value.ToString()),
            new(ClaimTypes.Email, user.Email),
            new("tenant_id", tenant.Id.Value.ToString()),
            new("tenant_schema", tenant.SchemaName),
            new("enabled_modules", JsonSerializer.Serialize(
                tenant.EnabledModules.Select(m => m.ModuleType.ToString())
            ))
        };
        
        // Add role claims
        foreach (var role in user.Roles)
        {
            claims.Add(new(ClaimTypes.Role, role.RoleName));
            
            // Add permission claims
            foreach (var permission in role.Permissions)
            {
                claims.Add(new("permission", permission.Name));
            }
        }
        
        var accessToken = GenerateAccessToken(claims);
        var refreshToken = GenerateRefreshToken();
        
        return new TokenResponse(accessToken, refreshToken, _settings.AccessTokenExpiryMinutes);
    }
}
```

#### **2.7 Role-Based Access Control (RBAC)**
```csharp
// Pre-defined Roles
public static class DefaultRoles
{
    public static readonly Role Admin = new("Admin", new[]
    {
        Permissions.All
    });
    
    public static readonly Role Accountant = new("Accountant", new[]
    {
        Permissions.Voucher.Create,
        Permissions.Voucher.Edit,
        Permissions.Voucher.View,
        Permissions.Ledger.Create,
        Permissions.Ledger.Edit,
        Permissions.Ledger.View,
        Permissions.Reports.View,
        Permissions.GST.View,
        Permissions.GST.File
    });
    
    public static readonly Role DataEntry = new("DataEntry", new[]
    {
        Permissions.Voucher.Create,
        Permissions.Voucher.View,
        Permissions.Ledger.View
    });
    
    public static readonly Role Viewer = new("Viewer", new[]
    {
        Permissions.Voucher.View,
        Permissions.Ledger.View,
        Permissions.Reports.View
    });
    
    public static readonly Role Owner = new("Owner", new[]
    {
        Permissions.All,
        Permissions.Settings.Manage,
        Permissions.Users.Manage,
        Permissions.Subscription.Manage
    });
}

// Permissions Structure
public static class Permissions
{
    public const string All = "*";
    
    public static class Voucher
    {
        public const string Create = "voucher:create";
        public const string Edit = "voucher:edit";
        public const string Delete = "voucher:delete";
        public const string View = "voucher:view";
        public const string Approve = "voucher:approve";
    }
    
    public static class Ledger
    {
        public const string Create = "ledger:create";
        public const string Edit = "ledger:edit";
        public const string Delete = "ledger:delete";
        public const string View = "ledger:view";
    }
    
    public static class GST
    {
        public const string View = "gst:view";
        public const string File = "gst:file";
        public const string GenerateEInvoice = "gst:einvoice";
    }
    
    // ... more permissions
}
```

---

### **Week 4-5: Shared Infrastructure Services**

#### **2.8 Multi-Tenant Database Context**
```csharp
// Shared.Infrastructure/Persistence/TenantDbContext.cs
public abstract class TenantDbContext : DbContext
{
    private readonly ITenantProvider _tenantProvider;
    private readonly ICurrentUserService _currentUser;
    
    protected TenantDbContext(
        DbContextOptions options,
        ITenantProvider tenantProvider,
        ICurrentUserService currentUser) : base(options)
    {
        _tenantProvider = tenantProvider;
        _currentUser = currentUser;
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Set schema based on tenant
        var schema = _tenantProvider.GetCurrentSchema();
        modelBuilder.HasDefaultSchema(schema);
        
        base.OnModelCreating(modelBuilder);
    }
    
    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Auto-populate audit fields
        foreach (var entry in ChangeTracker.Entries<IAuditable>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = DateTime.UtcNow;
                    entry.Entity.CreatedBy = _currentUser.UserId;
                    break;
                case EntityState.Modified:
                    entry.Entity.ModifiedAt = DateTime.UtcNow;
                    entry.Entity.ModifiedBy = _currentUser.UserId;
                    break;
            }
        }
        
        return await base.SaveChangesAsync(cancellationToken);
    }
}

// Tenant Schema Management
public class TenantSchemaManager
{
    public async Task CreateTenantSchemaAsync(string schemaName)
    {
        // Create new schema for tenant
        await _connection.ExecuteAsync($"CREATE SCHEMA IF NOT EXISTS {schemaName}");
        
        // Run migrations for the new schema
        await _migrationRunner.MigrateAsync(schemaName);
    }
}
```

#### **2.9 Audit Trail Service**
```csharp
// Service.AuditTrail/Domain/AuditLogEntry.cs
public class AuditLogEntry
{
    public Guid Id { get; init; }
    public Guid TenantId { get; init; }
    public Guid UserId { get; init; }
    public string UserEmail { get; init; }
    public DateTime Timestamp { get; init; }
    public string EntityType { get; init; }
    public string EntityId { get; init; }
    public AuditAction Action { get; init; }  // Create, Update, Delete
    public JsonDocument OldValues { get; init; }
    public JsonDocument NewValues { get; init; }
    public string IpAddress { get; init; }
    public string UserAgent { get; init; }
    
    // Hash for tamper detection
    public string ContentHash { get; init; }
    public string PreviousEntryHash { get; init; }  // Blockchain-like chain
}

public enum AuditAction
{
    Create = 1,
    Update = 2,
    Delete = 3,
    View = 4,
    Export = 5,
    Print = 6
}

// PostgreSQL Trigger for Automatic Audit Logging
/*
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
DECLARE
    old_data jsonb;
    new_data jsonb;
BEGIN
    IF TG_OP = 'INSERT' THEN
        new_data = row_to_json(NEW)::jsonb;
        INSERT INTO audit_logs(id, tenant_id, entity_type, entity_id, action, new_values, timestamp)
        VALUES (gen_random_uuid(), current_setting('app.tenant_id')::uuid, TG_TABLE_NAME, NEW.id, 'CREATE', new_data, now());
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        old_data = row_to_json(OLD)::jsonb;
        new_data = row_to_json(NEW)::jsonb;
        INSERT INTO audit_logs(id, tenant_id, entity_type, entity_id, action, old_values, new_values, timestamp)
        VALUES (gen_random_uuid(), current_setting('app.tenant_id')::uuid, TG_TABLE_NAME, NEW.id, 'UPDATE', old_data, new_data, now());
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        old_data = row_to_json(OLD)::jsonb;
        INSERT INTO audit_logs(id, tenant_id, entity_type, entity_id, action, old_values, timestamp)
        VALUES (gen_random_uuid(), current_setting('app.tenant_id')::uuid, TG_TABLE_NAME, OLD.id, 'DELETE', old_data, now());
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;
*/
```

#### **2.10 Event Bus Configuration**
```csharp
// Shared.Infrastructure/Messaging/RabbitMqEventBus.cs
public class RabbitMqEventBus : IEventBus
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    
    public async Task PublishAsync<T>(T @event) where T : IntegrationEvent
    {
        var eventName = @event.GetType().Name;
        var message = JsonSerializer.Serialize(@event);
        var body = Encoding.UTF8.GetBytes(message);
        
        _channel.BasicPublish(
            exchange: "auditflow.events",
            routingKey: eventName,
            basicProperties: null,
            body: body
        );
    }
    
    public void Subscribe<T, THandler>() 
        where T : IntegrationEvent 
        where THandler : IIntegrationEventHandler<T>
    {
        var eventName = typeof(T).Name;
        _channel.QueueBind(
            queue: $"auditflow.{eventName}",
            exchange: "auditflow.events",
            routingKey: eventName
        );
    }
}

// Example Integration Events (Cross-Module Communication)
public record VoucherCreatedEvent(
    Guid VoucherId,
    Guid TenantId,
    VoucherType Type,
    decimal Amount,
    DateTime VoucherDate
) : IntegrationEvent;

public record InventoryUpdatedEvent(
    Guid ItemId,
    Guid TenantId,
    Guid GodownId,
    decimal Quantity,
    string TransactionType
) : IntegrationEvent;
```

---

### **Week 5-6: API Gateway & Frontend Setup**

#### **2.11 API Gateway Configuration (YARP)**
```csharp
// Service.Gateway/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Identity:Authority"];
        options.Audience = "auditflow-api";
    });

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// Custom middleware for module access check
app.UseMiddleware<ModuleAccessMiddleware>();

app.MapReverseProxy();
app.Run();

// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "core-route": {
        "ClusterId": "core-cluster",
        "Match": { "Path": "/api/v1/accounting/{**catch-all}" }
      },
      "gst-route": {
        "ClusterId": "gst-cluster",
        "Match": { "Path": "/api/v1/gst/{**catch-all}" },
        "Metadata": { "RequiredModule": "GSTCompliance" }
      },
      "inventory-route": {
        "ClusterId": "inventory-cluster",
        "Match": { "Path": "/api/v1/inventory/{**catch-all}" },
        "Metadata": { "RequiredModule": "Inventory" }
      }
    },
    "Clusters": {
      "core-cluster": {
        "Destinations": {
          "core-1": { "Address": "http://core-accounting:8080" }
        }
      },
      "gst-cluster": {
        "Destinations": {
          "gst-1": { "Address": "http://gst-compliance:8080" }
        }
      }
    }
  }
}
```

#### **2.12 Module Access Middleware**
```csharp
// Ensures tenant has access to the requested module
public class ModuleAccessMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        var endpoint = context.GetEndpoint();
        var requiredModule = endpoint?.Metadata.GetMetadata<string>("RequiredModule");
        
        if (!string.IsNullOrEmpty(requiredModule))
        {
            var enabledModules = context.User.FindFirst("enabled_modules")?.Value;
            var modules = JsonSerializer.Deserialize<string[]>(enabledModules ?? "[]");
            
            if (!modules.Contains(requiredModule))
            {
                context.Response.StatusCode = 403;
                await context.Response.WriteAsJsonAsync(new 
                { 
                    Error = "Module not enabled",
                    Message = $"Your subscription does not include the {requiredModule} module."
                });
                return;
            }
        }
        
        await _next(context);
    }
}
```

#### **2.13 Next.js Project Setup**
```typescript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    serverActions: true,
  },
  // Server-side rendering by default
  output: 'standalone',
}

module.exports = nextConfig

// src/lib/api-client.ts
import { cookies } from 'next/headers';

const API_BASE = process.env.API_GATEWAY_URL;

export async function serverFetch<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const cookieStore = cookies();
  const token = cookieStore.get('access_token')?.value;

  const response = await fetch(`${API_BASE}${endpoint}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
      ...options.headers,
    },
  });

  if (!response.ok) {
    throw new Error(`API Error: ${response.status}`);
  }

  return response.json();
}

// src/app/layout.tsx
import { Inter } from 'next/font/google';
import { Providers } from './providers';
import { Sidebar } from '@/components/layout/sidebar';
import { Header } from '@/components/layout/header';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={inter.className}>
        <Providers>
          <div className="flex h-screen">
            <Sidebar />
            <div className="flex-1 flex flex-col">
              <Header />
              <main className="flex-1 overflow-auto bg-gray-50 dark:bg-gray-900">
                {children}
              </main>
            </div>
          </div>
        </Providers>
      </body>
    </html>
  );
}
```

---

## **3. Acceptance Criteria**

| Criteria | Target |
|----------|--------|
| All environments (Dev/Staging/Prod) provisioned | ✓ |
| CI/CD pipeline with automated tests | ✓ |
| Code coverage > 80% for shared libraries | ✓ |
| Identity service with JWT + MFA | ✓ |
| Multi-tenant schema creation working | ✓ |
| Audit trail capturing all DB changes | ✓ |
| API Gateway routing to modules | ✓ |
| Next.js project with auth integration | ✓ |
| Docker Compose for local development | ✓ |
| Kubernetes manifests for deployment | ✓ |

---

## **4. Definition of Done**

- [ ] All code reviewed and merged to main
- [ ] Unit tests passing with >80% coverage
- [ ] Integration tests passing
- [ ] Documentation updated
- [ ] Security scan completed (no critical/high vulnerabilities)
- [ ] Performance baseline established
- [ ] Staging deployment successful
- [ ] Architect sign-off received

---

## **5. Dependencies for Next Phase**

Phase 1 (Core Accounting) requires:
- ✅ Database infrastructure ready
- ✅ Identity service deployed
- ✅ Audit trail service operational
- ✅ Event bus configured
- ✅ API Gateway running
- ✅ Frontend project scaffolded

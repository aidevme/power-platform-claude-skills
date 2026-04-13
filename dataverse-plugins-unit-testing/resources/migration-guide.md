# Migration Guide: v1.x to v2.x/v3.x

Complete guide for migrating from FakeXrmEasy v1.x to v2.x (.NET Framework) or v3.x (.NET Core).

## Overview

**Why Migrate?**
- **v1.x is deprecated** - No longer maintained since December 2022
- **Performance improvements** - Up to 10x faster
- **New features** - Bulk operations, file storage, ILogger support
- **Better testing patterns** - Middleware architecture
- **Future-proof** - Only v2.x/v3.x receive updates
- **Security** - Critical bugs only fixed in v2.x/v3.x
- **Cloud-native** - v3.x supports .NET Core, Azure Functions

**Timeline:**
- June 2022: v2.x/v3.x released, v1.x deprecation announced
- December 2022: v1.x end of life

## Version Selection

Choose based on your target runtime:

| Scenario | Version | .NET Version | Use Case |
|----------|---------|--------------|----------|
| Server-side plugins | v2.x | .NET Framework 4.6.2+ | Traditional Dataverse plugins |
| Client-side/Azure Functions | v3.x | .NET Core 3.1+ | Modern cloud applications |
| Both | Both | Dual | Shared business logic libraries |

### v2.x (Server-Side)

**Use when:**
- Testing traditional Dataverse plugins (.NET Framework)
- Workflow activities (CodeActivities)
- Custom Actions running server-side
- Legacy .NET Framework projects

**Package:** `FakeXrmEasy.v9` version 2.x

```powershell
Install-Package FakeXrmEasy.v9 -Version 2.6.3
```

### v3.x (Client-Side)

**Use when:**
- Azure Functions with Dataverse
- ASP.NET Core applications
- Modern .NET 6/7/8 projects
- Cross-platform development
- Async/await patterns (IOrganizationServiceAsync2)

**Package:** `FakeXrmEasy.v9` version 3.x

```powershell
Install-Package FakeXrmEasy.v9 -Version 3.6.3
```

## Migration Steps

### Step 1: Update NuGet Packages

#### Remove v1.x Packages

```powershell
# Remove old packages
Uninstall-Package FakeXrmEasy
Uninstall-Package FakeXrmEasy.9
```

#### Install v2.x/v3.x Packages

```powershell
# For .NET Framework (v2.x)
Install-Package FakeXrmEasy.v9 -Version 2.6.3

# For .NET Core (v3.x)
Install-Package FakeXrmEasy.v9 -Version 3.6.3
```

### Step 2: Update Namespaces

**v1.x namespaces:**
```csharp
using FakeXrmEasy;
using FakeXrmEasy.Extensions;
using FakeXrmEasy.FakeMessageExecutors;
```

**v2.x/v3.x namespaces:**
```csharp
using FakeXrmEasy;
using FakeXrmEasy.Abstractions;
using FakeXrmEasy.Abstractions.Plugins;
using FakeXrmEasy.Middleware;
using FakeXrmEasy.Plugins;
```

### Step 3: Update Context Initialization

**v1.x approach:**
```csharp
// OLD - v1.x
var context = new XrmFakedContext();
var service = context.GetOrganizationService();

context.Initialize(new[] { account, contact });
```

**v2.x/v3.x approach:**
```csharp
// NEW - v2.x/v3.x with Middleware
var context = MiddlewareBuilder.New()
    .AddCrud()                    // Enable CRUD operations
    .AddFakeMessageExecutors()    // Enable standard messages
    .UseCrud()                    // Use CRUD middleware
    .SetLicense(FakeXrmEasyLicense.RPL_1_5)  // Set license
    .Build();
    
var service = context.GetOrganizationService();

context.Initialize(new[] { account, contact });
```

### Step 4: Update Plugin Execution

**v1.x approach:**
```csharp
// OLD - v1.x
context.ExecutePluginWith<AccountPlugin>(target);
```

**v2.x/v3.x approach:**
```csharp
// NEW - v2.x/v3.x
context.ExecutePluginWith<AccountPlugin>(
    target,
    messageName: "Create",
    stage: ProcessingStepStage.Postoperation,
    mode: ProcessingStepMode.Synchronous
);

// Alternative: More explicit syntax
var pluginContext = context.GetDefaultPluginContext();
pluginContext.MessageName = "Create";
pluginContext.Stage = (int)ProcessingStepStage.Postoperation;
pluginContext.Mode = (int)ProcessingStepMode.Synchronous;
pluginContext.InputParameters["Target"] = target;

context.ExecutePluginWith(pluginContext, new AccountPlugin());
```

### Step 5: Update Message Executors

**v1.x approach:**
```csharp
// OLD - v1.x
context.Initialize(new[] { account });

var request = new AssignRequest
{
    Target = account.ToEntityReference(),
    Assignee = newOwner.ToEntityReference()
};

var response = (AssignResponse)service.Execute(request);
```

**v2.x/v3.x approach:**
```csharp
// NEW - v2.x/v3.x - Add specific message executor
var context = MiddlewareBuilder.New()
    .AddCrud()
    .AddFakeMessageExecutors(typeof(AssignRequestExecutor).Assembly)
    .UseCrud()
    .Build();

var service = context.GetOrganizationService();
context.Initialize(new[] { account });

var request = new AssignRequest
{
    Target = account.ToEntityReference(),
    Assignee = newOwner.ToEntityReference()
};

var response = (AssignResponse)service.Execute(request);
```

### Step 6: Update Pipeline Simulation

**v1.x approach:**
```csharp
// OLD - v1.x (limited pipeline support)
context.RegisterPluginStep<PreValidationPlugin>("account", ProcessingStepStage.Prevalidation);
```

**v2.x/v3.x approach:**
```csharp
// NEW - v2.x/v3.x with full pipeline
var context = MiddlewareBuilder.New()
    .AddCrud()
    .AddPipelineSimulation()      // Add pipeline simulation
    .UseCrud()
    .UsePipelineSimulation()      // Enable pipeline
    .Build();

context.RegisterPluginStep<PreValidationPlugin>(
    messageName: "Create",
    entityLogicalName: "account",
    executionStage: ProcessingStepStage.Prevalidation,
    executionMode: ProcessingStepMode.Synchronous
);
```

### Step 7: Update Licensing

**NOTE:** v2.x/v3.x require license configuration (free for non-commercial use).

```csharp
var context = MiddlewareBuilder.New()
    .AddCrud()
    .SetLicense(FakeXrmEasyLicense.RPL_1_5)  // Reciprocal Public License
    .UseCrud()
    .Build();
```

**License options:**
- `FakeXrmEasyLicense.RPL_1_5` - Free for open source/non-commercial
- Commercial license - For proprietary/commercial projects (contact DynamicsValue)

## Common Migration Scenarios

### Scenario 1: Basic Unit Tests

**Before (v1.x):**
```csharp
[Fact]
public void Should_Create_Account_v1()
{
    var context = new XrmFakedContext();
    var service = context.GetOrganizationService();
    
    var account = new Account { Name = "Contoso" };
    var accountId = service.Create(account);
    
    Assert.NotEqual(Guid.Empty, accountId);
}
```

**After (v2.x/v3.x):**
```csharp
[Fact]
public void Should_Create_Account_v2()
{
    var context = MiddlewareBuilder.New()
        .AddCrud()
        .UseCrud()
        .SetLicense(FakeXrmEasyLicense.RPL_1_5)
        .Build();
        
    var service = context.GetOrganizationService();
    
    var account = new Account { Name = "Contoso" };
    var accountId = service.Create(account);
    
    Assert.NotEqual(Guid.Empty, accountId);
}
```

### Scenario 2: Plugin Testing

**Before (v1.x):**
```csharp
[Fact]
public void Should_Execute_Plugin_v1()
{
    var context = new XrmFakedContext();
    var account = new Account { Name = "Contoso" };
    
    context.ExecutePluginWith<AccountPlugin>(account);
    
    // Assert...
}
```

**After (v2.x/v3.x):**
```csharp
[Fact]
public void Should_Execute_Plugin_v2()
{
    var context = MiddlewareBuilder.New()
        .AddCrud()
        .UseCrud()
        .SetLicense(FakeXrmEasyLicense.RPL_1_5)
        .Build();
    
    var account = new Account { Name = "Contoso" };
    
    context.ExecutePluginWith<AccountPlugin>(
        account.ToEntity<Entity>(),
        "Create",
        ProcessingStepStage.Postoperation
    );
    
    // Assert...
}
```

### Scenario 3: Base Test Class

**Before (v1.x):**
```csharp
public class TestsBase
{
    protected XrmFakedContext _context;
    protected IOrganizationService _service;
    
    public TestsBase()
    {
        _context = new XrmFakedContext();
        _service = _context.GetOrganizationService();
    }
}
```

**After (v2.x/v3.x):**
```csharp
public class TestsBase
{
    protected IXrmFakedContext _context;
    protected IOrganizationService _service;
    
    public TestsBase()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .AddFakeMessageExecutors()
            .UseCrud()
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();
            
        _service = _context.GetOrganizationService();
    }
}
```

## Breaking Changes

### 1. Context Type Changed

- **v1.x:** `XrmFakedContext` class
- **v2.x/v3.x:** `IXrmFakedContext` interface via `MiddlewareBuilder`

### 2. ExecutePluginWith Signature

- **v1.x:** `ExecutePluginWith<T>(Entity target)`
- **v2.x/v3.x:** `ExecutePluginWith<T>(Entity target, string messageName, ProcessingStepStage stage)`

### 3. Namespace Changes

- **v1.x:** `FakeXrmEasy.Extensions`
- **v2.x/v3.x:** `FakeXrmEasy.Abstractions`, `FakeXrmEasy.Middleware`

### 4. Licensing Requirement

- **v1.x:** No license needed
- **v2.x/v3.x:** License configuration required (free RPL_1_5 available)

### 5. Message Executors

- **v1.x:** Auto-registered
- **v2.x/v3.x:** Explicit registration via `AddFakeMessageExecutors()`

## Troubleshooting

### Common Error: "CRUD operations not enabled"

**Solution:** Add CRUD middleware
```csharp
.AddCrud()
.UseCrud()
```

### Common Error: "License not configured"

**Solution:** Set license
```csharp
.SetLicense(FakeXrmEasyLicense.RPL_1_5)
```

### Common Error: "Message executor not found"

**Solution:** Register message executors
```csharp
.AddFakeMessageExecutors()
```

### Common Error: "ExecutePluginWith requires message name"

**Solution:** Provide explicit parameters
```csharp
context.ExecutePluginWith<Plugin>(
    target,
    "Create",  // Add message name
    ProcessingStepStage.Postoperation  // Add stage
);
```

## Best Practices for Migration

1. **Migrate tests incrementally** - Don't try to migrate everything at once
2. **Create base test class** - Centralize middleware configuration
3. **Use dependency injection** - Easier to swap FakeXrmEasy implementations
4. **Test in isolation** - Each test should initialize its own context
5. **Update to latest versions** - v2.6.3+ or v3.6.3+ have latest fixes
6. **Read release notes** - Check for version-specific breaking changes
7. **Use early-bound entities** - Better IntelliSense and type safety
8. **Leverage new features** - Bulk operations, file storage, ILogger

## Additional Resources

- **Official Documentation:** https://dynamicsvalue.github.io/fake-xrm-easy-docs/
- **Migration Guide:** https://dynamicsvalue.github.io/fake-xrm-easy-docs/migration/from-fake-xrm-easy-v1/
- **GitHub Issues:** https://github.com/DynamicsValue/fake-xrm-easy/issues
- **Community Support:** Stack Overflow tag `fake-xrm-easy`

## Summary Checklist

- [ ] Update NuGet packages (remove v1.x, install v2.x/v3.x)
- [ ] Update namespaces (FakeXrmEasy.Abstractions, FakeXrmEasy.Middleware)
- [ ] Configure middleware (MiddlewareBuilder with AddCrud, UseCrud)
- [ ] Set license (FakeXrmEasyLicense.RPL_1_5)
- [ ] Update ExecutePluginWith calls (add messageName, stage parameters)
- [ ] Register message executors (AddFakeMessageExecutors)
- [ ] Update base test classes (use IXrmFakedContext interface)
- [ ] Test all scenarios (CRUD, plugins, messages, queries)
- [ ] Remove v1.x compatibility code
- [ ] Document migration for team

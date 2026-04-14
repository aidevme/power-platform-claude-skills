# Dataverse Plugin Unit Testing - Comprehensive Documentation

Complete guide for unit testing Microsoft Dataverse plugins using the **FakeXrmEasy** framework. This documentation consolidates all testing patterns, scenarios, and best practices from the `dataverse-plugins-unit-testing` skill.

**Framework:** FakeXrmEasy v2.x (.NET Framework) / v3.x (.NET Core)  
**License:** MIT  
**Version:** 1.0.0 _(See [VERSION](../../../VERSION) file for current version)_

---

## Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Setup and Configuration](#setup-and-configuration)
4. [Testing Patterns (AAA)](#testing-patterns-aaa)
5. [Pipeline Simulation](#pipeline-simulation)
6. [Testing Queries](#testing-queries)
7. [Testing Relationships](#testing-relationships)
8. [Custom Messages & APIs](#custom-messages--apis)
9. [Advanced Scenarios](#advanced-scenarios)
10. [Async Testing](#async-testing)
11. [Bulk Operations](#bulk-operations)
12. [CodeActivities Testing](#codeactivities-testing)
13. [Azure Functions Testing](#azure-functions-testing)
14. [File Storage Testing](#file-storage-testing)
15. [Logging & Telemetry](#logging--telemetry)
16. [Migration Guide (v1.x → v2.x/v3.x)](#migration-guide)
17. [Best Practices](#best-practices)
18. [Troubleshooting](#troubleshooting)

---

## Overview

### What is FakeXrmEasy?

FakeXrmEasy is a **state-based, data-driven** testing framework for Dataverse plugins. It provides an **in-memory** implementation of `IOrganizationService` that simulates Dataverse without requiring a live connection.

**Key Benefits:**
- ✅ Fast test execution (no network latency)
- ✅ Isolated tests (no shared state between tests)
- ✅ Repeatable results (deterministic)
- ✅ No environment setup required
- ✅ Test-driven development (TDD) friendly
- ✅ Pipeline simulation for integration testing
- ✅ Supports FetchXML, QueryExpression, LINQ queries

### CRITICAL RULES

1. **Use FakeXrmEasy for all plugin unit tests.** Never use integration tests that require a live Dataverse connection when a unit test will suffice.

2. **Always inherit from `FakeXrmEasyTestsBase`.** This provides `_context` and `_service` fields pre-configured with the middleware pipeline.

3. **Test plugins in isolation.** Mock external dependencies. Don't make real HTTP calls, database connections, or file system operations.

4. **Use Pipeline Simulation for integration-style plugin tests.** When testing multiple plugins firing in sequence, use `RegisterPluginStep` and enable pipeline simulation.

5. **Assert against the Target entity reference for PreOperation changes.** Plugins that modify Target in PreOperation don't call `service.Update()`. Assert against the same object reference you passed into the test.

6. **Test negative cases and exception handling.** Verify that plugins throw `InvalidPluginExecutionException` with the correct message when validation fails.

7. **Use early-bound entities when possible.** Call `_context.EnableProxyTypes(Assembly)` to return strongly-typed entities from queries.

8. **Test bulk operations for performance-critical code.** Use `CreateMultipleRequest`, `UpdateMultipleRequest`, and `UpsertMultipleRequest` (v2.5+/v3.5+) when processing multiple records.

### Version Selection Guide

| Scenario | Version | .NET Target | Package |
|----------|---------|-------------|---------|
| Server-side plugins | v2.x | .NET Framework 4.6.2+ | `FakeXrmEasy.Plugins.v9` 2.6.3+ |
| Azure Functions / Client apps | v3.x | .NET Core 3.1+ | `FakeXrmEasy.v9` 3.6.3+ |
| Workflow activities (CodeActivities) | v2.x | .NET Framework | `FakeXrmEasy.CodeActivities.v9` 2.6.3+ |

---

## Quick Start

### Installation

```powershell
# For .NET Framework plugins (most common)
Install-Package FakeXrmEasy.Plugins.v9 -Version 2.6.3

# For .NET Core/Azure Functions
Install-Package FakeXrmEasy.v9 -Version 3.6.3

# For testing workflow activities
Install-Package FakeXrmEasy.CodeActivities.v9 -Version 2.6.3
```

### Your First Test

```csharp
using FakeXrmEasy;
using FakeXrmEasy.Middleware;
using FakeXrmEasy.Plugins;
using Microsoft.Xrm.Sdk;
using Xunit;

public class AccountNumberPluginTests : FakeXrmEasyTestsBase
{
    [Fact]
    public void When_Account_Created_Should_Set_Account_Number()
    {
        // ARRANGE: Setup test data
        var accountId = Guid.NewGuid();
        var target = new Account { Id = accountId, Name = "Contoso" };
        
        var pluginContext = _context.GetDefaultPluginContext();
        pluginContext.InputParameters["Target"] = target;
        pluginContext.MessageName = "Create";
        pluginContext.Stage = 20; // PreOperation
        
        // ACT: Execute the plugin
        _context.ExecutePluginWith<AccountNumberPlugin>(pluginContext);
        
        // ASSERT: Verify results
        Assert.NotNull(target.AccountNumber);
        Assert.StartsWith("ACC-", target.AccountNumber);
    }
}

public abstract class FakeXrmEasyTestsBase
{
    protected readonly IXrmFakedContext _context;
    protected readonly IOrganizationService _service;
    
    protected FakeXrmEasyTestsBase()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .AddFakeMessageExecutors()
            .UseCrud()
            .UseMessages()
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();
            
        _service = _context.GetOrganizationService();
    }
}
```

### Quick Reference Table

| Task | Method/Pattern |
|------|----------------|
| Install package | `Install-Package FakeXrmEasy.Plugins.v9 -Version 2.6.3` |
| Base test class | Inherit from `FakeXrmEasyTestsBase` |
| Initialize data | `_context.Initialize(new[] { account1, contact1 })` |
| Execute plugin | `_context.ExecutePluginWith<MyPlugin>(pluginContext)` |
| Simple execution | `_context.ExecutePluginWithTarget<MyPlugin>(targetEntity)` |
| Pipeline simulation | `_context.RegisterPluginStep<MyPlugin>("Create", stage)` |
| Query results | `_context.CreateQuery<Account>().Where(a => a.Name == "test")` |
| Entity images | Register with `PluginImageDefinition` in `RegisterPluginStep` |
| Mock dependencies | Use constructor injection + `PluginInstance` parameter |
| Bulk operations | `CreateMultipleRequest`, `UpdateMultipleRequest` (v2.5+/v3.5+) |
| Async service | `_context.GetAsyncOrganizationService2()` (v3.x only) |

---

## Setup and Configuration

### Package Installation

FakeXrmEasy has two major version lines:

#### Version 2.x — .NET Framework

Use for traditional plugin projects targeting .NET Framework 4.6.2+.

```powershell
# For Dataverse / Dynamics 365 (v9.x)
Install-Package FakeXrmEasy.Plugins.v9 -Version 2.6.3

# For older versions
Install-Package FakeXrmEasy.Plugins.v365 -Version 2.x  # D365 v8.2
Install-Package FakeXrmEasy.Plugins.v2016 -Version 2.x # D365 v8.1
```

**When to use:**
- Server-side plugin development (most common)
- Plugin projects that must target .NET Framework
- Testing traditional workflow activities (CodeActivities)

#### Version 3.x — .NET Core 3.1+

Use for modern applications targeting .NET Core/5+.

```powershell
Install-Package FakeXrmEasy.v9 -Version 3.6.3
```

**When to use:**
- Azure Functions that interact with Dataverse
- .NET Core/5/6/7+ applications
- Modern client applications
- **NOT** for plugins deployed to Dataverse (use v2.x)

### Project Structure

Recommended solution structure:

```
MySolution/
├── MyPlugins/                  # Plugin implementation project
│   ├── MyPlugins.csproj        # .NET Framework 4.6.2
│   ├── Plugins/
│   │   ├── AccountNumberPlugin.cs
│   │   └── ValidateContactPlugin.cs
│   └── EarlyBound/
│       └── Entities.cs         # Optional: generated entities
│
└── MyPlugins.Tests/             # Test project
    ├── MyPlugins.Tests.csproj   # Same framework as MyPlugins
    ├── AccountNumberPluginTests.cs
    ├── ValidateContactPluginTests.cs
    └── TestBase.cs              # Shared base class
```

### Test Project Configuration

**MyPlugins.Tests.csproj** (SDK-style):

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net462</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="FakeXrmEasy.Plugins.v9" Version="2.6.3" />
    <PackageReference Include="xunit" Version="2.4.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.4.5" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.5.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyPlugins\MyPlugins.csproj" />
  </ItemGroup>
</Project>
```

### Test Runners

FakeXrmEasy works with all major .NET test frameworks.

#### xUnit (Recommended)

```csharp
using Xunit;

public class AccountPluginTests : FakeXrmEasyTestsBase
{
    [Fact]
    public void Should_Generate_Account_Number()
    {
        // Test implementation
    }
    
    [Theory]
    [InlineData("Contoso", "ACC-CONTOSO")]
    [InlineData("Fabrikam", "ACC-FABRIKAM")]
    public void Should_Generate_Based_On_Name(string name, string expected)
    {
        // Theory tests multiple scenarios
    }
}
```

#### NUnit

```csharp
using NUnit.Framework;

[TestFixture]
public class AccountPluginTests : FakeXrmEasyTestsBase
{
    [Test]
    public void Should_Generate_Account_Number()
    {
        // Test implementation
    }
}
```

#### MSTest

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class AccountPluginTests : FakeXrmEasyTestsBase
{
    [TestMethod]
    public void Should_Generate_Account_Number()
    {
        // Test implementation
    }
}
```

### Base Test Class Pattern

Create a shared base class for common setup:

```csharp
using FakeXrmEasy;
using FakeXrmEasy.Middleware;
using System.Reflection;

public abstract class PluginTestBase : FakeXrmEasyTestsBase
{
    protected PluginTestBase()
    {
        // Enable early-bound entities
        _context.EnableProxyTypes(Assembly.GetExecutingAssembly());
        
        // Optional: Set organization name
        _context.OrganizationName = "TestOrg";
    }
    
    // Helper methods
    protected Guid CreateTestAccount(string name)
    {
        var account = new Account { Id = Guid.NewGuid(), Name = name };
        _context.Initialize(new[] { account });
        return account.Id;
    }
    
    protected Contact CreateTestContact(string firstName, string lastName)
    {
        var contact = new Contact 
        { 
            Id = Guid.NewGuid(), 
            FirstName = firstName,
            LastName = lastName 
        };
        _context.Initialize(new[] { contact });
        return contact;
    }
}
```

### Middleware Configuration

The middleware builder configures which messages and features are available:

```csharp
public class FakeXrmEasyTestsBase
{
    protected readonly IXrmFakedContext _context;
    protected readonly IOrganizationService _service;
    
    public FakeXrmEasyTestsBase()
    {
        _context = MiddlewareBuilder.New()
            // Add capabilities
            .AddCrud()                    // Create, Retrieve, Update, Delete, Associate, Disassociate
            .AddFakeMessageExecutors()    // Standard messages (Assign, SetState, etc.)
            
            // Use capabilities (order matters!)
            .UseCrud()
            .UseMessages()
            
            // License (required for v2.x/v3.x)
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            
            .Build();
            
        _service = _context.GetOrganizationService();
    }
}
```

**Middleware Options:**

- `.AddCrud()` - Basic CRUD operations
- `.AddFakeMessageExecutors()` - Standard Dataverse messages
- `.AddPipelineSimulation()` - Plugin pipeline functionality
- `.AddGenericFakeMessageExecutors()` - Custom action/API support
- `.SetLicense()` - Set license type (RPL_1_5 for open source/non-commercial)

---

## Testing Patterns (AAA)

Every plugin unit test follows the **Arrange-Act-Assert** pattern.

### The AAA Pattern Explained

1. **Arrange** — Set up test data, configure the plugin context, initialize the in-memory database
2. **Act** — Execute the plugin against the test context
3. **Assert** — Verify expected outcomes by querying the context or inspecting entities

### Basic Plugin Execution

#### ExecutePluginWith — Full Control

Use when you need complete control over the plugin execution context:

```csharp
[Fact]
public void When_Account_Created_Should_Set_Account_Number()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var target = new Account { Id = accountId, Name = "Contoso" };
    
    var pluginContext = _context.GetDefaultPluginContext();
    pluginContext.MessageName = "Create";
    pluginContext.Stage = 20; // PreOperation
    pluginContext.PrimaryEntityName = "account";
    pluginContext.InputParameters["Target"] = target;
    pluginContext.OutputParameters["id"] = accountId;
    
    // ACT
    _context.ExecutePluginWith<AccountNumberPlugin>(pluginContext);
    
    // ASSERT
    Assert.NotNull(target.AccountNumber);
    Assert.StartsWith("ACC-", target.AccountNumber);
}
```

**When to use:**
- Testing specific stages (PreValidation, PreOperation, PostOperation)
- Testing specific messages (Create, Update, Delete, custom actions)
- Need to set SharedVariables, ParentContext, or other context properties

#### ExecutePluginWithTarget — Simple Scenarios

Use for simple plugins that just need a Target entity:

```csharp
[Fact]
public void When_Target_Is_Account_Should_Set_Default_Values()
{
    // ARRANGE
    var account = new Account { Id = Guid.NewGuid(), Name = "Contoso" };
    
    // ACT
    _context.ExecutePluginWithTarget<DefaultValuesPlugin>(account);
    
    // ASSERT
    Assert.Equal("Default Industry", account.IndustryCode?.Value);
    Assert.NotNull(account.CreatedOn);
}
```

**When to use:**
- Simple plugins with minimal context dependencies
- Plugins that only read/modify the Target entity
- Quick smoke tests

### Assertion Strategies

#### Asserting on Target Entity (PreOperation)

For PreOperation plugins that modify Target without calling `service.Update()`:

```csharp
[Fact]
public void PreOperation_Update_Should_Modify_Target()
{
    // ARRANGE
    var account = new Account { Id = Guid.NewGuid(), Name = "old name" };
    _context.Initialize(new[] { account });
    
    var updateTarget = new Account { Id = account.Id, Name = "new name" };
    var pluginContext = _context.GetDefaultPluginContext();
    pluginContext.MessageName = "Update";
    pluginContext.Stage = 20; // PreOperation
    pluginContext.InputParameters["Target"] = updateTarget;
    
    // ACT
    _context.ExecutePluginWith<ValidationPlugin>(pluginContext);
    
    // ASSERT - Check the Target reference
    Assert.True(updateTarget.Attributes.ContainsKey("description"));
    Assert.Equal("Auto-generated", updateTarget.Description);
}
```

**Critical:** Assert against the same object reference you passed as Target.

#### Asserting on Database State (PostOperation)

For PostOperation plugins that create/update other records:

```csharp
[Fact]
public void PostOperation_Create_Should_Create_Related_Task()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var target = new Entity("account") { Id = accountId };
    target["name"] = "Contoso";
    
    var pluginContext = _context.GetDefaultPluginContext();
    pluginContext.MessageName = "Create";
    pluginContext.Stage = 40; // PostOperation
    pluginContext.InputParameters["Target"] = target;
    pluginContext.OutputParameters["id"] = accountId;
    
    // ACT
    _context.ExecutePluginWith<FollowupTaskPlugin>(pluginContext);
    
    // ASSERT - Query the database
    var tasks = _context.CreateQuery<Task>()
        .Where(t => t.RegardingObjectId.Id == accountId)
        .ToList();
    
    Assert.Single(tasks);
    Assert.Equal("Follow up with new account", tasks[0].Subject);
}
```

#### Asserting Exceptions

Test that plugins throw the correct exceptions:

```csharp
[Fact]
public void When_Email_Invalid_Should_Throw_Exception()
{
    // ARRANGE
    var contact = new Contact 
    { 
        Id = Guid.NewGuid(), 
        EmailAddress1 = "not-an-email" 
    };
    
    var pluginContext = _context.GetDefaultPluginContext();
    pluginContext.MessageName = "Create";
    pluginContext.InputParameters["Target"] = contact;
    
    // ACT & ASSERT
    var ex = Assert.Throws<InvalidPluginExecutionException>(() =>
    {
        _context.ExecutePluginWith<EmailValidationPlugin>(pluginContext);
    });
    
    Assert.Contains("valid email address", ex.Message);
}
```

### Testing Different Messages

#### Create Message

```csharp
[Fact]
public void Create_Message_Test()
{
    var target = new Account { Id = Guid.NewGuid(), Name = "Contoso" };
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Create";
    ctx.Stage = 40; // PostOperation
    ctx.InputParameters["Target"] = target;
    ctx.OutputParameters["id"] = target.Id;
    
    _context.ExecutePluginWith<MyPlugin>(ctx);
}
```

#### Update Message

```csharp
[Fact]
public void Update_Message_Test()
{
    // ARRANGE: Create existing record
    var accountId = Guid.NewGuid();
    var existing = new Account { Id = accountId, Name = "Old Name" };
    _context.Initialize(new[] { existing });
    
    // Create update target
    var target = new Account { Id = accountId, Name = "New Name" };
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Update";
    ctx.Stage = 20; // PreOperation
    ctx.InputParameters["Target"] = target;
    
    _context.ExecutePluginWith<MyPlugin>(ctx);
}
```

#### Delete Message

```csharp
[Fact]
public void Delete_Message_Test()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    var targetRef = account.ToEntityReference();
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Delete";
    ctx.Stage = 40; // PostOperation
    ctx.InputParameters["Target"] = targetRef;
    
    _context.ExecutePluginWith<MyPlugin>(ctx);
}
```

#### Associate/Disassociate Messages

```csharp
[Fact]
public void Associate_Message_Test()
{
    // ARRANGE
    var userId = Guid.NewGuid();
    var roleId = Guid.NewGuid();
    
    _context.AddRelationship("systemuserroles", new XrmFakedRelationship
    {
        IntersectEntity = "systemuserroles",
        Entity1LogicalName = "systemuser",
        Entity1Attribute = "systemuserid",
        Entity2LogicalName = "role",
        Entity2Attribute = "roleid"
    });
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Associate";
    ctx.InputParameters["Target"] = new EntityReference("systemuser", userId);
    ctx.InputParameters["Relationship"] = new Relationship("systemuserroles");
    ctx.InputParameters["RelatedEntities"] = new EntityReferenceCollection 
    { 
        new EntityReference("role", roleId) 
    };
    
    _context.ExecutePluginWith<MyPlugin>(ctx);
}
```

---

## Pipeline Simulation

Pipeline Simulation automatically executes registered plugins when service operations are performed, mimicking the real Dataverse execution pipeline.

### When to Use Pipeline Simulation

- Testing multiple plugins that fire in sequence
- Verifying plugin registration configurations (stage, mode, filtering attributes)
- Testing plugins that trigger other plugins
- Validating entity images (PreImage/PostImage)
- Simulating realistic pre/post operation behavior

### Setup Pipeline Simulation

#### Middleware Configuration

```csharp
public class PipelineTestsBase
{
    protected readonly IXrmFakedContext _context;
    protected readonly IOrganizationService _service;
    
    public PipelineTestsBase()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .AddFakeMessageExecutors()
            .AddPipelineSimulation()      // Add pipeline simulation
            
            .UsePipelineSimulation()      // Must be FIRST in Use* chain
            .UseCrud()
            .UseMessages()
            
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();
        
        _service = _context.GetOrganizationService();
    }
}
```

**Critical:** `UsePipelineSimulation()` must be called **before** other `Use*` methods.

### Registering Plugin Steps

#### Basic Registration

```csharp
[Fact]
public void Should_Auto_Number_Account_On_Create()
{
    // ARRANGE: Register the plugin
    _context.RegisterPluginStep<AccountNumberPlugin>(
        "Create", 
        ProcessingStepStage.Preoperation
    );
    
    // ACT: Normal service operation - plugin fires automatically
    var accountId = _service.Create(new Account { Name = "Contoso" });
    
    // ASSERT: Check database state
    var account = _service.Retrieve("account", accountId, new ColumnSet(true)) as Account;
    Assert.NotNull(account.AccountNumber);
    Assert.StartsWith("ACC-", account.AccountNumber);
}
```

#### Entity-Specific Registration

```csharp
// Only fires for Account creates
_context.RegisterPluginStep<AccountNumberPlugin, Account>("Create");

// Explicit entity logical name
_context.RegisterPluginStep<MyPlugin>(new PluginStepDefinition
{
    EntityLogicalName = "account",
    MessageName = "Create",
    Stage = ProcessingStepStage.Preoperation,
    Mode = ProcessingStepMode.Synchronous
});
```

#### All Stages

```csharp
// PreValidation (Stage 10)
_context.RegisterPluginStep<ValidationPlugin, Contact>(
    "Update", 
    ProcessingStepStage.Prevalidation
);

// PreOperation (Stage 20)
_context.RegisterPluginStep<SetDefaultsPlugin, Account>(
    "Create", 
    ProcessingStepStage.Preoperation
);

// PostOperation (Stage 40)
_context.RegisterPluginStep<FollowupPlugin, Account>(
    "Create", 
    ProcessingStepStage.Postoperation
);
```

### Filtering Attributes

Register plugins that only fire when specific attributes change:

```csharp
[Fact]
public void Should_Fire_Only_When_Name_Updated()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Old Name", Revenue = new Money(100000) };
    _context.Initialize(new[] { account });
    
    // Register with filtering attributes
    _context.RegisterPluginStep<NameChangePlugin, Account>(
        "Update",
        ProcessingStepStage.Postoperation,
        ProcessingStepMode.Synchronous,
        filteringAttributes: new string[] { "name" }
    );
    
    // ACT: Update revenue only - plugin should NOT fire
    _service.Update(new Account { Id = accountId, Revenue = new Money(200000) });
    
    // ACT: Update name - plugin SHOULD fire
    _service.Update(new Account { Id = accountId, Name = "New Name" });
}
```

**Multiple Attributes:** Plugin fires if **any** specified attribute is in the update.

```csharp
filteringAttributes: new string[] { "name", "revenue", "industrycode" }
// Fires if name OR revenue OR industrycode is updated
```

### Entity Images

Images are snapshots of the entity before (PreImage) and after (PostImage) the operation.

#### Registering Images

```csharp
[Fact]
public void Should_Access_PreImage_In_Update()
{
    // ARRANGE: Setup existing record
    var accountId = Guid.NewGuid();
    var existing = new Account 
    { 
        Id = accountId, 
        Name = "Old Name",
        Revenue = new Money(100000)
    };
    _context.Initialize(new[] { existing });
    
    // Register plugin with PreImage
    var preImageDefinition = new PluginImageDefinition(
        "PreImage",                         // Image name (used in plugin code)
        ProcessingStepImageType.PreImage    // PreImage = before changes
    );
    
    _context.RegisterPluginStep<AuditChangePlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "Update",
        Stage = ProcessingStepStage.Postoperation,
        RegisteredImages = new[] { preImageDefinition }
    });
    
    // ACT: Update only revenue
    _service.Update(new Account { Id = accountId, Revenue = new Money(200000) });
    
    // Plugin can access:
    // - Target: { revenue = 200000 } (only changed attributes)
    // - PreImage["name"]: "Old Name" (all attributes from before update)
    // - PreImage["revenue"]: 100000
}
```

#### PostImage Example

```csharp
[Fact]
public void Should_Access_PostImage_After_Update()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var existing = new Account { Id = accountId, Name = "Old Name" };
    _context.Initialize(new[] { existing });
    
    // Register with PostImage
    var postImageDef = new PluginImageDefinition(
        "PostImage",
        ProcessingStepImageType.PostImage
    );
    
    _context.RegisterPluginStep<MyPlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "Update",
        Stage = ProcessingStepStage.Postoperation,
        RegisteredImages = new[] { postImageDef }
    });
    
    // ACT
    _service.Update(new Account { Id = accountId, Name = "New Name" });
    
    // Plugin sees PostImage with complete updated record
}
```

#### Both PreImage and PostImage

```csharp
var preImage = new PluginImageDefinition("PreImage", ProcessingStepImageType.PreImage);
var postImage = new PluginImageDefinition("PostImage", ProcessingStepImageType.PostImage);

_context.RegisterPluginStep<MyPlugin>(new PluginStepDefinition
{
    EntityLogicalName = "account",
    MessageName = "Update",
    Stage = ProcessingStepStage.Postoperation,
    RegisteredImages = new[] { preImage, postImage }
});
```

### Multiple Plugins in Sequence

```csharp
[Fact]
public void Should_Execute_Multiple_Plugins_In_Order()
{
    // ARRANGE: Register multiple plugins
    _context.RegisterPluginStep<ValidationPlugin, Account>(
        "Create",
        ProcessingStepStage.Prevalidation,
        executionOrder: 1  // Runs first
    );
    
    _context.RegisterPluginStep<SetDefaultsPlugin, Account>(
        "Create",
        ProcessingStepStage.Preoperation,
        executionOrder: 1
    );
    
    _context.RegisterPluginStep<FollowupPlugin, Account>(
        "Create",
        ProcessingStepStage.Postoperation,
        executionOrder: 1
    );
    
    // ACT: Single service call fires all plugins
    var accountId = _service.Create(new Account { Name = "Contoso" });
    
    // ASSERT: Verify all plugins executed
    var account = _service.Retrieve("account", accountId, new ColumnSet(true));
    var tasks = _context.CreateQuery<Task>()
        .Where(t => t.RegardingObjectId.Id == accountId)
        .ToList();
    
    Assert.NotNull(account.GetAttributeValue<string>("accountnumber"));
    Assert.Single(tasks);
}
```

---

## Testing Queries

Comprehensive guide for testing FetchXML, QueryExpression, and LINQ queries.

### QueryExpression Testing

#### Basic Query

```csharp
[Fact]
public void Should_Query_Active_Accounts()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso", StateCode = AccountState.Active };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam", StateCode = AccountState.Inactive };
    _context.Initialize(new[] { account1, account2 });
    
    // ACT
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name", "statecode");
    query.Criteria.AddCondition("statecode", ConditionOperator.Equal, 0); // Active
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Single(results.Entities);
    Assert.Equal("Contoso", results.Entities[0].GetAttributeValue<string>("name"));
}
```

#### Multiple Conditions (AND)

```csharp
[Fact]
public void Should_Query_With_AND_Conditions()
{
    // ARRANGE
    var contact1 = new Contact { Id = Guid.NewGuid(), FirstName = "John", StateCode = ContactState.Active, Address1_City = "Seattle" };
    var contact2 = new Contact { Id = Guid.NewGuid(), FirstName = "Jane", StateCode = ContactState.Active, Address1_City = "Portland" };
    _context.Initialize(new[] { contact1, contact2 });
    
    // ACT
    var query = new QueryExpression("contact");
    query.ColumnSet = new ColumnSet(true);
    query.Criteria.FilterOperator = LogicalOperator.And;
    query.Criteria.AddCondition("statecode", ConditionOperator.Equal, 0);
    query.Criteria.AddCondition("address1_city", ConditionOperator.Equal, "Seattle");
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Single(results.Entities);
    Assert.Equal("John", results.Entities[0].GetAttributeValue<string>("firstname"));
}
```

#### OR Conditions

```csharp
[Fact]
public void Should_Query_With_OR_Conditions()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso", IndustryCode = new OptionSetValue(1) };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam", IndustryCode = new OptionSetValue(2) };
    var account3 = new Account { Id = Guid.NewGuid(), Name = "Adventure Works", IndustryCode = new OptionSetValue(3) };
    _context.Initialize(new[] { account1, account2, account3 });
    
    // ACT
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name", "industrycode");
    query.Criteria.FilterOperator = LogicalOperator.Or;
    query.Criteria.AddCondition("industrycode", ConditionOperator.Equal, 1);
    query.Criteria.AddCondition("industrycode", ConditionOperator.Equal, 2);
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count);
}
```

#### Nested Filters

```csharp
[Fact]
public void Should_Query_With_Nested_Filters()
{
    // ACT - (Active AND (Seattle OR Portland))
    var query = new QueryExpression("contact");
    query.Criteria.FilterOperator = LogicalOperator.And;
    query.Criteria.AddCondition("statecode", ConditionOperator.Equal, 0);
    
    var cityFilter = new FilterExpression(LogicalOperator.Or);
    cityFilter.AddCondition("address1_city", ConditionOperator.Equal, "Seattle");
    cityFilter.AddCondition("address1_city", ConditionOperator.Equal, "Portland");
    query.Criteria.AddFilter(cityFilter);
    
    var results = _service.RetrieveMultiple(query);
}
```

#### Comparison Operators

```csharp
[Theory]
[InlineData(ConditionOperator.GreaterThan, 100, 1)]
[InlineData(ConditionOperator.LessThan, 100, 1)]
[InlineData(ConditionOperator.GreaterEqual, 150, 1)]
[InlineData(ConditionOperator.Between, null, 2)]
public void Should_Query_With_Numeric_Operators(ConditionOperator op, int? value, int expectedCount)
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), NumberOfEmployees = 50 };
    var account2 = new Account { Id = Guid.NewGuid(), NumberOfEmployees = 150 };
    _context.Initialize(new[] { account1, account2 });
    
    // ACT
    var query = new QueryExpression("account");
    if (op == ConditionOperator.Between)
        query.Criteria.AddCondition("numberofemployees", op, 75, 175);
    else
        query.Criteria.AddCondition("numberofemployees", op, value);
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(expectedCount, results.Entities.Count);
}
```

### FetchXML Testing

```csharp
[Fact]
public void Should_Execute_FetchXML_Query()
{
    // ARRANGE
    var account = new Account { Id = Guid.NewGuid(), Name = "Contoso", StateCode = AccountState.Active };
    _context.Initialize(new[] { account });
    
    // ACT
    var fetchXml = @"
        <fetch>
          <entity name='account'>
            <attribute name='name'/>
            <attribute name='statecode'/>
            <filter type='and'>
              <condition attribute='statecode' operator='eq' value='0'/>
            </filter>
          </entity>
        </fetch>";
    
    var results = _service.RetrieveMultiple(new FetchExpression(fetchXml));
    
    // ASSERT
    Assert.Single(results.Entities);
    Assert.Equal("Contoso", results.Entities[0].GetAttributeValue<string>("name"));
}
```

### LINQ Query Testing

```csharp
[Fact]
public void Should_Query_Using_LINQ()
{
    // ARRANGE
    _context.EnableProxyTypes(Assembly.GetExecutingAssembly());
    
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso", NumberOfEmployees = 100 };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam", NumberOfEmployees = 200 };
    _context.Initialize(new[] { account1, account2 });
    
    // ACT
    var results = _context.CreateQuery<Account>()
        .Where(a => a.NumberOfEmployees > 150)
        .ToList();
    
    // ASSERT
    Assert.Single(results);
    Assert.Equal("Fabrikam", results[0].Name);
}
```

### QueryByAttribute Testing

```csharp
[Fact]
public void Should_Query_By_Attribute()
{
    // ARRANGE
    var account = new Account { Id = Guid.NewGuid(), Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    // ACT
    var query = new QueryByAttribute("account");
    query.ColumnSet = new ColumnSet("name");
    query.Attributes.Add("name");
    query.Values.Add("Contoso");
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Single(results.Entities);
}
```

---

## Testing Relationships

### Entity References (Lookups)

#### Creating Records with Lookups

```csharp
[Fact]
public void Should_Create_Contact_With_Account_Reference()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    var contact = new Contact
    {
        FirstName = "John",
        LastName = "Doe",
        ParentCustomerId = account.ToEntityReference()
    };
    
    // ACT
    var contactId = _service.Create(contact);
    
    // ASSERT
    var created = _service.Retrieve("contact", contactId, new ColumnSet(true))
        .ToEntity<Contact>();
    
    Assert.NotNull(created.ParentCustomerId);
    Assert.Equal(accountId, created.ParentCustomerId.Id);
}
```

#### Updating References

```csharp
[Fact]
public void Should_Update_Lookup_Field()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso" };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam" };
    var contact = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "John",
        ParentCustomerId = account1.ToEntityReference()
    };
    _context.Initialize(new Entity[] { account1, account2, contact });
    
    // ACT
    var contactUpdate = new Contact
    {
        Id = contact.Id,
        ParentCustomerId = account2.ToEntityReference()
    };
    _service.Update(contactUpdate);
    
    // ASSERT
    var updated = _service.Retrieve("contact", contact.Id, new ColumnSet("parentcustomerid"))
        .ToEntity<Contact>();
    Assert.Equal(account2.Id, updated.ParentCustomerId.Id);
}
```

### One-to-Many Relationships

```csharp
[Fact]
public void Should_Query_Account_With_Related_Contacts()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    
    var contact1 = new Contact { Id = Guid.NewGuid(), FirstName = "John", ParentCustomerId = account.ToEntityReference() };
    var contact2 = new Contact { Id = Guid.NewGuid(), FirstName = "Jane", ParentCustomerId = account.ToEntityReference() };
    var contact3 = new Contact { Id = Guid.NewGuid(), FirstName = "Bob", ParentCustomerId = new EntityReference("account", Guid.NewGuid()) };
    
    _context.Initialize(new Entity[] { account, contact1, contact2, contact3 });
    
    // ACT
    var query = new QueryExpression("contact");
    query.ColumnSet = new ColumnSet("firstname");
    query.Criteria.AddCondition("parentcustomerid", ConditionOperator.Equal, accountId);
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count);
}
```

### Many-to-Many Relationships

#### Setup N:N Relationship

```csharp
[Fact]
public void Should_Associate_Many_To_Many()
{
    // ARRANGE
    var userId = Guid.NewGuid();
    var role1Id = Guid.NewGuid();
    var role2Id = Guid.NewGuid();
    
    var user = new Entity("systemuser") { Id = userId };
    var role1 = new Entity("role") { Id = role1Id };
    var role2 = new Entity("role") { Id = role2Id };
    
    _context.Initialize(new[] { user, role1, role2 });
    
    // Define N:N relationship
    _context.AddRelationship("systemuserroles_association", new XrmFakedRelationship
    {
        IntersectEntity = "systemuserroles",
        Entity1LogicalName = "systemuser",
        Entity1Attribute = "systemuserid",
        Entity2LogicalName = "role",
        Entity2Attribute = "roleid"
    });
    
    // ACT
    _service.Associate(
        "systemuser",
        userId,
        new Relationship("systemuserroles_association"),
        new EntityReferenceCollection 
        { 
            new EntityReference("role", role1Id),
            new EntityReference("role", role2Id)
        }
    );
    
    // ASSERT
    var associations = _context.CreateQuery("systemuserroles").ToList();
    Assert.Equal(2, associations.Count);
}
```

#### Disassociate Test

```csharp
[Fact]
public void Should_Disassociate_Many_To_Many()
{
    // ARRANGE - Setup existing association
    var userId = Guid.NewGuid();
    var roleId = Guid.NewGuid();
    
    _context.AddRelationship("systemuserroles_association", new XrmFakedRelationship
    {
        IntersectEntity = "systemuserroles",
        Entity1LogicalName = "systemuser",
        Entity1Attribute = "systemuserid",
        Entity2LogicalName = "role",
        Entity2Attribute = "roleid"
    });
    
    _service.Associate("systemuser", userId, new Relationship("systemuserroles_association"),
        new EntityReferenceCollection { new EntityReference("role", roleId) });
    
    // ACT
    _service.Disassociate("systemuser", userId, new Relationship("systemuserroles_association"),
        new EntityReferenceCollection { new EntityReference("role", roleId) });
    
    // ASSERT
    var associations = _context.CreateQuery("systemuserroles").ToList();
    Assert.Empty(associations);
}
```

---

## Custom Messages & APIs

### Custom Actions Testing

#### Simple Custom Action

```csharp
public class SendWelcomeEmailPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var serviceFactory = (IOrganizationServiceFactory)serviceProvider.GetService(typeof(IOrganizationServiceFactory));
        var service = serviceFactory.CreateOrganizationService(context.UserId);
        
        var target = (EntityReference)context.InputParameters["Target"];
        var contact = service.Retrieve(target.LogicalName, target.Id, new ColumnSet("emailaddress1"));
        var email = contact.GetAttributeValue<string>("emailaddress1");
        
        bool success = !string.IsNullOrEmpty(email);
        context.OutputParameters["Success"] = success;
    }
}

[Fact]
public void Should_Execute_Custom_Action()
{
    // ARRANGE
    var contactId = Guid.NewGuid();
    var contact = new Contact { Id = contactId, EMailAddress1 = "john@example.com" };
    _context.Initialize(new[] { contact });
    
    _context.RegisterPluginStep<SendWelcomeEmailPlugin>(
        "new_SendWelcomeEmail",
        executionStage: ProcessingStepStage.Postoperation
    );
    
    // ACT
    var request = new OrganizationRequest("new_SendWelcomeEmail");
    request["Target"] = contact.ToEntityReference();
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.True((bool)response["Success"]);
}
```

#### Custom Action with Parameters

```csharp
[Theory]
[InlineData(1000.00, "VIP", 200.00, 800.00)]
[InlineData(1000.00, "Gold", 150.00, 850.00)]
public void Should_Calculate_Discount(decimal baseAmount, string type, decimal expectedDiscount, decimal expectedFinal)
{
    // ARRANGE
    _context.RegisterPluginStep<CalculateDiscountPlugin>("new_CalculateDiscount");
    
    // ACT
    var request = new OrganizationRequest("new_CalculateDiscount");
    request["BaseAmount"] = baseAmount;
    request["CustomerType"] = type;
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.Equal(expectedDiscount, (decimal)response["DiscountAmount"]);
    Assert.Equal(expectedFinal, (decimal)response["FinalAmount"]);
}
```

### Custom APIs Testing

```csharp
[Fact]
public void Should_Execute_Custom_API()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    _context.RegisterPluginStep<MyCustomAPIPlugin>("new_MyCustomAPI");
    
    // ACT
    var request = new OrganizationRequest("new_MyCustomAPI");
    request["InputParameter"] = "test value";
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.NotNull(response["OutputParameter"]);
}
```

---

## Advanced Scenarios

### Dependency Injection

#### Constructor Injection Pattern

```csharp
public class NotificationPlugin : IPlugin
{
    private readonly IEmailService _emailService;
    private readonly ILogger _logger;
    
    // Production constructor
    public NotificationPlugin() : this(new EmailService(), new Logger()) { }
    
    // Test constructor
    public NotificationPlugin(IEmailService emailService, ILogger logger)
    {
        _emailService = emailService;
        _logger = logger;
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        _logger.Log("Plugin executing");
        _emailService.SendEmail("test@example.com", "Subject", "Body");
    }
}

[Fact]
public void Should_Send_Email_With_Mocked_Service()
{
    // ARRANGE
    var mockEmail = new Mock<IEmailService>();
    var mockLogger = new Mock<ILogger>();
    var plugin = new NotificationPlugin(mockEmail.Object, mockLogger.Object);
    
    _context.RegisterPluginStep<NotificationPlugin>(new PluginStepDefinition
    {
        MessageName = "Create",
        EntityLogicalName = "account",
        PluginInstance = plugin  // Use our instance with mocks
    });
    
    // ACT
    _service.Create(new Account { Name = "Contoso" });
    
    // ASSERT
    mockEmail.Verify(e => e.SendEmail(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()), Times.Once);
}
```

### Mocking External Services

#### HTTP Client Mock

```csharp
public interface IHttpClientWrapper
{
    Task<string> PostAsync(string url, string content);
}

[Fact]
public void Should_Call_External_API()
{
    // ARRANGE
    var mockHttp = new Mock<IHttpClientWrapper>();
    mockHttp.Setup(h => h.PostAsync(It.IsAny<string>(), It.IsAny<string>()))
        .ReturnsAsync("{\"status\":\"success\"}");
    
    var plugin = new ExternalApiPlugin(mockHttp.Object);
    
    _context.RegisterPluginStep<ExternalApiPlugin>(new PluginStepDefinition
    {
        MessageName = "Create",
        PluginInstance = plugin
    });
    
    // ACT
    _service.Create(new Account { Name = "Contoso" });
    
    // ASSERT
    mockHttp.Verify(h => h.PostAsync(It.IsAny<string>(), It.IsAny<string>()), Times.Once);
}
```

---

## Async Testing

**Version Required:** FakeXrmEasy v3.x only (.NET Core 3.1+)

### IOrganizationServiceAsync

```csharp
[Fact]
public async Task Should_Create_Account_Asynchronously()
{
    // ARRANGE
    var service = _context.GetAsyncOrganizationService();
    var account = new Account { Name = "Contoso" };
    
    // ACT
    var accountId = await service.CreateAsync(account);
    
    // ASSERT
    Assert.NotEqual(Guid.Empty, accountId);
    var created = await service.RetrieveAsync("account", accountId, new ColumnSet(true));
    Assert.Equal("Contoso", created.GetAttributeValue<string>("name"));
}
```

### IOrganizationServiceAsync2

```csharp
[Fact]
public async Task Should_Query_Asynchronously()
{
    // ARRANGE
    var account = new Account { Id = Guid.NewGuid(), Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    var service = _context.GetAsyncOrganizationService2();
    
    // ACT
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name");
    var results = await service.RetrieveMultipleAsync(query);
    
    // ASSERT
    Assert.Single(results.Entities);
}
```

---

## Bulk Operations

**Minimum Version:** FakeXrmEasy 2.5.0+ / 3.5.0+

### CreateMultiple

```csharp
[Fact]
public void Should_Create_Multiple_Accounts()
{
    // ARRANGE
    var accounts = new[]
    {
        new Account { Name = "Contoso" },
        new Account { Name = "Fabrikam" },
        new Account { Name = "Adventure Works" }
    };
    
    var request = new CreateMultipleRequest
    {
        Targets = new EntityCollection(accounts)
    };
    
    // ACT
    var response = (CreateMultipleResponse)_service.Execute(request);
    
    // ASSERT
    Assert.Equal(3, response.Ids.Count);
    var created = _context.CreateQuery<Account>().ToList();
    Assert.Equal(3, created.Count);
}
```

### UpdateMultiple

```csharp
[Fact]
public void Should_Update_Multiple_Accounts()
{
    // ARRANGE
    var account1Id = Guid.NewGuid();
    var account2Id = Guid.NewGuid();
    
    _context.Initialize(new[]
    {
        new Account { Id = account1Id, Name = "Old 1", Revenue = new Money(100000) },
        new Account { Id = account2Id, Name = "Old 2", Revenue = new Money(200000) }
    });
    
    var updates = new[]
    {
        new Account { Id = account1Id, Revenue = new Money(150000) },
        new Account { Id = account2Id, Revenue = new Money(250000) }
    };
    
    var request = new UpdateMultipleRequest
    {
        Targets = new EntityCollection(updates)
    };
    
    // ACT
    _service.Execute(request);
    
    // ASSERT
    var updated = _context.CreateQuery<Account>().ToList();
    Assert.Equal(150000m, updated.First(a => a.Id == account1Id).Revenue.Value);
    Assert.Equal(250000m, updated.First(a => a.Id == account2Id).Revenue.Value);
}
```

### UpsertMultiple

```csharp
[Fact]
public void Should_Upsert_Multiple_Records()
{
    // ARRANGE
    var existingId = Guid.NewGuid();
    _context.Initialize(new[] { new Account { Id = existingId, Name = "Existing" } });
    
    var records = new[]
    {
        new Account { Id = existingId, Name = "Updated" },  // Update
        new Account { Name = "New" }                        // Create
    };
    
    var request = new UpsertMultipleRequest
    {
        Targets = new EntityCollection(records)
    };
    
    // ACT
    var response = (UpsertMultipleResponse)_service.Execute(request);
    
    // ASSERT
    Assert.Equal(2, response.Results.Count);
    Assert.False(response.Results[0].RecordCreated); // Updated
    Assert.True(response.Results[1].RecordCreated);  // Created
}
```

---

## CodeActivities Testing

**Package Required:** `FakeXrmEasy.CodeActivities.v9` v2.x

### Basic CodeActivity Test

```csharp
public class CalculateTaxActivity : CodeActivity
{
    [RequiredArgument]
    [Input("Amount")]
    public InArgument<decimal> Amount { get; set; }
    
    [Input("Tax Rate")]
    [Default("0.20")]
    public InArgument<decimal> TaxRate { get; set; }
    
    [Output("Tax Amount")]
    public OutArgument<decimal> TaxAmount { get; set; }
    
    protected override void Execute(CodeActivityContext context)
    {
        var amount = Amount.Get(context);
        var taxRate = TaxRate.Get(context);
        TaxAmount.Set(context, amount * taxRate);
    }
}

[Fact]
public void Should_Calculate_Tax()
{
    // ARRANGE
    var inputs = new Dictionary<string, object>
    {
        { "Amount", 100.00m },
        { "Tax Rate", 0.20m }
    };
    
    var activity = new CalculateTaxActivity();
    
    // ACT
    var outputs = _context.ExecuteCodeActivity<CalculateTaxActivity>(activity, inputs);
    
    // ASSERT
    Assert.Equal(20.00m, outputs["Tax Amount"]);
}
```

---

## Azure Functions Testing

**Recommended Version:** FakeXrmEasy v3.x

### HTTP Trigger Function Test

```csharp
[Fact]
public async Task Should_Create_Account_From_HTTP_Request()
{
    // ARRANGE
    var mockServiceFactory = new Mock<IOrganizationServiceFactory>();
    mockServiceFactory.Setup(f => f.CreateOrganizationService(It.IsAny<Guid?>()))
        .Returns(_service);
    
    var function = new CreateAccountFunction(mockServiceFactory.Object, Mock.Of<ILogger>());
    
    var mockRequest = new Mock<HttpRequestData>(Mock.Of<FunctionContext>());
    var requestBody = JsonSerializer.Serialize(new { Name = "Contoso", Phone = "555-1234" });
    mockRequest.Setup(r => r.ReadAsStringAsync()).ReturnsAsync(requestBody);
    
    // ACT
    var response = await function.Run(mockRequest.Object);
    
    // ASSERT
    Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    var accounts = _context.CreateQuery<Account>().ToList();
    Assert.Single(accounts);
    Assert.Equal("Contoso", accounts[0].Name);
}
```

---

## File Storage Testing

**Minimum Version:** FakeXrmEasy 2.6.0+ / 3.6.0+

### File Upload Test

```csharp
[Fact]
public void Should_Upload_File()
{
    // ARRANGE
    var documentId = Guid.NewGuid();
    var document = new Entity("dv_document") { Id = documentId };
    _context.Initialize(new[] { document });
    
    var fileContent = Encoding.UTF8.GetBytes("Test file content");
    var fileBase64 = Convert.ToBase64String(fileContent);
    
    // ACT: Initialize upload
    var initRequest = new InitializeFileBlocksUploadRequest
    {
        Target = new EntityReference("dv_document", documentId),
        FileAttributeName = "dv_file",
        FileName = "test.txt"
    };
    var initResponse = (InitializeFileBlocksUploadResponse)_service.Execute(initRequest);
    
    // Upload block
    var uploadRequest = new UploadBlockRequest
    {
        BlockId = Convert.ToBase64String(Encoding.UTF8.GetBytes("block1")),
        BlockData = fileBase64,
        FileContinuationToken = initResponse.FileContinuationToken
    };
    _service.Execute(uploadRequest);
    
    // Commit
    var commitRequest = new CommitFileBlocksUploadRequest
    {
        BlockList = new[] { uploadRequest.BlockId },
        FileContinuationToken = initResponse.FileContinuationToken,
        FileName = "test.txt",
        MimeType = "text/plain"
    };
    var commitResponse = (CommitFileBlocksUploadResponse)_service.Execute(commitRequest);
    
    // ASSERT
    Assert.NotNull(commitResponse.FileId);
}
```

### File Download Test

```csharp
[Fact]
public void Should_Download_File()
{
    // ARRANGE: Upload file first
    var fileId = UploadTestFile();
    
    // ACT
    var downloadRequest = new DownloadFileRequest
    {
        FileId = fileId
    };
    var downloadResponse = (DownloadFileResponse)_service.Execute(downloadRequest);
    
    // ASSERT
    Assert.NotNull(downloadResponse.Data);
    var content = Convert.FromBase64String(downloadResponse.Data);
    Assert.Equal("Test file content", Encoding.UTF8.GetString(content));
}
```

---

## Logging & Telemetry

**Minimum Version:** FakeXrmEasy 2.6.0+ / 3.6.0+

### ILogger Testing

```csharp
public class AccountPlugin : IPlugin
{
    private readonly ILogger<AccountPlugin> _logger;
    
    public AccountPlugin(ILogger<AccountPlugin> logger = null)
    {
        _logger = logger;
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        _logger?.LogInformation("Plugin executing for {EntityName}", context.PrimaryEntityName);
    }
}

[Fact]
public void Should_Log_Information()
{
    // ARRANGE
    var mockLogger = new Mock<ILogger<AccountPlugin>>();
    var plugin = new AccountPlugin(mockLogger.Object);
    
    _context.RegisterPluginStep<AccountPlugin>(new PluginStepDefinition
    {
        MessageName = "Create",
        PluginInstance = plugin
    });
    
    // ACT
    _service.Create(new Account { Name = "Contoso" });
    
    // ASSERT
    mockLogger.Verify(
        logger => logger.Log(
            LogLevel.Information,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((v, t) => v.ToString().Contains("Plugin executing")),
            null,
            It.IsAny<Func<It.IsAnyType, Exception, string>>()
        ),
        Times.Once
    );
}
```

### Structured Logging Verification

```csharp
[Fact]
public void Should_Log_With_Structured_Data()
{
    // ARRANGE
    var mockLogger = new Mock<ILogger<ContactPlugin>>();
    var plugin = new ContactPlugin(mockLogger.Object);
    
    // ACT
    _context.ExecutePluginWith<ContactPlugin>(plugin, new Contact { FirstName = "John", LastName = "Doe" });
    
    // ASSERT
    mockLogger.Verify(
        logger => logger.Log(
            LogLevel.Information,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((v, t) => v.ToString().Contains("John") && v.ToString().Contains("Doe")),
            null,
            It.IsAny<Func<It.IsAnyType, Exception, string>>()
        )
    );
}
```

---

## Migration Guide

### v1.x to v2.x/v3.x Migration

#### Step 1: Update Packages

```powershell
# Remove v1.x
Uninstall-Package FakeXrmEasy

# Install v2.x (Framework) or v3.x (Core)
Install-Package FakeXrmEasy.v9 -Version 2.6.3
# OR
Install-Package FakeXrmEasy.v9 -Version 3.6.3
```

#### Step 2: Update Namespaces

**Before (v1.x):**
```csharp
using FakeXrmEasy;
```

**After (v2.x/v3.x):**
```csharp
using FakeXrmEasy;
using FakeXrmEasy.Abstractions;
using FakeXrmEasy.Middleware;
using FakeXrmEasy.Plugins;
```

#### Step 3: Update Context Initialization

**Before (v1.x):**
```csharp
var context = new XrmFakedContext();
var service = context.GetOrganizationService();
```

**After (v2.x/v3.x):**
```csharp
var context = MiddlewareBuilder.New()
    .AddCrud()
    .UseCrud()
    .SetLicense(FakeXrmEasyLicense.RPL_1_5)
    .Build();
var service = context.GetOrganizationService();
```

#### Step 4: Update Plugin Execution

**Before (v1.x):**
```csharp
context.ExecutePluginWith<MyPlugin>(target);
```

**After (v2.x/v3.x):**
```csharp
context.ExecutePluginWith<MyPlugin>(
    target,
    messageName: "Create",
    stage: ProcessingStepStage.Postoperation
);
```

### Breaking Changes

- Context initialization requires middleware builder
- License must be set (RPL_1_5 for non-commercial)
- Plugin execution requires explicit message/stage
- Some message executors require separate packages
- Assembly references changed to FakeXrmEasy.Abstractions

---

## Best Practices

### 1. Use AAA Pattern Consistently

```csharp
[Fact]
public void Test_Name_Should_Describe_Behavior()
{
    // ARRANGE: Setup
    var account = new Account { Name = "Contoso" };
    
    // ACT: Execute
    var result = DoSomething(account);
    
    // ASSERT: Verify
    Assert.NotNull(result);
}
```

### 2. Test One Thing Per Test

❌ **Don't:**
```csharp
[Fact]
public void Should_Do_Everything()
{
    // Creates account
    // Updates contact
    // Validates email
    // Sends notification
    // Too much in one test!
}
```

✅ **Do:**
```csharp
[Fact]
public void Should_Create_Account() { /* Test account creation */ }

[Fact]
public void Should_Update_Contact() { /* Test contact update */ }

[Fact]
public void Should_Validate_Email() { /* Test validation */ }
```

### 3. Use Descriptive Test Names

✅ **Good:**
```csharp
[Fact]
public void When_Account_Created_Without_Email_Should_Throw_ValidationException()
```

❌ **Bad:**
```csharp
[Fact]
public void Test1()
```

### 4. Mock External Dependencies

```csharp
// Mock HTTP client, file system, external APIs
var mockHttp = new Mock<IHttpClient>();
mockHttp.Setup(h => h.GetAsync(It.IsAny<string>()))
    .ReturnsAsync("mocked response");

var plugin = new MyPlugin(mockHttp.Object);
```

### 5. Test Negative Cases

```csharp
[Fact]
public void When_Required_Field_Missing_Should_Throw_Exception()
{
    var contact = new Contact(); // Missing required fields
    
    Assert.Throws<InvalidPluginExecutionException>(() =>
    {
        _context.ExecutePluginWith<ValidationPlugin>(contact);
    });
}
```

### 6. Use Theory for Multiple Scenarios

```csharp
[Theory]
[InlineData(100, 0.20, 120)]
[InlineData(200, 0.15, 230)]
[InlineData(300, 0.10, 330)]
public void Should_Calculate_Total_With_Tax(decimal amount, decimal rate, decimal expected)
{
    var result = CalculateTotal(amount, rate);
    Assert.Equal(expected, result);
}
```

### 7. Create Test Helpers

```csharp
protected Account CreateTestAccount(string name, decimal? revenue = null)
{
    var account = new Account 
    { 
        Id = Guid.NewGuid(), 
        Name = name,
        Revenue = revenue.HasValue ? new Money(revenue.Value) : null
    };
    _context.Initialize(new[] { account });
    return account;
}
```

### 8. Verify Exception Messages

```csharp
var ex = Assert.Throws<InvalidPluginExecutionException>(() => ExecutePlugin());
Assert.Contains("Email address is required", ex.Message);
```

### 9. Use Pipeline Simulation Sparingly

- Use for integration-style tests
- Not needed for simple unit tests
- Adds complexity - only when testing plugin interactions

### 10. Keep Tests Fast

- Don't connect to external services
- Don't use `Thread.Sleep()`
- Mock everything external
- Target < 100ms per test

---

## Troubleshooting

### Common Issues

#### Issue: "License not set" error

**Solution:**
```csharp
.SetLicense(FakeXrmEasyLicense.RPL_1_5)
```

#### Issue: Message executor not found

**Solution:**
```csharp
.AddFakeMessageExecutors(typeof(MessageExecutor).Assembly)
```

#### Issue: Pipeline not firing

**Solution:**
```csharp
.UsePipelineSimulation()  // Must be FIRST in Use* chain
.UseCrud()
```

#### Issue: PreOperation changes not visible

**Solution:**
```csharp
// Assert against Target reference, not database
var target = new Account();
_context.ExecutePluginWith<MyPlugin>(target);
Assert.NotNull(target.AccountNumber); // Check target, not DB
```

#### Issue: Query returns no results

**Solution:**
```csharp
// Initialize data first
_context.Initialize(new[] { account });

// Then query
var results = _context.CreateQuery<Account>().ToList();
```

### Debugging Tips

1. **Enable early-bound types:**
```csharp
_context.EnableProxyTypes(Assembly.GetExecutingAssembly());
```

2. **Check in-memory data:**
```csharp
var allAccounts = _context.CreateQuery<Account>().ToList();
Console.WriteLine($"Count: {allAccounts.Count}");
```

3. **Verify plugin registration:**
```csharp
// Ensure RegisterPluginStep called before service operation
_context.RegisterPluginStep<MyPlugin>("Create");
_service.Create(account); // Plugin fires here
```

4. **Use test output:**
```csharp
public class MyTests : ITestOutputHelper
{
    private readonly ITestOutputHelper _output;
    
    public MyTests(ITestOutputHelper output) => _output = output;
    
    [Fact]
    public void Test()
    {
        _output.WriteLine("Debug info");
    }
}
```

---

## Additional Resources

- **Official FakeXrmEasy Docs:** https://dynamicsvalue.github.io/fake-xrm-easy-docs/
- **GitHub Repository:** https://github.com/DynamicsValue/fake-xrm-easy-core
- **NuGet Package:** https://www.nuget.org/packages/FakeXrmEasy.v9
- **Code Samples:** https://github.com/DynamicsValue/fake-xrm-easy-samples

---

## License

This documentation is licensed under the MIT License.

FakeXrmEasy framework uses Reciprocal Public License (RPL) 1.5 for non-commercial use. Commercial licenses available from DynamicsValue.

---

**Last Updated:** April 2026  
**Skill Version:** 1.0.0  
**Framework Version:** FakeXrmEasy 2.6.3+ / 3.6.3+

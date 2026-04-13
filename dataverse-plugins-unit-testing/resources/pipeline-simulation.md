# Pipeline Simulation

Pipeline Simulation is an advanced FakeXrmEasy feature that automatically executes registered
plugins when service operations are performed, mimicking the real Dataverse execution pipeline.

**Use Pipeline Simulation when:**
- Testing multiple plugins that fire in sequence
- Verifying plugin registration configurations (stage, mode, filtering attributes)
- Testing plugins that trigger other plugins
- Validating entity images (PreImage/PostImage)
- Simulating realistic pre/post operation behavior

## Setup Pipeline Simulation

### Middleware Configuration

Enable pipeline simulation in your test base class:

```csharp
using FakeXrmEasy;
using FakeXrmEasy.Middleware;
using FakeXrmEasy.Plugins;

public class PipelineTestsBase
{
    protected readonly IXrmFakedContext _context;
    protected readonly IOrganizationService _service;
    
    public PipelineTestsBase()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()                    // Enable Create/Read/Update/Delete
            .AddFakeMessageExecutors()    // Enable standard messages
            .AddPipelineSimulation()      // Add pipeline simulation capability
            
            .UsePipelineSimulation()      // Must be FIRST in Use* chain
            .UseCrud()
            .UseMessages()
            
            .Build();
        
        _service = _context.GetAsyncOrganizationService();
    }
}
```

**Critical:** `UsePipelineSimulation()` must be called **before** other `Use*` methods. This ensures
PreOperation plugins execute before the actual database changes.

## Registering Plugin Steps

### Basic Registration

Register a plugin step with message and stage:

```csharp
[Fact]
public void Should_Auto_Number_Account_On_Create()
{
    // ARRANGE: Register the plugin
    _context.RegisterPluginStep<AccountNumberPlugin>("Create", ProcessingStepStage.Preoperation);
    
    // ACT: Normal service operation - plugin fires automatically
    var accountId = _service.Create(new Account { Name = "Contoso" });
    
    // ASSERT: Check database state
    var account = _service.Retrieve("account", accountId, new ColumnSet(true)) as Account;
    Assert.NotNull(account.AccountNumber);
    Assert.StartsWith("ACC-", account.AccountNumber);
}
```

### Entity-Specific Registration

Register plugin for a specific entity type:

```csharp
// Only fires for Account creates, not Contact creates
_context.RegisterPluginStep<AccountNumberPlugin, Account>("Create");

// Explicitly specify entity logical name
_context.RegisterPluginStep<MyPlugin>(new PluginStepDefinition
{
    EntityLogicalName = "account",
    MessageName = "Create",
    Stage = ProcessingStepStage.Preoperation,
    Mode = ProcessingStepMode.Synchronous
});
```

### All Stages

```csharp
// PreValidation (Stage 10)
_context.RegisterPluginStep<ValidationPlugin, Contact>(
    "Update", 
    ProcessingStepStage.Prevalidation, 
    ProcessingStepMode.Synchronous
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

**Default:** If stage/mode not specified, defaults to PostOperation, Synchronous.

## Filtering Attributes

Register plugins that only fire when specific attributes change:

```csharp
[Fact]
public void Should_Fire_Only_When_Name_Updated()
{
    // ARRANGE: Initialize existing account
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
    
    // Plugin didn't execute (verify with side effects)
    
    // ACT: Update name - plugin SHOULD fire
    _service.Update(new Account { Id = accountId, Name = "New Name" });
    
    // ASSERT: Plugin executed (verify side effects)
}
```

**Multiple Attributes:** Plugin fires if **any** of the specified attributes are in the update.

```csharp
filteringAttributes: new string[] { "name", "revenue", "industrycode" }
// Fires if name OR revenue OR industrycode is updated
```

## Entity Images

### Registering Images

Images are snapshots of the entity before (PreImage) and after (PostImage) the operation:

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
        Revenue = new Money(100000),
        IndustryCode = new OptionSetValue(1)
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
    
    // Plugin can now access:
    // - Target: { revenue = 200000 } (only changed attributes)
    // - PreImage["name"]: "Old Name" (all attributes from before update)
    // - PreImage["revenue"]: 100000
}
```

### PreImage vs PostImage

| Image Type | When Available | Contains | Common Use Cases |
|---|---|---|---|
| **PreImage** | Update, Delete | Entity state **before** operation | Compare old vs new values, audit logs, conditional logic |
| **PostImage** | Create, Update | Entity state **after** operation | Access auto-calculated fields, get complete record |

### Image with Specific Attributes

Reduce image size by specifying only needed attributes:

```csharp
var preImageDefinition = new PluginImageDefinition(
    "PreImage",
    ProcessingStepImageType.PreImage,
    new string[] { "name", "revenue", "industrycode" }  // Only these attributes
);

_context.RegisterPluginStep<MyPlugin>(new PluginStepDefinition
{
    EntityLogicalName = "account",
    MessageName = "Update",
    Stage = ProcessingStepStage.Postoperation,
    RegisteredImages = new[] { preImageDefinition }
});
```

Now `PreImage` entity only contains name, revenue, and industrycode — even if the actual record has more attributes.

### Both PreImage and PostImage

```csharp
var preImage = new PluginImageDefinition("PreImage", ProcessingStepImageType.PreImage);
var postImage = new PluginImageDefinition("PostImage", ProcessingStepImageType.PostImage);

_context.RegisterPluginStep<ComparePlugin>(new PluginStepDefinition
{
    EntityLogicalName = "account",
    MessageName = "Update",
    Stage = ProcessingStepStage.Postoperation,
    RegisteredImages = new[] { preImage, postImage }
});
```

### Accessing Images in Plugin Code

```csharp
public class AuditChangePlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        if (context.MessageName == "Update" && context.PreEntityImages.Contains("PreImage"))
        {
            var preImage = context.PreEntityImages["PreImage"];
            var target = (Entity)context.InputParameters["Target"];
            
            // Compare old vs new
            var oldRevenue = preImage.GetAttributeValue<Money>("revenue");
            var newRevenue = target.GetAttributeValue<Money>("revenue");
            
            if (newRevenue != null && oldRevenue.Value != newRevenue.Value)
            {
                // Revenue changed - do something
            }
        }
    }
}
```

## Plugin Execution Order (Rank)

When multiple plugins are registered for the same message/stage/mode, control execution order with Rank:

```csharp
[Fact]
public void Plugins_Should_Execute_In_Rank_Order()
{
    // ARRANGE
    _context.RegisterPluginStep<FirstPlugin>(new PluginStepDefinition
    {
        MessageName = "Create",
        EntityLogicalName = "account",
        Stage = ProcessingStepStage.Preoperation,
        Rank = 1  // Executes first
    });
    
    _context.RegisterPluginStep<SecondPlugin>(new PluginStepDefinition
    {
        MessageName = "Create",
        EntityLogicalName = "account",
        Stage = ProcessingStepStage.Preoperation,
        Rank = 2  // Executes second
    });
    
    _context.RegisterPluginStep<ThirdPlugin>(new PluginStepDefinition
    {
        MessageName = "Create",
        EntityLogicalName = "account",
        Stage = ProcessingStepStage.Preoperation,
        Rank = 10  // Executes third (higher number = later)
    });
    
    // ACT
    var accountId = _service.Create(new Account { Name = "Test" });
    
    // ASSERT: Verify execution order via side effects
}
```

**Best Practice:** Use gaps (1, 10, 20) instead of consecutive numbers (1, 2, 3) to allow inserting
plugins later without reordering.

## Secure and Unsecure Configurations

Test plugins that use constructor configurations:

```csharp
[Fact]
public void Should_Pass_Configurations_To_Plugin()
{
    // ARRANGE
    var secureConfig = "SecretApiKey123";
    var unsecureConfig = "PublicSetting=Value";
    
    _context.RegisterPluginStep<ConfigurablePlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "Create",
        Stage = ProcessingStepStage.Postoperation,
        Configurations = new PluginStepConfigurations
        {
            SecureConfig = secureConfig,
            UnsecureConfig = unsecureConfig
        }
    });
    
    // ACT
    _service.Create(new Account { Name = "Test" });
    
    // Plugin constructor receives: new ConfigurablePlugin(unsecureConfig, secureConfig)
}
```

**Plugin Implementation:**

```csharp
public class ConfigurablePlugin : IPlugin
{
    private readonly string _unsecureConfig;
    private readonly string _secureConfig;
    
    public ConfigurablePlugin(string unsecureConfig, string secureConfig)
    {
        _unsecureConfig = unsecureConfig;
        _secureConfig = secureConfig;
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        // Use configurations
    }
}
```

## Custom Plugin Instances (Dependency Injection)

Register plugin with pre-constructed instance:

```csharp
[Fact]
public void Should_Use_Custom_Plugin_Instance()
{
    // ARRANGE: Create plugin with injected dependencies
    var mockLogger = new Mock<ILogger>();
    var mockApiService = new Mock<IExternalApiService>();
    
    var pluginInstance = new CustomPlugin(mockLogger.Object, mockApiService.Object);
    
    _context.RegisterPluginStep<CustomPlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "Create",
        Stage = ProcessingStepStage.Postoperation,
        PluginInstance = pluginInstance  // Use this specific instance
    });
    
    // ACT
    _service.Create(new Account { Name = "Test" });
    
    // ASSERT
    mockLogger.Verify(l => l.Log(It.IsAny<string>()), Times.Once);
    mockApiService.Verify(a => a.SendData(It.IsAny<string>()), Times.Once);
}
```

## Testing Max Depth

Prevent infinite loops when plugins trigger other plugins:

```csharp
[Fact]
public void Should_Respect_Max_Depth()
{
    // ARRANGE
    _context.RegisterPluginStep<RecursivePlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "Update",
        Stage = ProcessingStepStage.Postoperation
    });
    
    var accountId = Guid.NewGuid();
    _context.Initialize(new[] { new Account { Id = accountId, Name = "Test" } });
    
    // ACT: Plugin updates account, which triggers itself
    // FakeXrmEasy tracks depth automatically
    _service.Update(new Account { Id = accountId, Revenue = new Money(100) });
    
    // Plugin should check context.Depth and stop recursion
    // Verify it doesn't infinite loop
}
```

**Plugin Implementation:**

```csharp
public class RecursivePlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        if (context.Depth > 1)
            return; // Prevent infinite recursion
        
        // Do work that might trigger this plugin again
    }
}
```

## Complete Example: Multi-Plugin Test

```csharp
[Fact]
public void Complete_Pipeline_Simulation_Example()
{
    // ARRANGE: Setup pipeline with multiple plugins
    
    // 1. PreValidation - Validate data
    _context.RegisterPluginStep<ValidationPlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "Create",
        Stage = ProcessingStepStage.Prevalidation,
        Rank = 1
    });
    
    // 2. PreOperation - Set defaults
    _context.RegisterPluginStep<DefaultsPlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "Create",
        Stage = ProcessingStepStage.Preoperation,
        Rank = 1
    });
    
    // 3. PostOperation - Create related records
    var preImage = new PluginImageDefinition("PostImage", ProcessingStepImageType.PostImage);
    _context.RegisterPluginStep<RelatedRecordsPlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "Create",
        Stage = ProcessingStepStage.Postoperation,
        Rank = 1,
        RegisteredImages = new[] { preImage }
    });
    
    // ACT: Single service call triggers all plugins
    var accountId = _service.Create(new Account { Name = "Contoso" });
    
    // ASSERT: Verify complete pipeline execution
    var account = _service.Retrieve("account", accountId, new ColumnSet(true)) as Account;
    Assert.NotNull(account.AccountNumber); // DefaultsPlugin set this
    Assert.NotNull(account.CreatedOn);
    
    // RelatedRecordsPlugin created a task
    var tasks = _context.CreateQuery<Task>()
        .Where(t => t.RegardingObjectId.Id == accountId)
        .ToList();
    Assert.Single(tasks);
}
```

## Troubleshooting Pipeline Simulation

### Plugin Not Firing

1. Check middleware setup - `UsePipelineSimulation()` must be first
2. Verify entity logical name matches
3. Check message name (case-sensitive: "Create" not "create")
4. Verify filtering attributes if specified

### Images Not Available in Plugin

1. Check `RegisteredImages` array in `PluginStepDefinition`
2. Verify image name matches what plugin expects
3. PreImage not available on Create (entity doesn't exist yet)
4. PostImage not available on Delete (entity already deleted)

### Depth Issues

FakeXrmEasy tracks depth automatically, but doesn't enforce max depth limits. Your plugin must check
`context.Depth` and handle recursion.

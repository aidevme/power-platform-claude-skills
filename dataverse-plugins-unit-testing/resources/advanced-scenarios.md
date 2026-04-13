# Advanced Testing Scenarios

This guide covers advanced plugin testing techniques including dependency injection, mocking external
services, testing metadata operations, and interaction testing.

## Dependency Injection in Plugins

### Constructor Injection

The recommended approach for testable plugins:

```csharp
public class NotificationPlugin : IPlugin
{
    private readonly IEmailService _emailService;
    private readonly ILogger _logger;
    
    // Production constructor (called by Dataverse)
    public NotificationPlugin()
        : this(new EmailService(), new Logger())
    {
    }
    
    // Test constructor (called by FakeXrmEasy)
    public NotificationPlugin(IEmailService emailService, ILogger logger)
    {
        _emailService = emailService ?? throw new ArgumentNullException(nameof(emailService));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        _logger.Log("Plugin executing");
        // Use _emailService and _logger
    }
}
```

**Test with mocked dependencies:**

```csharp
[Fact]
public void Should_Send_Email_When_Account_Created()
{
    // ARRANGE: Create mocks
    var mockEmailService = new Mock<IEmailService>();
    var mockLogger = new Mock<ILogger>();
    
    var pluginInstance = new NotificationPlugin(
        mockEmailService.Object,
        mockLogger.Object
    );
    
    _context.RegisterPluginStep<NotificationPlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "Create",
        Stage = ProcessingStepStage.Postoperation,
        PluginInstance = pluginInstance  // Use our instance with mocks
    });
    
    // ACT
    _service.Create(new Account { Name = "Contoso", EmailAddress1 = "test@contoso.com" });
    
    // ASSERT: Verify interactions
    mockEmailService.Verify(
        e => e.SendEmail(
            "test@contoso.com",
            It.Is<string>(s => s.Contains("Welcome")),
            It.IsAny<string>()
        ),
        Times.Once
    );
    
    mockLogger.Verify(l => l.Log(It.IsAny<string>()), Times.AtLeastOnce);
}
```

### Service Locator Pattern

Alternative using a service locator:

```csharp
public class PluginWithServiceLocator : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var emailService = ServiceLocator.Instance.GetService<IEmailService>();
        emailService.SendEmail(...);
    }
}

[Fact]
public void Test_With_Service_Locator()
{
    // ARRANGE: Register mock in service locator
    var mockEmailService = new Mock<IEmailService>();
    ServiceLocator.Instance.RegisterService<IEmailService>(mockEmailService.Object);
    
    _context.RegisterPluginStep<PluginWithServiceLocator>("Create", ProcessingStepStage.Postoperation);
    
    // ACT
    _service.Create(new Account { Name = "Test" });
    
    // ASSERT
    mockEmailService.Verify(e => e.SendEmail(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()));
}
```

## Mocking External Services

### HTTP Requests

Test plugins that call external APIs:

```csharp
public interface IHttpClientWrapper
{
    Task<string> GetAsync(string url);
    Task<string> PostAsync(string url, string content);
}

public class ExternalApiPlugin : IPlugin
{
    private readonly IHttpClientWrapper _httpClient;
    
    public ExternalApiPlugin() : this(new HttpClientWrapper()) { }
    
    public ExternalApiPlugin(IHttpClientWrapper httpClient)
    {
        _httpClient = httpClient;
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        var result = _httpClient.PostAsync(
            "https://api.example.com/notify",
            "{\"message\":\"Account created\"}"
        ).Result;
    }
}

[Fact]
public void Should_Call_External_Api_On_Create()
{
    // ARRANGE
    var mockHttp = new Mock<IHttpClientWrapper>();
    mockHttp
        .Setup(h => h.PostAsync(
            It.Is<string>(url => url.Contains("api.example.com")),
            It.IsAny<string>()
        ))
        .ReturnsAsync("{\"status\":\"success\"}");
    
    var plugin = new ExternalApiPlugin(mockHttp.Object);
    _context.RegisterPluginStep<ExternalApiPlugin>(new PluginStepDefinition
    {
        MessageName = "Create",
        EntityLogicalName = "account",
        Stage = ProcessingStepStage.Postoperation,
        PluginInstance = plugin
    });
    
    // ACT
    _service.Create(new Account { Name = "Contoso" });
    
    // ASSERT
    mockHttp.Verify(
        h => h.PostAsync(
            "https://api.example.com/notify",
            It.Is<string>(json => json.Contains("Account created"))
        ),
        Times.Once
    );
}
```

### File System Operations

Test plugins that interact with files:

```csharp
public interface IFileSystem
{
    string ReadFile(string path);
    void WriteFile(string path, string content);
    bool FileExists(string path);
}

[Fact]
public void Should_Write_Audit_Log_To_File()
{
    // ARRANGE
    var mockFileSystem = new Mock<IFileSystem>();
    var plugin = new AuditPlugin(mockFileSystem.Object);
    
    _context.RegisterPluginStep<AuditPlugin>(new PluginStepDefinition
    {
        MessageName = "Update",
        EntityLogicalName = "account",
        Stage = ProcessingStepStage.Postoperation,
        PluginInstance = plugin
    });
    
    var accountId = Guid.NewGuid();
    _context.Initialize(new[] { new Account { Id = accountId, Name = "Old" } });
    
    // ACT
    _service.Update(new Account { Id = accountId, Name = "New" });
    
    // ASSERT
    mockFileSystem.Verify(
        fs => fs.WriteFile(
            It.Is<string>(path => path.Contains("audit")),
            It.Is<string>(content => content.Contains("Old") && content.Contains("New"))
        ),
        Times.Once
    );
}
```

## Testing with Metadata

### Fake Attribute Metadata

Test plugins that query attribute metadata:

```csharp
[Fact]
public void Should_Process_All_String_Attributes()
{
    // ARRANGE: Setup fake metadata
    var accountMetadata = new EntityMetadata
    {
        LogicalName = "account"
    };
    accountMetadata.SetSealedPropertyValue("Attributes", new AttributeMetadata[]
    {
        new StringAttributeMetadata { LogicalName = "name", MaxLength = 100 },
        new StringAttributeMetadata { LogicalName = "description", MaxLength = 500 },
        new IntegerAttributeMetadata { LogicalName = "revenue" }
    });
    
    _context.InitializeMetadata(accountMetadata);
    
    // Plugin can now use RetrieveEntityRequest, RetrieveAttributeRequest
    _context.RegisterPluginStep<MetadataPlugin>("Create", ProcessingStepStage.Preoperation);
    
    // ACT & ASSERT
    _service.Create(new Account { Name = "Test" });
}
```

### Entity Relationships

Test plugins that traverse relationships:

```csharp
[Fact]
public void Should_Access_Related_Entities_Via_Metadata()
{
    // ARRANGE: Setup relationship metadata
    var relationship = new OneToManyRelationshipMetadata
    {
        ReferencedEntity = "account",
        ReferencedAttribute = "accountid",
        ReferencingEntity = "contact",
        ReferencingAttribute = "parentcustomerid",
        SchemaName = "account_primary_contact"
    };
    
    var accountMetadata = new EntityMetadata { LogicalName = "account" };
    accountMetadata.SetSealedPropertyValue("OneToManyRelationships", 
        new[] { relationship });
    
    _context.InitializeMetadata(accountMetadata);
    
    // Test plugin that uses this relationship
}
```

## Interaction Testing

Test **what** the plugin does, not **how** it does it. Verify service calls without testing implementation details.

### Verifying Service Operations

```csharp
public interface IOrganizationServiceWrapper
{
    Guid Create(Entity entity);
    void Update(Entity entity);
    void Delete(string entityName, Guid id);
}

[Fact]
public void Should_Create_Task_After_Opportunity_Won()
{
    // ARRANGE
    var mockService = new Mock<IOrganizationServiceWrapper>();
    var plugin = new OpportunityWonPlugin(mockService.Object);
    
    var opportunity = new Opportunity 
    { 
        Id = Guid.NewGuid(), 
        Name = "Big Deal",
        StateCode = new OptionSetValue(1), // Won
        StatusCode = new OptionSetValue(3) // Won
    };
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Update";
    ctx.InputParameters["Target"] = opportunity;
    
    // ACT
    plugin.Execute(MockServiceProvider(ctx));
    
    // ASSERT: Verify Task was created
    mockService.Verify(
        s => s.Create(It.Is<Entity>(e => 
            e.LogicalName == "task" &&
            e.GetAttributeValue<EntityReference>("regardingobjectid").Id == opportunity.Id &&
            e.GetAttributeValue<string>("subject").Contains("Follow up")
        )),
        Times.Once,
        "Plugin should create follow-up task when opportunity is won"
    );
}
```

### Verifying Query Interactions

Test that plugin queries data correctly:

```csharp
[Fact]
public void Should_Query_Related_Contacts()
{
    // ARRANGE
    var mockService = new Mock<IOrganizationService>();
    
    var accountId = Guid.NewGuid();
    var contacts = new EntityCollection(new[]
    {
        new Entity("contact") { Id = Guid.NewGuid(), ["lastname"] = "Smith" },
        new Entity("contact") { Id = Guid.NewGuid(), ["lastname"] = "Jones" }
    });
    
    mockService
        .Setup(s => s.RetrieveMultiple(It.IsAny<QueryExpression>()))
        .Returns<QueryExpression>(query =>
        {
            // Verify query structure
            Assert.Equal("contact", query.EntityName);
            Assert.Contains(query.Criteria.Conditions, 
                c => c.AttributeName == "parentcustomerid" && 
                     ((Guid)c.Values[0]) == accountId);
            
            return contacts;
        });
    
    var plugin = new UpdateContactsPlugin(mockService.Object);
    
    // Test plugin execution...
    
    // Verify RetrieveMultiple was called
    mockService.Verify(s => s.RetrieveMultiple(It.IsAny<QueryExpression>()), Times.Once);
}
```

## Testing Custom Messages and Actions

### Custom Actions

Test plugins that respond to custom actions:

```csharp
[Fact]
public void Should_Handle_Custom_Action()
{
    // ARRANGE
    _context.RegisterPluginStep<CustomActionPlugin>(new PluginStepDefinition
    {
        MessageName = "new_CalculateDiscount", // Custom action name
        EntityLogicalName = "account",
        Stage = ProcessingStepStage.Postoperation
    });
    
    var accountId = Guid.NewGuid();
    _context.Initialize(new[] { new Account { Id = accountId, Revenue = new Money(100000) } });
    
    // ACT: Execute custom action
    var request = new OrganizationRequest("new_CalculateDiscount")
    {
        ["Target"] = new EntityReference("account", accountId),
        ["DiscountPercentage"] = 10m
    };
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.True(response.Results.Contains("DiscountAmount"));
    Assert.Equal(10000m, (decimal)response.Results["DiscountAmount"]);
}
```

### Custom APIs

```csharp
[Fact]
public void Should_Execute_Custom_API()
{
    // ARRANGE: Register plugin for Custom API
    _context.RegisterPluginStep<CustomApiPlugin>(new PluginStepDefinition
    {
        MessageName = "new_GetAccountSummary",
        Stage = ProcessingStepStage.Postoperation
    });
    
    var accountId = Guid.NewGuid();
    _context.Initialize(new[] 
    { 
        new Account { Id = accountId, Name = "Contoso", Revenue = new Money(500000) }
    });
    
    // ACT
    var request = new OrganizationRequest("new_GetAccountSummary");
    request["AccountId"] = accountId.ToString();
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.Equal("Contoso", response["AccountName"]);
    Assert.Equal(500000m, ((Money)response["Revenue"]).Value);
}
```

## Testing Async Plugins

While plugins themselves don't use async/await (IPlugin.Execute is synchronous), you can test
plugins that use async operations internally:

```csharp
public class AsyncOperationsPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        // Plugin execution is synchronous, but can call async methods
        var task = ProcessAsync(serviceProvider);
        task.Wait(); // Block until complete
    }
    
    private async Task ProcessAsync(IServiceProvider serviceProvider)
    {
        await Task.Delay(100); // Simulate async operation
        // Do work
    }
}

[Fact]
public async Task Should_Complete_Async_Operations()
{
    // Standard test - Execute blocks until async work completes
    _context.RegisterPluginStep<AsyncOperationsPlugin>("Create", ProcessingStepStage.Postoperation);
    
    _service.Create(new Account { Name = "Test" });
    
    // By the time Create returns, async work is done
    // Assert results...
}
```

### Using IOrganizationServiceAsync

For .NET Core 3.1+ projects (FakeXrmEasy 3.x), you can use async service interfaces:

```csharp
public class AsyncServiceTestsBase
{
    protected readonly IXrmFakedContext _context;
    protected readonly IOrganizationServiceAsync _asyncService;
    
    public AsyncServiceTestsBase()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .UseCrud()
            .Build();
        
        // Get async service interface
        _asyncService = _context.GetAsyncOrganizationService();
    }
}

[Fact]
public async Task Should_Create_Account_Async()
{
    // ARRANGE
    var account = new Account { Name = "Async Account" };
    
    // ACT: Use async Create method
    var accountId = await _asyncService.CreateAsync(account);
    
    // ASSERT
    Assert.NotEqual(Guid.Empty, accountId);
    
    var created = await _asyncService.RetrieveAsync("account", accountId, new ColumnSet(true));
    Assert.Equal("Async Account", ((Account)created).Name);
}
```

### IOrganizationServiceAsync2 with Cancellation Tokens

```csharp
public class AsyncWithCancellationTestsBase
{
    protected readonly IXrmFakedContext _context;
    protected readonly IOrganizationServiceAsync2 _asyncService;
    
    public AsyncWithCancellationTestsBase()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .UseCrud()
            .Build();
        
        _asyncService = _context.GetAsyncOrganizationService2();
    }
}

[Fact]
public async Task Should_Support_Cancellation_Tokens()
{
    // ARRANGE
    var cancellationTokenSource = new CancellationTokenSource();
    var account = new Account { Name = "Cancellable Operation" };
    
    // ACT
    var accountId = await _asyncService.CreateAsync(account, cancellationTokenSource.Token);
    
    // ASSERT
    Assert.NotEqual(Guid.Empty, accountId);
}
```

## Testing with Tracing

Verify plugin writes useful trace messages:

```csharp
public class TracingPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var tracingService = (ITracingService)serviceProvider.GetService(typeof(ITracingService));
        
        tracingService.Trace("Plugin started");
        
        try
        {
            // Do work
            tracingService.Trace("Work completed successfully");
        }
        catch (Exception ex)
        {
            tracingService.Trace($"Error: {ex.Message}");
            throw;
        }
    }
}

[Fact]
public void Should_Write_Trace_Messages()
{
    // ARRANGE
    var traceMessages = new List<string>();
    
    var mockTracingService = new Mock<ITracingService>();
    mockTracingService
        .Setup(t => t.Trace(It.IsAny<string>()))
        .Callback<string>(msg => traceMessages.Add(msg));
    
    // Inject mock tracing service via custom service provider
    // (Advanced - requires custom IServiceProvider implementation)
    
    // ACT
    // Execute plugin...
    
    // ASSERT
    Assert.Contains(traceMessages, m => m.Contains("Plugin started"));
    Assert.Contains(traceMessages, m => m.Contains("completed successfully"));
}
```

## Testing with ILogger (Application Insights Telemetry)

**Minimum Version:** FakeXrmEasy 2.1.0+, Microsoft.CrmSdk.CoreAssemblies 9.0.2.27+

Dataverse supports sending telemetry to Azure Application Insights using the ILogger interface.
FakeXrmEasy provides a default implementation for testing.

### Plugin Using ILogger

```csharp
public class TelemetryPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var logger = (ILogger)serviceProvider.GetService(typeof(ILogger));
        
        logger?.LogInformation($"Plugin executing for {context.MessageName}");
        
        try
        {
            // Do work
            var account = (Entity)context.InputParameters["Target"];
            logger?.LogInformation($"Processing account: {account.GetAttributeValue<string>("name")}");
            
            // Business logic...
            
            logger?.LogInformation("Plugin completed successfully");
        }
        catch (Exception ex)
        {
            logger?.LogError($"Plugin error: {ex.Message}", ex);
            throw;
        }
    }
}

[Fact]
public void Should_Log_Telemetry_Events()
{
    // ARRANGE: FakeXrmEasy provides ILogger automatically
    _context.RegisterPluginStep<TelemetryPlugin>("Create", ProcessingStepStage.Postoperation);
    
    // ACT
    _service.Create(new Account { Name = "Telemetry Test" });
    
    // ASSERT: Plugin executed without errors
    // In production, these logs would appear in Application Insights
    // In tests, FakeXrmEasy's default ILogger handles them
}
```

### Mocking ILogger for Verification

```csharp
[Fact]
public void Should_Verify_Specific_Log_Messages()
{
    // ARRANGE: Create mock logger
    var mockLogger = new Mock<ILogger>();
    
    // Track logged messages
    var loggedMessages = new List<string>();
    mockLogger
        .Setup(l => l.LogInformation(It.IsAny<string>()))
        .Callback<string>(msg => loggedMessages.Add(msg));
    
    // Custom plugin instance with mock logger injected
    // (Requires custom IServiceProvider that returns mock logger)
    
    // ACT: Execute plugin
    _service.Create(new Account { Name = "Test" });
    
    // ASSERT: Verify expected log messages
    Assert.Contains(loggedMessages, m => m.Contains("Plugin executing"));
    Assert.Contains(loggedMessages, m => m.Contains("completed successfully"));
}
```

### ILogger Methods Supported

```csharp
public class ComprehensiveLoggingPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var logger = (ILogger)serviceProvider.GetService(typeof(ILogger));
        
        // Different log levels
        logger.LogTrace("Trace message");           // Detailed diagnostic
        logger.LogDebug("Debug message");           // Debugging info
        logger.LogInformation("Info message");      // General information
        logger.LogWarning("Warning message");       // Warning conditions
        logger.LogError("Error message");           // Error conditions
        logger.LogCritical("Critical message");     // Critical failures
        
        // With exception
        try
        {
            // Risky operation
        }
        catch (Exception ex)
        {
            logger.LogError("Operation failed", ex);
        }
    }
}
```

**Note:** FakeXrmEasy 2.1.0+ automatically provides a working ILogger implementation. You don't
need special configuration unless you want to mock/verify specific log calls.

## Performance Testing

Test plugin execution time:

```csharp
[Fact]
public void Should_Execute_Within_Time_Limit()
{
    // ARRANGE
    _context.RegisterPluginStep<PerformancePlugin>("Create", ProcessingStepStage.Postoperation);
    
    var stopwatch = Stopwatch.StartNew();
    
    // ACT
    _service.Create(new Account { Name = "Test" });
    
    stopwatch.Stop();
    
    // ASSERT: Should complete in under 1 second
    Assert.True(stopwatch.ElapsedMilliseconds < 1000, 
        $"Plugin took {stopwatch.ElapsedMilliseconds}ms, expected < 1000ms");
}
```

## Testing with Large Datasets

Test plugin performance with realistic data volumes:

```csharp
[Fact]
public void Should_Handle_Large_Account_Portfolio()
{
    // ARRANGE: Create 1000 test accounts
    var accounts = Enumerable.Range(1, 1000)
        .Select(i => new Account 
        { 
            Id = Guid.NewGuid(), 
            Name = $"Account {i}",
            Revenue = new Money(i * 1000)
        })
        .ToList();
    
    _context.Initialize(accounts);
    
    _context.RegisterPluginStep<CalculateTotalRevenuePlugin>("Update", ProcessingStepStage.Postoperation);
    
    // ACT: Update that triggers calculation across all accounts
    _service.Update(new Account { Id = accounts[0].Id, Name = "Updated" });
    
    // ASSERT: Verify calculation completed
    // Should still be fast even with 1000 records in memory
}
```

## Testing Error Scenarios

### Network Timeouts

```csharp
[Fact]
public void Should_Handle_External_Service_Timeout()
{
    // ARRANGE
    var mockHttp = new Mock<IHttpClientWrapper>();
    mockHttp
        .Setup(h => h.PostAsync(It.IsAny<string>(), It.IsAny<string>()))
        .ThrowsAsync(new TimeoutException("Request timed out"));
    
    var plugin = new ExternalApiPlugin(mockHttp.Object);
    
    // ACT & ASSERT
    var ex = Assert.Throws<InvalidPluginExecutionException>(() =>
    {
        var ctx = _context.GetDefaultPluginContext();
        ctx.InputParameters["Target"] = new Account { Name = "Test" };
        plugin.Execute(MockServiceProvider(ctx));
    });
    
    Assert.Contains("external service", ex.Message.ToLower());
}
```

### Invalid Data

```csharp
[Theory]
[InlineData(null)]
[InlineData("")]
[InlineData("   ")]
public void Should_Throw_When_Required_Field_Missing(string invalidName)
{
    // ARRANGE
    _context.RegisterPluginStep<ValidationPlugin>("Create", ProcessingStepStage.Prevalidation);
    
    // ACT & ASSERT
    var ex = Assert.Throws<InvalidPluginExecutionException>(() =>
    {
        _service.Create(new Account { Name = invalidName });
    });
    
    Assert.Contains("Name is required", ex.Message);
}
```

## Best Practices Summary

1. **Mock external dependencies** — HTTP, files, external databases
2. **Use constructor injection** for testability
3. **Test error paths** as thoroughly as happy paths
4. **Verify interactions**, not implementation details
5. **Keep tests fast** — in-memory operations, no real I/O
6. **Use descriptive test names** — explain the scenario
7. **Test realistic data volumes** for performance-critical plugins
8. **Verify trace messages** for debugging production issues

# Azure Functions Testing with Dataverse

Test Azure Functions that interact with Dataverse using FakeXrmEasy. This allows local development
and testing without deploying to Azure or connecting to a live Dataverse instance.

**Recommended Version:** FakeXrmEasy 3.x (.NET Core 3.1+) for modern Azure Functions v3/v4

## Overview

Azure Functions + Dataverse scenarios:
- HTTP triggers that create/update Dataverse records
- Timer triggers for scheduled data synchronization
- Queue triggers for asynchronous processing
- Webhook endpoints for external system integration
- Background jobs for bulk operations

**Benefits of testing with FakeXrmEasy:**
- Fast test execution (no network latency)
- Isolated tests (no shared state)
- Repeatable results
- No Azure deployment required
- No Dataverse environment needed

## Setup

### Install Packages

```powershell
# FakeXrmEasy for .NET Core (Azure Functions v3/v4)
Install-Package FakeXrmEasy.v9 -Version 3.6.3

# Azure Functions testing dependencies
Install-Package Microsoft.Azure.Functions.Worker -Version 1.19.0
Install-Package Microsoft.Extensions.Logging.Abstractions -Version 7.0.0
```

### Project Structure

```
MyFunctionApp/
├── MyFunctionApp/                  # Function project (.NET 6/7)
│   ├── Functions/
│   │   ├── CreateAccountFunction.cs
│   │   └── SyncContactsFunction.cs
│   ├── Services/
│   │   └── DataverseService.cs
│   └── host.json
│
└── MyFunctionApp.Tests/             # Test project
    ├── CreateAccountFunctionTests.cs
    ├── SyncContactsFunctionTests.cs
    └── TestBase.cs
```

## Testing HTTP Triggered Functions

### Function Implementation

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using Microsoft.Xrm.Sdk;
using System.Net;

public class CreateAccountFunction
{
    private readonly IOrganizationService _dataverseService;
    private readonly ILogger<CreateAccountFunction> _logger;
    
    public CreateAccountFunction(
        IOrganizationServiceFactory serviceFactory,
        ILogger<CreateAccountFunction> logger)
    {
        _dataverseService = serviceFactory.CreateOrganizationService(null);
        _logger = logger;
    }
    
    [Function("CreateAccount")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req)
    {
        _logger.LogInformation("CreateAccount function triggered");
        
        var requestBody = await req.ReadAsStringAsync();
        var accountData = JsonSerializer.Deserialize<AccountRequest>(requestBody);
        
        var account = new Entity("account");
        account["name"] = accountData.Name;
        account["telephone1"] = accountData.Phone;
        
        var accountId = _dataverseService.Create(account);
        
        _logger.LogInformation($"Created account with ID: {accountId}");
        
        var response = req.CreateResponse(HttpStatusCode.Created);
        await response.WriteAsJsonAsync(new { id = accountId });
        return response;
    }
}

public class AccountRequest
{
    public string Name { get; set; }
    public string Phone { get; set; }
}
```

### Test Implementation

```csharp
using FakeXrmEasy;
using FakeXrmEasy.Abstractions;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using Moq;
using System.Net;
using Xunit;

public class CreateAccountFunctionTests
{
    private readonly IXrmFakedContext _context;
    private readonly IOrganizationService _service;
    private readonly Mock<ILogger<CreateAccountFunction>> _loggerMock;
    
    public CreateAccountFunctionTests()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .UseCrud()
            .Build();
            
        _service = _context.GetOrganizationService();
        _loggerMock = new Mock<ILogger<CreateAccountFunction>>();
    }
    
    [Fact]
    public async Task Should_Create_Account_From_HTTP_Request()
    {
        // ARRANGE
        var serviceFactoryMock = new Mock<IOrganizationServiceFactory>();
        serviceFactoryMock
            .Setup(f => f.CreateOrganizationService(It.IsAny<Guid?>()))
            .Returns(_service);
        
        var function = new CreateAccountFunction(
            serviceFactoryMock.Object,
            _loggerMock.Object
        );
        
        var requestMock = new Mock<HttpRequestData>(Mock.Of<FunctionContext>());
        var requestBody = JsonSerializer.Serialize(new AccountRequest
        {
            Name = "Contoso",
            Phone = "555-1234"
        });
        
        var requestStream = new MemoryStream(Encoding.UTF8.GetBytes(requestBody));
        requestMock.Setup(r => r.Body).Returns(requestStream);
        requestMock.Setup(r => r.ReadAsStringAsync()).ReturnsAsync(requestBody);
        
        var responseMock = new Mock<HttpResponseData>(Mock.Of<FunctionContext>());
        responseMock.SetupProperty(r => r.StatusCode);
        requestMock.Setup(r => r.CreateResponse(It.IsAny<HttpStatusCode>()))
            .Returns(responseMock.Object);
        
        // ACT
        var response = await function.Run(requestMock.Object);
        
        // ASSERT
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        
        var accounts = _context.CreateQuery<Account>().ToList();
        Assert.Single(accounts);
        Assert.Equal("Contoso", accounts[0].Name);
        Assert.Equal("555-1234", accounts[0].Telephone1);
        
        _loggerMock.Verify(
            l => l.Log(
                LogLevel.Information,
                It.IsAny<EventId>(),
                It.Is<It.IsAnyType>((v, t) => v.ToString().Contains("Created account")),
                null,
                It.IsAny<Func<It.IsAnyType, Exception, string>>()
            ),
            Times.Once
        );
    }
}
```

## Testing Timer Triggered Functions

### Function Implementation

```csharp
public class SyncContactsFunction
{
    private readonly IOrganizationService _dataverseService;
    private readonly ILogger<SyncContactsFunction> _logger;
    
    public SyncContactsFunction(
        IOrganizationServiceFactory serviceFactory,
        ILogger<SyncContactsFunction> logger)
    {
        _dataverseService = serviceFactory.CreateOrganizationService(null);
        _logger = logger;
    }
    
    [Function("SyncContacts")]
    public void Run([TimerTrigger("0 */5 * * * *")] TimerInfo timer)
    {
        _logger.LogInformation($"SyncContacts function started at: {DateTime.Now}");
        
        // Query contacts that need syncing
        var query = new QueryExpression("contact");
        query.ColumnSet = new ColumnSet("firstname", "lastname", "emailaddress1");
        query.Criteria.AddCondition("statecode", ConditionOperator.Equal, 0);
        query.Criteria.AddCondition("needssync", ConditionOperator.Equal, true);
        
        var contacts = _dataverseService.RetrieveMultiple(query).Entities;
        
        _logger.LogInformation($"Found {contacts.Count} contacts to sync");
        
        foreach (var contact in contacts)
        {
            // Sync logic here
            contact["needssync"] = false;
            contact["lastsyncdate"] = DateTime.UtcNow;
            _dataverseService.Update(contact);
        }
        
        _logger.LogInformation($"Sync completed. Processed {contacts.Count} contacts");
    }
}
```

### Test Implementation

```csharp
[Fact]
public void Should_Sync_Contacts_Marked_For_Sync()
{
    // ARRANGE
    var contact1 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "John",
        LastName = "Doe",
        StateCode = ContactState.Active,
        ["needssync"] = true
    };
    
    var contact2 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "Jane",
        LastName = "Smith",
        StateCode = ContactState.Active,
        ["needssync"] = true
    };
    
    var contact3 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "Bob",
        LastName = "Johnson",
        StateCode = ContactState.Active,
        ["needssync"] = false  // Already synced
    };
    
    _context.Initialize(new[] { contact1, contact2, contact3 });
    
    var serviceFactoryMock = new Mock<IOrganizationServiceFactory>();
    serviceFactoryMock
        .Setup(f => f.CreateOrganizationService(It.IsAny<Guid?>()))
        .Returns(_service);
    
    var function = new SyncContactsFunction(
        serviceFactoryMock.Object,
        _loggerMock.Object
    );
    
    var timerInfo = new TimerInfo();
    
    // ACT
    function.Run(timerInfo);
    
    // ASSERT
    var syncedContacts = _context.CreateQuery<Contact>()
        .Where(c => c.GetAttributeValue<bool>("needssync") == false)
        .ToList();
    
    Assert.Equal(3, syncedContacts.Count); // All 3 should now be synced (2 were updated, 1 already was)
    
    // Verify contact1 and contact2 were updated
    var updatedContact1 = _context.CreateQuery<Contact>()
        .First(c => c.Id == contact1.Id);
    Assert.False(updatedContact1.GetAttributeValue<bool>("needssync"));
    Assert.NotNull(updatedContact1.GetAttributeValue<DateTime?>("lastsyncdate"));
    
    _loggerMock.Verify(
        l => l.Log(
            LogLevel.Information,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((v, t) => v.ToString().Contains("Found 2 contacts")),
            null,
            It.IsAny<Func<It.IsAnyType, Exception, string>>()
        ),
        Times.Once
    );
}
```

## Testing with Dependency Injection

Azure Functions typically use dependency injection:

```csharp
public class Startup : FunctionsStartup
{
    public override void Configure(IFunctionsHostBuilder builder)
    {
        // Production: Register real Dataverse service
        builder.Services.AddSingleton<IOrganizationServiceFactory>(provider =>
        {
            var connectionString = Environment.GetEnvironmentVariable("DataverseConnectionString");
            return new ServiceClient(connectionString);
        });
        
        // Register other services
        builder.Services.AddScoped<IDataverseService, DataverseService>();
    }
}

// Test: Override with FakeXrmEasy
public class FunctionTestsBase
{
    protected IXrmFakedContext _context;
    protected IOrganizationService _service;
    
    public FunctionTestsBase()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .UseCrud()
            .Build();
            
        _service = _context.GetOrganizationService();
    }
    
    protected T CreateFunctionWithDependencies<T>()
    {
        var serviceFactoryMock = new Mock<IOrganizationServiceFactory>();
        serviceFactoryMock.Setup(f => f.CreateOrganizationService(It.IsAny<Guid?>()))
            .Returns(_service);
        
        // Use reflection or factory pattern to instantiate function with dependencies
        return (T)Activator.CreateInstance(typeof(T), serviceFactoryMock.Object);
    }
}
```

## Testing Async/Await Operations

FakeXrmEasy 3.x supports async IOrganizationService interfaces:

```csharp
public class AsyncDataFunction
{
    private readonly IOrganizationServiceAsync2 _service;
    
    public AsyncDataFunction(IOrganizationServiceFactory factory)
    {
        _service = (IOrganizationServiceAsync2)factory.CreateOrganizationService(null);
    }
    
    [Function("ProcessDataAsync")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req)
    {
        var accountId = await _service.CreateAsync(new Entity("account"));
        var account = await _service.RetrieveAsync("account", accountId, new ColumnSet(true));
        
        // Process...
        
        return req.CreateResponse(HttpStatusCode.OK);
    }
}

[Fact]
public async Task Should_Process_Data_Asynchronously()
{
    // ARRANGE
    var asyncService = _context.GetAsyncOrganizationService2();
    var serviceFactoryMock = new Mock<IOrganizationServiceFactory>();
    serviceFactoryMock.Setup(f => f.CreateOrganizationService(It.IsAny<Guid?>()))
        .Returns(asyncService);
    
    var function = new AsyncDataFunction(serviceFactoryMock.Object);
    
    // ACT & ASSERT
    // Test async operations...
}
```

## Best Practices

1. **Mock external dependencies** - HTTP clients, file systems, external APIs
2. **Use IOrganizationServiceFactory** - Allows easy substitution in tests
3. **Test error scenarios** - Network failures, invalid data, timeouts
4. **Verify logging** - Ensure diagnostic messages are written
5. **Test both sync and async** - Use async methods when available (v3.x)
6. **Use early-bound entities** - Better IntelliSense and type safety
7. **Test idempotency** - Functions should handle re-execution gracefully
8. **Validate HTTP responses** - Status codes, response bodies, headers

## Common Scenarios

### Webhook Endpoint

```csharp
[Fact]
public async Task Should_Process_Webhook_And_Create_Lead()
{
    // Test function that receives webhook and creates lead
    var webhookPayload = new { email = "test@example.com", name = "Test Lead" };
    // Setup request, execute function, verify lead created
}
```

### Queue-Triggered Processing

```csharp
[Fact]
public void Should_Process_Queue_Message_And_Update_Record()
{
    // Test function triggered by Azure Queue message
    var queueMessage = "{ \"accountId\": \"" + Guid.NewGuid() + "\" }";
    // Execute function, verify record updated
}
```

### Bulk Data Import

```csharp
[Fact]
public async Task Should_Import_Bulk_Data_Using_CreateMultiple()
{
    // Test function that imports CSV data in bulk
    // Use CreateMultipleRequest for performance
}
```

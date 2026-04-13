# Async/Await Testing with IOrganizationServiceAsync

FakeXrmEasy v3.x (.NET Core 3.1+) supports async/await patterns with `IOrganizationServiceAsync` 
and `IOrganizationServiceAsync2` interfaces, enabling modern asynchronous plugin development.

**Version Required:** FakeXrmEasy 3.x only (.NET Core 3.1+)

## Overview

Async interfaces provide:
- Non-blocking I/O operations
- Better scalability for high-volume scenarios
- Improved responsiveness in Azure Functions
- Modern C# async/await patterns

**Note:** .NET Framework plugins (2.x) do NOT support async - plugins execute synchronously in Dataverse.

## Async Interfaces

### IOrganizationServiceAsync

Classic async interface with basic operations:

```csharp
public interface IOrganizationServiceAsync
{
    Task<Guid> CreateAsync(Entity entity);
    Task<Guid> CreateAsync(Entity entity, CancellationToken cancellationToken);
    
    Task<Entity> RetrieveAsync(string entityName, Guid id, ColumnSet columnSet);
    Task<Entity> RetrieveAsync(string entityName, Guid id, ColumnSet columnSet, CancellationToken cancellationToken);
    
    Task UpdateAsync(Entity entity);
    Task UpdateAsync(Entity entity, CancellationToken cancellationToken);
    
    Task DeleteAsync(string entityName, Guid id);
    Task DeleteAsync(string entityName, Guid id, CancellationToken cancellationToken);
    
    Task<OrganizationResponse> ExecuteAsync(OrganizationRequest request);
    Task<OrganizationResponse> ExecuteAsync(OrganizationRequest request, CancellationToken cancellationToken);
}
```

### IOrganizationServiceAsync2

Extended interface with RetrieveMultiple:

```csharp
public interface IOrganizationServiceAsync2 : IOrganizationServiceAsync
{
    Task<EntityCollection> RetrieveMultipleAsync(QueryBase query);
    Task<EntityCollection> RetrieveMultipleAsync(QueryBase query, CancellationToken cancellationToken);
}
```

## Getting Async Service from FakeXrmEasy

```csharp
using FakeXrmEasy;
using FakeXrmEasy.Abstractions;
using Microsoft.Xrm.Sdk;
using Microsoft.Xrm.Sdk.Query;

public class AsyncTestsBase
{
    protected readonly IXrmFakedContext _context;
    
    public AsyncTestsBase()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .UseCrud()
            .Build();
    }
    
    protected IOrganizationServiceAsync GetAsyncService()
    {
        return _context.GetAsyncOrganizationService();
    }
    
    protected IOrganizationServiceAsync2 GetAsyncService2()
    {
        return _context.GetAsyncOrganizationService2();
    }
}
```

## Testing Async CRUD Operations

### CreateAsync

```csharp
[Fact]
public async Task Should_Create_Account_Asynchronously()
{
    // ARRANGE
    var service = _context.GetAsyncOrganizationService();
    
    var account = new Account
    {
        Name = "Contoso",
        Telephone1 = "555-1234"
    };
    
    // ACT
    var accountId = await service.CreateAsync(account);
    
    // ASSERT
    Assert.NotEqual(Guid.Empty, accountId);
    
    var created = await service.RetrieveAsync("account", accountId, new ColumnSet(true));
    Assert.Equal("Contoso", created.GetAttributeValue<string>("name"));
    Assert.Equal("555-1234", created.GetAttributeValue<string>("telephone1"));
}
```

### RetrieveAsync

```csharp
[Fact]
public async Task Should_Retrieve_Account_Asynchronously()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account
    {
        Id = accountId,
        Name = "Contoso",
        NumberOfEmployees = 100
    };
    _context.Initialize(new[] { account });
    
    var service = _context.GetAsyncOrganizationService();
    
    // ACT
    var retrieved = await service.RetrieveAsync(
        "account", 
        accountId, 
        new ColumnSet("name", "numberofemployees")
    );
    
    // ASSERT
    Assert.Equal(accountId, retrieved.Id);
    Assert.Equal("Contoso", retrieved.GetAttributeValue<string>("name"));
    Assert.Equal(100, retrieved.GetAttributeValue<int>("numberofemployees"));
}
```

### UpdateAsync

```csharp
[Fact]
public async Task Should_Update_Account_Asynchronously()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account
    {
        Id = accountId,
        Name = "Contoso",
        Telephone1 = "555-1234"
    };
    _context.Initialize(new[] { account });
    
    var service = _context.GetAsyncOrganizationService();
    
    // ACT
    var accountUpdate = new Account
    {
        Id = accountId,
        Telephone1 = "555-5678"
    };
    await service.UpdateAsync(accountUpdate);
    
    // ASSERT
    var updated = await service.RetrieveAsync("account", accountId, new ColumnSet(true));
    Assert.Equal("555-5678", updated.GetAttributeValue<string>("telephone1"));
    Assert.Equal("Contoso", updated.GetAttributeValue<string>("name")); // Unchanged
}
```

### DeleteAsync

```csharp
[Fact]
public async Task Should_Delete_Account_Asynchronously()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    var service = _context.GetAsyncOrganizationService();
    
    // ACT
    await service.DeleteAsync("account", accountId);
    
    // ASSERT
    var accounts = _context.CreateQuery<Account>().ToList();
    Assert.Empty(accounts);
}
```

## Testing Async Query Operations

### RetrieveMultipleAsync (IOrganizationServiceAsync2)

```csharp
[Fact]
public async Task Should_Query_Accounts_Asynchronously()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso", StateCode = AccountState.Active };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam", StateCode = AccountState.Active };
    var account3 = new Account { Id = Guid.NewGuid(), Name = "Northwind", StateCode = AccountState.Inactive };
    _context.Initialize(new[] { account1, account2, account3 });
    
    var service = _context.GetAsyncOrganizationService2();
    
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name", "statecode");
    query.Criteria.AddCondition("statecode", ConditionOperator.Equal, 0); // Active only
    
    // ACT
    var results = await service.RetrieveMultipleAsync(query);
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count);
    Assert.All(results.Entities, e => 
        Assert.Equal(AccountState.Active, e.GetAttributeValue<AccountState>("statecode"))
    );
}
```

### FetchXML Async Query

```csharp
[Fact]
public async Task Should_Execute_FetchXml_Asynchronously()
{
    // ARRANGE
    var contact1 = new Contact { Id = Guid.NewGuid(), FirstName = "John", LastName = "Doe" };
    var contact2 = new Contact { Id = Guid.NewGuid(), FirstName = "Jane", LastName = "Smith" };
    _context.Initialize(new[] { contact1, contact2 });
    
    var service = _context.GetAsyncOrganizationService2();
    
    var fetchXml = @"
        <fetch>
            <entity name='contact'>
                <attribute name='firstname' />
                <attribute name='lastname' />
                <filter>
                    <condition attribute='lastname' operator='eq' value='Doe' />
                </filter>
            </entity>
        </fetch>";
    
    // ACT
    var results = await service.RetrieveMultipleAsync(new FetchExpression(fetchXml));
    
    // ASSERT
    Assert.Single(results.Entities);
    Assert.Equal("John", results.Entities[0].GetAttributeValue<string>("firstname"));
}
```

## Testing Async ExecuteAsync

### Associate/Disassociate

```csharp
[Fact]
public async Task Should_Associate_Records_Asynchronously()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var contactId = Guid.NewGuid();
    
    var account = new Account { Id = accountId, Name = "Contoso" };
    var contact = new Contact { Id = contactId, FirstName = "John" };
    _context.Initialize(new[] { account, contact });
    
    var service = _context.GetAsyncOrganizationService();
    
    var associateRequest = new AssociateRequest
    {
        Target = account.ToEntityReference(),
        Relationship = new Relationship("contact_customer_accounts"),
        RelatedEntities = new EntityReferenceCollection
        {
            contact.ToEntityReference()
        }
    };
    
    // ACT
    await service.ExecuteAsync(associateRequest);
    
    // ASSERT
    // Verify association exists
    var retrieveRequest = new RetrieveRequest
    {
        Target = account.ToEntityReference(),
        ColumnSet = new ColumnSet(true),
        RelatedEntitiesQuery = new RelationshipQueryCollection
        {
            {
                new Relationship("contact_customer_accounts"),
                new QueryExpression("contact") { ColumnSet = new ColumnSet(true) }
            }
        }
    };
    
    var response = (RetrieveResponse)await service.ExecuteAsync(retrieveRequest);
    // Assertions...
}
```

### WhoAmI

```csharp
[Fact]
public async Task Should_Execute_WhoAmI_Asynchronously()
{
    // ARRANGE
    var userId = Guid.NewGuid();
    var orgId = Guid.NewGuid();
    var context = MiddlewareBuilder.New()
        .AddCrud()
        .SetCallerProperties(new CallerProperties { CallerId = userId })
        .UseCrud()
        .Build();
    
    var service = context.GetAsyncOrganizationService();
    
    var request = new WhoAmIRequest();
    
    // ACT
    var response = (WhoAmIResponse)await service.ExecuteAsync(request);
    
    // ASSERT
    Assert.Equal(userId, response.UserId);
}
```

## Testing with CancellationToken

```csharp
[Fact]
public async Task Should_Support_Cancellation_Token()
{
    // ARRANGE
    var service = _context.GetAsyncOrganizationService();
    var cts = new CancellationTokenSource();
    cts.CancelAfter(TimeSpan.FromSeconds(5)); // Cancel after 5 seconds
    
    var account = new Account { Name = "Contoso" };
    
    // ACT
    var accountId = await service.CreateAsync(account, cts.Token);
    
    // ASSERT
    Assert.NotEqual(Guid.Empty, accountId);
}

[Fact]
public async Task Should_Throw_When_Operation_Cancelled()
{
    // ARRANGE
    var service = _context.GetAsyncOrganizationService();
    var cts = new CancellationTokenSource();
    cts.Cancel(); // Already cancelled
    
    var account = new Account { Name = "Contoso" };
    
    // ACT & ASSERT
    await Assert.ThrowsAsync<OperationCanceledException>(async () =>
        await service.CreateAsync(account, cts.Token)
    );
}
```

## Parallel Async Operations

```csharp
[Fact]
public async Task Should_Create_Multiple_Records_In_Parallel()
{
    // ARRANGE
    var service = _context.GetAsyncOrganizationService();
    
    var accounts = Enumerable.Range(1, 10)
        .Select(i => new Account { Name = $"Account {i}" })
        .ToList();
    
    // ACT
    var createTasks = accounts.Select(account => service.CreateAsync(account));
    var accountIds = await Task.WhenAll(createTasks);
    
    // ASSERT
    Assert.Equal(10, accountIds.Length);
    Assert.All(accountIds, id => Assert.NotEqual(Guid.Empty, id));
    
    var created = _context.CreateQuery<Account>().ToList();
    Assert.Equal(10, created.Count);
}
```

## Azure Function Integration

```csharp
public class DataverseAsyncFunction
{
    private readonly IOrganizationServiceAsync2 _service;
    
    public DataverseAsyncFunction(IOrganizationServiceFactory factory)
    {
        _service = (IOrganizationServiceAsync2)factory.CreateOrganizationService(null);
    }
    
    [Function("ProcessAccount")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req)
    {
        var accountId = await _service.CreateAsync(new Entity("account"));
        var account = await _service.RetrieveAsync("account", accountId, new ColumnSet(true));
        
        // Process account...
        
        return req.CreateResponse(HttpStatusCode.OK);
    }
}

[Fact]
public async Task Should_Process_Account_Asynchronously_In_Function()
{
    // ARRANGE
    var service = _context.GetAsyncOrganizationService2();
    var factoryMock = new Mock<IOrganizationServiceFactory>();
    factoryMock.Setup(f => f.CreateOrganizationService(It.IsAny<Guid?>()))
        .Returns(service);
    
    var function = new DataverseAsyncFunction(factoryMock.Object);
    var requestMock = new Mock<HttpRequestData>(Mock.Of<FunctionContext>());
    // ... setup request mock ...
    
    // ACT
    var response = await function.Run(requestMock.Object);
    
    // ASSERT
    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    var accounts = _context.CreateQuery<Account>().ToList();
    Assert.Single(accounts);
}
```

## Best Practices

1. **Use IOrganizationServiceAsync2** - Provides RetrieveMultiple for queries
2. **Prefer async/await over Task.Result** - Avoids deadlocks and improves performance
3. **Use CancellationToken** - Allows operation cancellation for long-running tasks
4. **Test parallel operations** - Verify thread safety with Task.WhenAll
5. **Handle exceptions properly** - Use try/catch with async operations
6. **Use ConfigureAwait(false)** - In library code to avoid context capturing
7. **Test timeout scenarios** - Verify behavior when operations timeout
8. **Avoid async void** - Use async Task instead

## Common Patterns

### Async Data Loading

```csharp
public async Task<List<Account>> LoadAccountsAsync(IOrganizationServiceAsync2 service)
{
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet(true);
    
    var results = await service.RetrieveMultipleAsync(query);
    return results.Entities.Select(e => e.ToEntity<Account>()).ToList();
}
```

### Async Batch Processing

```csharp
public async Task ProcessBatchAsync(
    IOrganizationServiceAsync service, 
    List<Entity> entities,
    int batchSize = 100)
{
    for (int i = 0; i < entities.Count; i += batchSize)
    {
        var batch = entities.Skip(i).Take(batchSize);
        var tasks = batch.Select(e => service.CreateAsync(e));
        await Task.WhenAll(tasks);
    }
}
```

### Async Error Handling

```csharp
public async Task<Entity> SafeRetrieveAsync(
    IOrganizationServiceAsync service,
    string entityName,
    Guid id)
{
    try
    {
        return await service.RetrieveAsync(entityName, id, new ColumnSet(true));
    }
    catch (Exception ex)
    {
        // Log error
        throw new ApplicationException($"Failed to retrieve {entityName} {id}", ex);
    }
}
```

## Limitations

- **Plugin context is synchronous** - Dataverse plugins execute synchronously regardless of async code
- **v2.x doesn't support async** - .NET Framework limitation
- **No async transactions** - Transaction scope doesn't support async operations
- **Middleware compatibility** - Some middleware may not support async operations

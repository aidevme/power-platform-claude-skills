# Bulk Operations Testing

Bulk operations (CreateMultiple, UpdateMultiple, UpsertMultiple) are performance-optimized messages
introduced in Dataverse to process multiple records in a single request. According to Microsoft,
bulk operations are the **preferred approach** for plugin logic that processes multiple records.

**Minimum Version Required:** FakeXrmEasy 2.5.0+ (Framework) or 3.5.0+ (.NET Core)

## Why Bulk Operations?

Traditional approach (inefficient):
```csharp
// DON'T DO THIS - Multiple round trips
foreach (var account in accounts)
{
    service.Create(account);
}
```

Bulk operations approach (efficient):
```csharp
// DO THIS - Single round trip
var request = new CreateMultipleRequest
{
    Targets = new EntityCollection(accounts)
};
service.Execute(request);
```

**Benefits:**
- Fewer round trips to Dataverse
- Better performance (up to 10x faster)
- Reduced transaction overhead
- Lower API limits consumption

## Setup for Bulk Operations

Bulk operations are automatically available when using `.AddCrud()` and `.UseCrud()` in middleware:

```csharp
public class BulkOperationsTestsBase : FakeXrmEasyTestsBase
{
    public BulkOperationsTestsBase()
    {
        // No special setup needed - bulk operations enabled by default
        // when you use AddCrud() and UseCrud()
    }
}
```

## Testing CreateMultiple

Creates multiple records in a single request.

### Basic CreateMultiple Test

```csharp
[Fact]
public void Should_Create_Multiple_Accounts()
{
    // ARRANGE
    var account1 = new Account { Name = "Contoso" };
    var account2 = new Account { Name = "Fabrikam" };
    var account3 = new Account { Name = "Adventure Works" };
    
    var request = new CreateMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { account1, account2, account3 })
    };
    
    // ACT
    var response = (CreateMultipleResponse)_service.Execute(request);
    
    // ASSERT
    Assert.NotNull(response);
    Assert.Equal(3, response.Ids.Count);
    
    var accounts = _context.CreateQuery<Account>().ToList();
    Assert.Equal(3, accounts.Count);
    Assert.Contains(accounts, a => a.Name == "Contoso");
    Assert.Contains(accounts, a => a.Name == "Fabrikam");
    Assert.Contains(accounts, a => a.Name == "Adventure Works");
}
```

### CreateMultiple with Different Entity Types

```csharp
[Fact]
public void Should_Create_Multiple_Entity_Types()
{
    // ARRANGE
    var account = new Account { Name = "Contoso" };
    var contact = new Contact { FirstName = "John", LastName = "Doe" };
    
    var accountRequest = new CreateMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { account })
    };
    
    var contactRequest = new CreateMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { contact })
    };
    
    // ACT
    var accountResponse = (CreateMultipleResponse)_service.Execute(accountRequest);
    var contactResponse = (CreateMultipleResponse)_service.Execute(contactRequest);
    
    // ASSERT
    Assert.Single(accountResponse.Ids);
    Assert.Single(contactResponse.Ids);
    
    Assert.Single(_context.CreateQuery<Account>());
    Assert.Single(_context.CreateQuery<Contact>());
}
```

## Testing UpdateMultiple

Updates multiple existing records in a single request. Records **must exist** or an exception is thrown.

### Basic UpdateMultiple Test

```csharp
[Fact]
public void Should_Update_Multiple_Accounts()
{
    // ARRANGE: Create existing records
    var account1Id = Guid.NewGuid();
    var account2Id = Guid.NewGuid();
    
    var existing1 = new Account { Id = account1Id, Name = "Old Name 1", Revenue = new Money(100000) };
    var existing2 = new Account { Id = account2Id, Name = "Old Name 2", Revenue = new Money(200000) };
    
    _context.Initialize(new Entity[] { existing1, existing2 });
    
    // Create update targets (only changed fields)
    var update1 = new Account { Id = account1Id, Revenue = new Money(150000) };
    var update2 = new Account { Id = account2Id, Revenue = new Money(250000) };
    
    var request = new UpdateMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { update1, update2 })
    };
    
    // ACT
    _service.Execute(request);
    
    // ASSERT
    var updated = _context.CreateQuery<Account>().ToList();
    
    var updatedAccount1 = updated.First(a => a.Id == account1Id);
    var updatedAccount2 = updated.First(a => a.Id == account2Id);
    
    // Revenue updated
    Assert.Equal(150000m, updatedAccount1.Revenue.Value);
    Assert.Equal(250000m, updatedAccount2.Revenue.Value);
    
    // Names unchanged
    Assert.Equal("Old Name 1", updatedAccount1.Name);
    Assert.Equal("Old Name 2", updatedAccount2.Name);
}
```

### UpdateMultiple with Alternate Keys

```csharp
[Fact]
public void Should_Update_Using_Alternate_Key()
{
    // ARRANGE: Setup entity with alternate key
    var accountNumber = "ACC-001";
    var accountId = Guid.NewGuid();
    
    var existing = new Account 
    { 
        Id = accountId, 
        AccountNumber = accountNumber,
        Name = "Contoso" 
    };
    
    _context.Initialize(new[] { existing });
    
    // Update using alternate key (not primary key)
    var update = new Account 
    { 
        KeyAttributes = new KeyAttributeCollection
        {
            { "accountnumber", accountNumber }
        },
        Name = "Contoso Updated"
    };
    
    var request = new UpdateMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { update })
    };
    
    // ACT
    _service.Execute(request);
    
    // ASSERT
    var updated = _context.CreateQuery<Account>().Single();
    Assert.Equal("Contoso Updated", updated.Name);
    Assert.Equal(accountNumber, updated.AccountNumber);
}
```

### UpdateMultiple Exception Handling

```csharp
[Fact]
public void Should_Throw_When_Record_Does_Not_Exist()
{
    // ARRANGE: Update non-existent record
    var nonExistentId = Guid.NewGuid();
    var update = new Account { Id = nonExistentId, Name = "Updated" };
    
    var request = new UpdateMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { update })
    };
    
    // ACT & ASSERT
    Assert.Throws<Exception>(() => _service.Execute(request));
}
```

## Testing UpsertMultiple

Upserts (create or update) multiple records. Creates records that don't exist, updates those that do.

### Basic UpsertMultiple Test

```csharp
[Fact]
public void Should_Upsert_Multiple_Accounts_Create_And_Update()
{
    // ARRANGE: One existing, one new
    var existingId = Guid.NewGuid();
    var existing = new Account { Id = existingId, Name = "Existing", Revenue = new Money(100000) };
    _context.Initialize(new[] { existing });
    
    // Update existing account
    var updateAccount = new Account { Id = existingId, Revenue = new Money(200000) };
    
    // Create new account
    var newAccount = new Account { Id = Guid.NewGuid(), Name = "New Account" };
    
    var request = new UpsertMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { updateAccount, newAccount })
    };
    
    // ACT
    var response = (UpsertMultipleResponse)_service.Execute(request);
    
    // ASSERT
    Assert.NotNull(response);
    
    var accounts = _context.CreateQuery<Account>().ToList();
    Assert.Equal(2, accounts.Count);
    
    // Existing account updated
    var updated = accounts.First(a => a.Id == existingId);
    Assert.Equal("Existing", updated.Name); // Name unchanged
    Assert.Equal(200000m, updated.Revenue.Value); // Revenue updated
    
    // New account created
    var created = accounts.First(a => a.Name == "New Account");
    Assert.NotNull(created);
}
```

### UpsertMultiple Response Details

```csharp
[Fact]
public void Should_Return_Upsert_Results()
{
    // ARRANGE
    var existingId = Guid.NewGuid();
    _context.Initialize(new[] { new Account { Id = existingId, Name = "Existing" } });
    
    var updateAccount = new Account { Id = existingId, Name = "Updated" };
    var newAccount = new Account { Id = Guid.NewGuid(), Name = "New" };
    
    var request = new UpsertMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { updateAccount, newAccount })
    };
    
    // ACT
    var response = (UpsertMultipleResponse)_service.Execute(request);
    
    // ASSERT
    Assert.Equal(2, response.Results.Count);
    
    // First was an update (RecordCreated = false)
    Assert.False(response.Results[0].RecordCreated);
    
    // Second was a create (RecordCreated = true)
    Assert.True(response.Results[1].RecordCreated);
}
```

## Testing Plugins for Bulk Operations

Plugins that fire on bulk operations use a new interface: **IPluginExecutionContext4**

### IPluginExecutionContext4 Features

New properties for bulk operations:
- `EntityCollection Targets` - Multiple target entities (instead of single Target)
- `EntityImageCollection PreEntityImagesCollection` - PreImages for each target
- `EntityImageCollection PostEntityImagesCollection` - PostImages for each target

### Testing CreateMultiple Plugin

```csharp
public class CreateMultiplePlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext4)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        if (context.MessageName == "CreateMultiple")
        {
            var targets = context.InputParameters["Targets"] as EntityCollection;
            
            foreach (var target in targets.Entities)
            {
                // Process each target
                if (target.LogicalName == "account" && !target.Contains("accountnumber"))
                {
                    target["accountnumber"] = $"ACC-{Guid.NewGuid().ToString().Substring(0, 8).ToUpper()}";
                }
            }
        }
    }
}

[Fact]
public void Should_Set_Account_Numbers_On_CreateMultiple()
{
    // ARRANGE
    var account1 = new Account { Name = "Account 1" };
    var account2 = new Account { Name = "Account 2" };
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "CreateMultiple";
    ctx.InputParameters["Targets"] = new EntityCollection(new Entity[] { account1, account2 });
    
    // ACT
    _context.ExecutePluginWith<CreateMultiplePlugin>(ctx);
    
    // ASSERT
    Assert.NotNull(account1.AccountNumber);
    Assert.NotNull(account2.AccountNumber);
    Assert.StartsWith("ACC-", account1.AccountNumber);
    Assert.StartsWith("ACC-", account2.AccountNumber);
}
```

### Testing UpdateMultiple Plugin with PreImages

```csharp
public class UpdateMultipleAuditPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext4)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        if (context.MessageName == "UpdateMultiple" && context.PreEntityImagesCollection.Count > 0)
        {
            var targets = context.InputParameters["Targets"] as EntityCollection;
            
            for (int i = 0; i < targets.Entities.Count; i++)
            {
                var target = targets.Entities[i];
                var preImage = context.PreEntityImagesCollection[i]["PreImage"];
                
                // Compare old vs new values
                if (target.Contains("revenue") && preImage.Contains("revenue"))
                {
                    var oldRevenue = preImage.GetAttributeValue<Money>("revenue");
                    var newRevenue = target.GetAttributeValue<Money>("revenue");
                    
                    // Log change (in real plugin would create audit record)
                    target["lastrevenueupdatedon"] = DateTime.UtcNow;
                }
            }
        }
    }
}

[Fact]
public void Should_Track_Revenue_Changes_In_UpdateMultiple()
{
    // ARRANGE: Existing accounts
    var account1Id = Guid.NewGuid();
    var account2Id = Guid.NewGuid();
    
    _context.Initialize(new Entity[]
    {
        new Account { Id = account1Id, Name = "Account 1", Revenue = new Money(100000) },
        new Account { Id = account2Id, Name = "Account 2", Revenue = new Money(200000) }
    });
    
    // Create updates
    var update1 = new Account { Id = account1Id, Revenue = new Money(150000) };
    var update2 = new Account { Id = account2Id, Revenue = new Money(250000) };
    
    // Setup context with PreImages
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "UpdateMultiple";
    ctx.InputParameters["Targets"] = new EntityCollection(new Entity[] { update1, update2 });
    
    // FakeXrmEasy automatically provides PreEntityImagesCollection
    
    // ACT
    _context.ExecutePluginWith<UpdateMultipleAuditPlugin>(ctx);
    
    // ASSERT
    Assert.True(update1.Contains("lastrevenueupdatedon"));
    Assert.True(update2.Contains("lastrevenueupdatedon"));
}
```

## Bulk Operations with Pipeline Simulation

Register plugins that fire automatically on bulk operations:

```csharp
[Fact]
public void Should_Fire_Plugin_On_CreateMultiple_Via_Pipeline()
{
    // ARRANGE: Setup pipeline simulation
    _context = MiddlewareBuilder.New()
        .AddCrud()
        .AddPipelineSimulation()
        .UsePipelineSimulation()
        .UseCrud()
        .Build();
    
    // Register plugin for CreateMultiple
    _context.RegisterPluginStep<AccountNumberPlugin>(new PluginStepDefinition
    {
        EntityLogicalName = "account",
        MessageName = "CreateMultiple",
        Stage = ProcessingStepStage.Preoperation
    });
    
    var account1 = new Account { Name = "Account 1" };
    var account2 = new Account { Name = "Account 2" };
    
    var request = new CreateMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { account1, account2 })
    };
    
    // ACT: Plugin fires automatically
    var response = (CreateMultipleResponse)_service.Execute(request);
    
    // ASSERT: Plugin set account numbers
    var accounts = _context.CreateQuery<Account>().ToList();
    Assert.All(accounts, a => Assert.NotNull(a.AccountNumber));
}
```

## Performance Testing with Bulk Operations

```csharp
[Fact]
public void Bulk_Operations_Should_Be_Faster_Than_Individual_Operations()
{
    // ARRANGE: 100 accounts
    var accounts = Enumerable.Range(1, 100)
        .Select(i => new Account { Name = $"Account {i}" })
        .ToList();
    
    // Individual creates
    var individualStopwatch = Stopwatch.StartNew();
    foreach (var account in accounts)
    {
        _service.Create(account);
    }
    individualStopwatch.Stop();
    
    // Clear for bulk test
    _context = MiddlewareBuilder.New().AddCrud().UseCrud().Build();
    _service = _context.GetOrganizationService();
    
    // Bulk create
    var bulkStopwatch = Stopwatch.StartNew();
    var request = new CreateMultipleRequest
    {
        Targets = new EntityCollection(accounts)
    };
    _service.Execute(request);
    bulkStopwatch.Stop();
    
    // ASSERT: Bulk is faster (in real Dataverse, much more dramatic)
    // In FakeXrmEasy, both are fast, but bulk should still be comparable
    Assert.True(bulkStopwatch.ElapsedMilliseconds <= individualStopwatch.ElapsedMilliseconds);
}
```

## Best Practices

1. **Use bulk operations for 10+ records** - Single operations are fine for small batches
2. **Batch size: 100-1000 records** - Balance between efficiency and transaction size
3. **Handle partial failures** - In real Dataverse, some records may succeed while others fail
4. **Test with realistic volumes** - If production processes 500 records, test with 500
5. **Use PreImages sparingly** - Only register images for attributes you actually need

## Common Patterns

### Bulk Create with Error Handling

```csharp
[Fact]
public void Should_Handle_Mixed_Valid_Invalid_Records()
{
    // ARRANGE: Some valid, some invalid
    var validAccount = new Account { Name = "Valid Account" };
    var invalidAccount = new Account { }; // Missing required Name
    
    var request = new CreateMultipleRequest
    {
        Targets = new EntityCollection(new Entity[] { validAccount, invalidAccount })
    };
    
    // ACT & ASSERT
    // FakeXrmEasy validates all records
    var ex = Assert.Throws<Exception>(() => _service.Execute(request));
    
    // In production, you might want ContinueOnError behavior
    // Check Microsoft docs for ContinueOnError parameter
}
```

### Converting Individual Operations to Bulk

Before (inefficient):
```csharp
public void UpdateAccountRevenues(List<Guid> accountIds, decimal newRevenue)
{
    foreach (var id in accountIds)
    {
        _service.Update(new Account { Id = id, Revenue = new Money(newRevenue) });
    }
}
```

After (efficient):
```csharp
public void UpdateAccountRevenues(List<Guid> accountIds, decimal newRevenue)
{
    var updates = accountIds.Select(id => new Account 
    { 
        Id = id, 
        Revenue = new Money(newRevenue) 
    }).ToArray();
    
    var request = new UpdateMultipleRequest
    {
        Targets = new EntityCollection(updates)
    };
    
    _service.Execute(request);
}
```

## Limitations

1. **DeleteMultiple** - Not yet supported in Dataverse (coming soon)
2. **Transaction boundaries** - Entire bulk operation is one transaction
3. **Plugin depth** - Bulk operations count toward depth limits
4. **API limits** - Still count against API request limits (but fewer requests overall)

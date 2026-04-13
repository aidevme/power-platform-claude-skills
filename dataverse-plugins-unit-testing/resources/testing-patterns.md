# Testing Patterns

This guide covers common testing patterns for Dataverse plugins using FakeXrmEasy's AAA
(Arrange-Act-Assert) approach.

## The AAA Pattern

Every good unit test follows three phases:

1. **Arrange** — Set up test data, configure the plugin context, initialize the in-memory database
2. **Act** — Execute the plugin against the test context
3. **Assert** — Verify expected outcomes by querying the context or inspecting entities

## Basic Plugin Execution

### ExecutePluginWith — Full Control

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
- Testing conditional logic based on message or stage

### ExecutePluginWithTarget — Simple Scenarios

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

**Limitations:**
- Can't control Stage, MessageName, or other context properties
- Uses default plugin context
- Less representative of real plugin execution

## Assertion Strategies

### Asserting on Target Entity

For PreOperation plugins that modify the Target without calling `service.Update()`:

```csharp
[Fact]
public void PreOperation_Update_Should_Modify_Target_Entity()
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
    
    // ASSERT
    // Check the Target reference - PreOperation changes don't update the DB yet
    Assert.True(updateTarget.Attributes.ContainsKey("description"));
    Assert.Equal("Auto-generated", updateTarget.Description);
}
```

**Critical:** Assert against the same object reference you passed as Target. PreOperation changes
won't be in the database until the transaction completes.

### Asserting on Database State

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
    
    // ASSERT
    var tasks = _context.CreateQuery<Task>()
        .Where(t => t.RegardingObjectId.Id == accountId)
        .ToList();
    
    Assert.Single(tasks);
    Assert.Equal("Follow up with new account", tasks[0].Subject);
}
```

**When to use:**
- PostOperation plugins that create/update entities via IOrganizationService
- Plugins that call service.Create(), service.Update(), etc.
- Verifying side effects

### Asserting Exceptions

Test that plugins throw the correct exceptions with meaningful messages:

```csharp
[Fact]
public void When_Email_Invalid_Should_Throw_InvalidPluginExecutionException()
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

**Best Practice:** Always verify the exception message. Users see this message in the UI.

## Testing Different Messages

### Create Message

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
    
    // Assertions...
}
```

### Update Message

```csharp
[Fact]
public void Update_Message_Test()
{
    // Setup existing record
    var accountId = Guid.NewGuid();
    var existing = new Account { Id = accountId, Name = "Old Name", Revenue = new Money(100000) };
    _context.Initialize(new[] { existing });
    
    // Create update target with only changed fields
    var target = new Account { Id = accountId, Revenue = new Money(200000) };
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Update";
    ctx.Stage = 20; // PreOperation
    ctx.PrimaryEntityName = "account";
    ctx.PrimaryEntityId = accountId;
    ctx.InputParameters["Target"] = target;
    
    _context.ExecutePluginWith<MyPlugin>(ctx);
    
    // Assertions...
}
```

**Note:** Update Target only contains changed attributes, not the full record. Use PreImage to
access unchanged values.

### Delete Message

```csharp
[Fact]
public void Delete_Message_Test()
{
    // Setup record to delete
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    var entityRef = new EntityReference("account", accountId);
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Delete";
    ctx.Stage = 20; // PreOperation
    ctx.PrimaryEntityName = "account";
    ctx.PrimaryEntityId = accountId;
    ctx.InputParameters["Target"] = entityRef;
    
    _context.ExecutePluginWith<MyPlugin>(ctx);
    
    // Assertions...
}
```

**Note:** Delete message has an EntityReference as Target, not an Entity.

### Associate/Disassociate Messages

```csharp
[Fact]
public void Associate_Message_Test()
{
    var accountId = Guid.NewGuid();
    var contactId = Guid.NewGuid();
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Associate";
    ctx.Stage = 40; // PostOperation
    ctx.PrimaryEntityName = "account";
    ctx.PrimaryEntityId = accountId;
    ctx.InputParameters["Target"] = new EntityReference("account", accountId);
    ctx.InputParameters["Relationship"] = new Relationship("account_primary_contact");
    ctx.InputParameters["RelatedEntities"] = new EntityReferenceCollection
    {
        new EntityReference("contact", contactId)
    };
    
    _context.ExecutePluginWith<MyPlugin>(ctx);
    
    // Assertions...
}
```

## Testing with In-Memory Data

### Initialize Test Data

Use `Initialize()` to seed the in-memory database:

```csharp
[Fact]
public void Test_With_Related_Records()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var contact1Id = Guid.NewGuid();
    var contact2Id = Guid.NewGuid();
    
    var account = new Account { Id = accountId, Name = "Contoso" };
    var contact1 = new Contact { Id = contact1Id, LastName = "Smith", ParentCustomerId = account.ToEntityReference() };
    var contact2 = new Contact { Id = contact2Id, LastName = "Jones", ParentCustomerId = account.ToEntityReference() };
    
    _context.Initialize(new Entity[] { account, contact1, contact2 });
    
    // Now plugin can retrieve related contacts
    var target = new Account { Id = accountId, Name = "Contoso Updated" };
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Update";
    ctx.InputParameters["Target"] = target;
    
    // ACT
    _context.ExecutePluginWith<UpdateRelatedContactsPlugin>(ctx);
    
    // ASSERT
    var contacts = _context.CreateQuery<Contact>()
        .Where(c => c.ParentCustomerId.Id == accountId)
        .ToList();
    
    Assert.Equal(2, contacts.Count);
    Assert.All(contacts, c => Assert.True(c.DoNotEmail == true));
}
```

### Query In-Memory Data

Use LINQ to query test data:

```csharp
// Early-bound
var activeAccounts = _context.CreateQuery<Account>()
    .Where(a => a.StateCode == AccountState.Active)
    .ToList();

// Late-bound
var activeAccounts = _context.CreateQuery("account")
    .Where(a => ((OptionSetValue)a["statecode"]).Value == 0)
    .ToList();
```

## Testing with SharedVariables

Test plugins that communicate via SharedVariables:

```csharp
[Fact]
public void Should_Set_SharedVariable_For_Downstream_Plugins()
{
    // ARRANGE
    var target = new Account { Id = Guid.NewGuid(), Name = "Contoso" };
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Create";
    ctx.InputParameters["Target"] = target;
    ctx.SharedVariables = new ParameterCollection();
    
    // ACT
    _context.ExecutePluginWith<UpstreamPlugin>(ctx);
    
    // ASSERT
    Assert.True(ctx.SharedVariables.Contains("SkipValidation"));
    Assert.True((bool)ctx.SharedVariables["SkipValidation"]);
}
```

## Testing Multi-Step Plugins

When one plugin calls the service and triggers another plugin:

```csharp
[Fact]
public void Should_Not_Cause_Infinite_Loop()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Update";
    ctx.Depth = 1; // Simulate first level
    ctx.InputParameters["Target"] = new Account { Id = accountId, Revenue = new Money(100000) };
    
    // ACT
    _context.ExecutePluginWith<RecursiveUpdatePlugin>(ctx);
    
    // ASSERT
    // Verify plugin respected Depth check and didn't recurse infinitely
    var updated = _context.CreateQuery<Account>().First(a => a.Id == accountId);
    Assert.NotNull(updated.Revenue);
}
```

**Best Practice:** Always check `context.Depth` in plugins that call the service to prevent
infinite loops. FakeXrmEasy doesn't automatically prevent recursion.

## Theory/Parameterized Tests

Test the same plugin with multiple inputs:

```csharp
[Theory]
[InlineData("test@contoso.com", true)]
[InlineData("invalid-email", false)]
[InlineData("", false)]
[InlineData(null, false)]
public void Email_Validation_Tests(string email, bool shouldSucceed)
{
    // ARRANGE
    var contact = new Contact { Id = Guid.NewGuid(), EmailAddress1 = email };
    var ctx = _context.GetDefaultPluginContext();
    ctx.MessageName = "Create";
    ctx.InputParameters["Target"] = contact;
    
    // ACT & ASSERT
    if (shouldSucceed)
    {
        _context.ExecutePluginWith<EmailValidationPlugin>(ctx);
        // Success - no exception
    }
    else
    {
        Assert.Throws<InvalidPluginExecutionException>(() =>
            _context.ExecutePluginWith<EmailValidationPlugin>(ctx));
    }
}
```

## Test Naming Conventions

Use descriptive test names that explain the scenario:

```csharp
// Good
[Fact] public void When_Account_Created_With_Valid_Email_Should_Send_Welcome_Email() { }
[Fact] public void When_Contact_Updated_Without_Email_Should_Throw_Exception() { }
[Fact] public void When_Opportunity_Closed_Should_Create_FollowUp_Task() { }

// Bad
[Fact] public void Test1() { }
[Fact] public void AccountTest() { }
[Fact] public void TestPlugin() { }
```

Convention: `When_[Condition]_Should_[ExpectedBehavior]`

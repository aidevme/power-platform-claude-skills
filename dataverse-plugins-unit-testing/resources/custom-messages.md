# Custom Messages Testing

Test custom actions, custom APIs, and generic message executors in FakeXrmEasy to verify
business logic in custom Dataverse operations.

## Overview

Custom message types in Dataverse:
- **Custom Actions** - Legacy workflow-based custom operations
- **Custom APIs** - Modern plugin-based custom operations (preferred)
- **Generic Message Executors** - Test any OrganizationRequest/Response

## Custom Actions Testing

### Simple Custom Action (No Parameters)

```csharp
// Custom Action: new_SendWelcomeEmail
// Input: EntityReference Target (contact)
// Output: bool Success

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
        
        // Send email logic...
        bool success = !string.IsNullOrEmpty(email);
        
        context.OutputParameters["Success"] = success;
    }
}

[Fact]
public void Should_Execute_SendWelcomeEmail_Custom_Action()
{
    // ARRANGE
    var contactId = Guid.NewGuid();
    var contact = new Contact
    {
        Id = contactId,
        FirstName = "John",
        EMailAddress1 = "john@example.com"
    };
    _context.Initialize(new[] { contact });
    
    // Register plugin for custom action message
    _context.RegisterPluginStep<SendWelcomeEmailPlugin>(
        "new_SendWelcomeEmail",
        executionStage: ProcessingStepStage.Postoperation,
        executionMode: ProcessingStepMode.Synchronous
    );
    
    // ACT
    var request = new OrganizationRequest("new_SendWelcomeEmail");
    request["Target"] = contact.ToEntityReference();
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.True((bool)response["Success"]);
}
```

### Custom Action with Multiple Parameters

```csharp
// Custom Action: new_CalculateDiscount
// Input: decimal BaseAmount, string CustomerType
// Output: decimal DiscountAmount, decimal FinalAmount

public class CalculateDiscountPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        var baseAmount = (decimal)context.InputParameters["BaseAmount"];
        var customerType = (string)context.InputParameters["CustomerType"];
        
        decimal discountPercent = customerType switch
        {
            "VIP" => 0.20m,
            "Gold" => 0.15m,
            "Silver" => 0.10m,
            _ => 0.05m
        };
        
        var discountAmount = baseAmount * discountPercent;
        var finalAmount = baseAmount - discountAmount;
        
        context.OutputParameters["DiscountAmount"] = discountAmount;
        context.OutputParameters["FinalAmount"] = finalAmount;
    }
}

[Theory]
[InlineData(1000.00, "VIP", 200.00, 800.00)]
[InlineData(1000.00, "Gold", 150.00, 850.00)]
[InlineData(1000.00, "Silver", 100.00, 900.00)]
[InlineData(1000.00, "Standard", 50.00, 950.00)]
public void Should_Calculate_Discount_Based_On_Customer_Type(
    decimal baseAmount,
    string customerType,
    decimal expectedDiscount,
    decimal expectedFinal)
{
    // ARRANGE
    _context.RegisterPluginStep<CalculateDiscountPlugin>(
        "new_CalculateDiscount",
        executionStage: ProcessingStepStage.Postoperation,
        executionMode: ProcessingStepMode.Synchronous
    );
    
    // ACT
    var request = new OrganizationRequest("new_CalculateDiscount");
    request["BaseAmount"] = baseAmount;
    request["CustomerType"] = customerType;
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.Equal(expectedDiscount, (decimal)response["DiscountAmount"]);
    Assert.Equal(expectedFinal, (decimal)response["FinalAmount"]);
}
```

### Custom Action with Entity Collection

```csharp
// Custom Action: new_ProcessOrders
// Input: EntityCollection Orders
// Output: int ProcessedCount, EntityCollection FailedOrders

public class ProcessOrdersPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var serviceFactory = (IOrganizationServiceFactory)serviceProvider.GetService(typeof(IOrganizationServiceFactory));
        var service = serviceFactory.CreateOrganizationService(context.UserId);
        
        var orders = (EntityCollection)context.InputParameters["Orders"];
        var failedOrders = new EntityCollection();
        int processedCount = 0;
        
        foreach (var order in orders.Entities)
        {
            try
            {
                // Process order
                order["statuscode"] = new OptionSetValue(100000); // Processed
                service.Update(order);
                processedCount++;
            }
            catch
            {
                failedOrders.Entities.Add(order);
            }
        }
        
        context.OutputParameters["ProcessedCount"] = processedCount;
        context.OutputParameters["FailedOrders"] = failedOrders;
    }
}

[Fact]
public void Should_Process_Order_Collection()
{
    // ARRANGE
    var order1 = new Entity("order") { Id = Guid.NewGuid() };
    var order2 = new Entity("order") { Id = Guid.NewGuid() };
    var order3 = new Entity("order") { Id = Guid.NewGuid() };
    
    _context.Initialize(new[] { order1, order2, order3 });
    
    _context.RegisterPluginStep<ProcessOrdersPlugin>(
        "new_ProcessOrders",
        executionStage: ProcessingStepStage.Postoperation,
        executionMode: ProcessingStepMode.Synchronous
    );
    
    var orderCollection = new EntityCollection();
    orderCollection.Entities.AddRange(new[] { order1, order2, order3 });
    
    // ACT
    var request = new OrganizationRequest("new_ProcessOrders");
    request["Orders"] = orderCollection;
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.Equal(3, (int)response["ProcessedCount"]);
    var failedOrders = (EntityCollection)response["FailedOrders"];
    Assert.Empty(failedOrders.Entities);
}
```

## Custom APIs Testing

Custom APIs are the modern replacement for custom actions (introduced in 2020).

### Simple Custom API

```csharp
// Custom API: new_ValidateAccount
// Input: Guid AccountId
// Output: bool IsValid, string ValidationMessage

public class ValidateAccountApiPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var serviceFactory = (IOrganizationServiceFactory)serviceProvider.GetService(typeof(IOrganizationServiceFactory));
        var service = serviceFactory.CreateOrganizationService(context.UserId);
        
        var accountId = (Guid)context.InputParameters["AccountId"];
        
        var account = service.Retrieve("account", accountId, new ColumnSet("name", "creditonhold"));
        
        bool isValid = !account.GetAttributeValue<bool>("creditonhold");
        string message = isValid 
            ? "Account is valid" 
            : "Account credit is on hold";
        
        context.OutputParameters["IsValid"] = isValid;
        context.OutputParameters["ValidationMessage"] = message;
    }
}

[Fact]
public void Should_Validate_Account_Via_Custom_API()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account
    {
        Id = accountId,
        Name = "Contoso",
        CreditOnHold = false
    };
    _context.Initialize(new[] { account });
    
    _context.RegisterPluginStep<ValidateAccountApiPlugin>(
        "new_ValidateAccount",
        executionStage: ProcessingStepStage.Postoperation,
        executionMode: ProcessingStepMode.Synchronous
    );
    
    // ACT
    var request = new OrganizationRequest("new_ValidateAccount");
    request["AccountId"] = accountId;
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.True((bool)response["IsValid"]);
    Assert.Equal("Account is valid", (string)response["ValidationMessage"]);
}
```

### Custom API with Complex Return Type

```csharp
// Custom API: new_GetAccountSummary
// Input: Guid AccountId
// Output: string JsonSummary

public class GetAccountSummaryApiPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var serviceFactory = (IOrganizationServiceFactory)serviceProvider.GetService(typeof(IOrganizationServiceFactory));
        var service = serviceFactory.CreateOrganizationService(context.UserId);
        
        var accountId = (Guid)context.InputParameters["AccountId"];
        
        var account = service.Retrieve("account", accountId, new ColumnSet(true));
        
        // Count related contacts
        var contactQuery = new QueryExpression("contact");
        contactQuery.Criteria.AddCondition("parentcustomerid", ConditionOperator.Equal, accountId);
        var contactCount = service.RetrieveMultiple(contactQuery).Entities.Count;
        
        var summary = new
        {
            AccountId = accountId,
            Name = account.GetAttributeValue<string>("name"),
            ContactCount = contactCount,
            CreatedOn = account.GetAttributeValue<DateTime>("createdon")
        };
        
        context.OutputParameters["JsonSummary"] = JsonSerializer.Serialize(summary);
    }
}

[Fact]
public void Should_Return_Account_Summary_As_Json()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account
    {
        Id = accountId,
        Name = "Contoso",
        CreatedOn = DateTime.Parse("2024-01-01")
    };
    
    var contact1 = new Contact { Id = Guid.NewGuid(), ParentCustomerId = account.ToEntityReference() };
    var contact2 = new Contact { Id = Guid.NewGuid(), ParentCustomerId = account.ToEntityReference() };
    
    _context.Initialize(new Entity[] { account, contact1, contact2 });
    
    _context.RegisterPluginStep<GetAccountSummaryApiPlugin>(
        "new_GetAccountSummary",
        executionStage: ProcessingStepStage.Postoperation,
        executionMode: ProcessingStepMode.Synchronous
    );
    
    // ACT
    var request = new OrganizationRequest("new_GetAccountSummary");
    request["AccountId"] = accountId;
    
    var response = _service.Execute(request);
    
    // ASSERT
    var jsonSummary = (string)response["JsonSummary"];
    Assert.Contains("Contoso", jsonSummary);
    Assert.Contains("\"ContactCount\":2", jsonSummary);
}
```

## Generic Message Executors

Test any custom message without creating specific request/response classes.

### Testing Unknown Messages

```csharp
[Fact]
public void Should_Execute_Generic_Custom_Message()
{
    // ARRANGE
    _context.RegisterPluginStep<MyCustomMessagePlugin>(
        "new_CustomMessage",
        executionStage: ProcessingStepStage.Postoperation,
        executionMode: ProcessingStepMode.Synchronous
    );
    
    // ACT - Use generic OrganizationRequest
    var request = new OrganizationRequest("new_CustomMessage");
    request["InputParam1"] = "Test Value";
    request["InputParam2"] = 42;
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.NotNull(response);
    Assert.Equal("Success", response["Status"]);
}
```

### Mock Custom Message Executor

For testing code that calls custom messages without implementing the plugin:

```csharp
[Fact]
public void Should_Mock_Custom_API_Response()
{
    // ARRANGE - Mock the custom API without implementing plugin
    var mockResponse = new OrganizationResponse();
    mockResponse["IsValid"] = true;
    mockResponse["Message"] = "Validation passed";
    
    // Register mock executor
    _context.AddGenericMessageExecutor("new_ValidateEntity", (request) =>
    {
        // Verify request parameters
        Assert.NotNull(request["EntityId"]);
        
        // Return mock response
        return mockResponse;
    });
    
    // ACT
    var request = new OrganizationRequest("new_ValidateEntity");
    request["EntityId"] = Guid.NewGuid();
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.True((bool)response["IsValid"]);
    Assert.Equal("Validation passed", response["Message"]);
}
```

## Testing Custom API with Binding

### Entity-Bound Custom API

```csharp
// Custom API: new_CloseAccount (bound to account entity)
// This: EntityReference (account)
// Output: bool Success

public class CloseAccountApiPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var serviceFactory = (IOrganizationServiceFactory)serviceProvider.GetService(typeof(IOrganizationServiceFactory));
        var service = serviceFactory.CreateOrganizationService(context.UserId);
        
        var accountRef = (EntityReference)context.InputParameters["Target"];
        
        var account = new Entity("account", accountRef.Id);
        account["statecode"] = new OptionSetValue(1); // Inactive
        account["statuscode"] = new OptionSetValue(2); // Inactive
        
        service.Update(account);
        
        context.OutputParameters["Success"] = true;
    }
}

[Fact]
public void Should_Execute_Entity_Bound_Custom_API()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account
    {
        Id = accountId,
        Name = "Contoso",
        StateCode = AccountState.Active
    };
    _context.Initialize(new[] { account });
    
    _context.RegisterPluginStep<CloseAccountApiPlugin>(
        "new_CloseAccount",
        executionStage: ProcessingStepStage.Postoperation,
        executionMode: ProcessingStepMode.Synchronous
    );
    
    // ACT
    var request = new OrganizationRequest("new_CloseAccount");
    request["Target"] = account.ToEntityReference();
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.True((bool)response["Success"]);
    
    var updated = _service.Retrieve("account", accountId, new ColumnSet("statecode"));
    Assert.Equal(AccountState.Inactive, updated.GetAttributeValue<AccountState>("statecode"));
}
```

### Entity Collection-Bound Custom API

```csharp
// Custom API: new_BulkUpdate (bound to entity collection)
// This: EntityReferenceCollection
// Input: string FieldName, object FieldValue
// Output: int UpdatedCount

[Fact]
public void Should_Execute_EntityCollection_Bound_Custom_API()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Account 1" };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Account 2" };
    _context.Initialize(new[] { account1, account2 });
    
    _context.RegisterPluginStep<BulkUpdateApiPlugin>(
        "new_BulkUpdate",
        executionStage: ProcessingStepStage.Postoperation,
        executionMode: ProcessingStepMode.Synchronous
    );
    
    var entityRefs = new EntityReferenceCollection
    {
        account1.ToEntityReference(),
        account2.ToEntityReference()
    };
    
    // ACT
    var request = new OrganizationRequest("new_BulkUpdate");
    request["Target"] = entityRefs;
    request["FieldName"] = "telephone1";
    request["FieldValue"] = "555-1234";
    
    var response = _service.Execute(request);
    
    // ASSERT
    Assert.Equal(2, (int)response["UpdatedCount"]);
}
```

## Testing Async Custom APIs (v3.x)

```csharp
[Fact]
public async Task Should_Execute_Custom_API_Asynchronously()
{
    // ARRANGE
    var service = _context.GetAsyncOrganizationService();
    
    _context.RegisterPluginStep<MyAsyncApiPlugin>(
        "new_AsyncOperation",
        executionStage: ProcessingStepStage.Postoperation,
        executionMode: ProcessingStepMode.Asynchronous
    );
    
    // ACT
    var request = new OrganizationRequest("new_AsyncOperation");
    request["Data"] = "Test";
    
    var response = await service.ExecuteAsync(request);
    
    // ASSERT
    Assert.NotNull(response);
}
```

## Best Practices

1. **Use descriptive message names** - Follow naming conventions (prefix with publisher)
2. **Test input validation** - Verify required parameters are checked
3. **Test error scenarios** - Missing parameters, invalid types, null values
4. **Use generic OrganizationRequest** - More flexible than strongly-typed requests
5. **Document parameter names** - Custom messages don't have IntelliSense
6. **Test output parameters** - Verify all expected outputs are returned
7. **Mock when appropriate** - Use AddGenericMessageExecutor for external APIs
8. **Test binding types** - Unbound, entity-bound, collection-bound

## Common Patterns

### Request/Response Wrapper

```csharp
public class CustomAPIRequest
{
    public string MessageName { get; set; }
    public Dictionary<string, object> Parameters { get; set; }
    
    public OrganizationRequest ToOrganizationRequest()
    {
        var request = new OrganizationRequest(MessageName);
        foreach (var param in Parameters)
        {
            request[param.Key] = param.Value;
        }
        return request;
    }
}

[Fact]
public void Should_Use_Request_Wrapper()
{
    var customRequest = new CustomAPIRequest
    {
        MessageName = "new_MyAPI",
        Parameters = new Dictionary<string, object>
        {
            { "Param1", "Value1" },
            { "Param2", 123 }
        }
    };
    
    var response = _service.Execute(customRequest.ToOrganizationRequest());
    // Assert...
}
```

### Fluent API Builder

```csharp
public class CustomMessageBuilder
{
    private OrganizationRequest _request;
    
    public CustomMessageBuilder(string messageName)
    {
        _request = new OrganizationRequest(messageName);
    }
    
    public CustomMessageBuilder WithParameter(string name, object value)
    {
        _request[name] = value;
        return this;
    }
    
    public OrganizationRequest Build() => _request;
}

[Fact]
public void Should_Use_Fluent_Builder()
{
    var request = new CustomMessageBuilder("new_MyAPI")
        .WithParameter("AccountId", Guid.NewGuid())
        .WithParameter("Action", "Process")
        .Build();
    
    var response = _service.Execute(request);
    // Assert...
}
```

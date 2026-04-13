# CodeActivities (Workflow Activities) Testing

CodeActivities are custom workflow activities (steps) used in traditional Dataverse workflows.
While workflows are legacy (replaced by Power Automate), many organizations still maintain them.

**Package Required:** `FakeXrmEasy.CodeActivities.v9` (Version 2.x for .NET Framework)

## Overview

CodeActivities:
- Inherit from `CodeActivity` or `CodeActivity<T>`
- Execute within workflow context
- Have InArguments (inputs) and OutArguments (outputs)
- Access IWorkflowContext and IOrganizationService
- Are synchronous only (no async support)

## Setup for CodeActivity Testing

### Install Package

```powershell
Install-Package FakeXrmEasy.CodeActivities.v9 -Version 2.6.3
```

### Base Test Class

```csharp
using FakeXrmEasy;
using FakeXrmEasy.Abstractions;
using Microsoft.Xrm.Sdk.Workflow;
using System.Activities;
using System.Collections.Generic;

public class CodeActivityTestsBase
{
    protected readonly IXrmFakedContext _context;
    protected readonly IOrganizationService _service;
    
    public CodeActivityTestsBase()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .UseCrud()
            .Build();
            
        _service = _context.GetOrganizationService();
    }
    
    protected IDictionary<string, object> ExecuteCodeActivity<T>(
        CodeActivity codeActivity, 
        IDictionary<string, object> inputs) 
        where T : CodeActivity
    {
        return _context.ExecuteCodeActivity<T>(codeActivity, inputs);
    }
}
```

## Testing InArguments and OutArguments

### Simple CodeActivity Example

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
    
    [Output("Total Amount")]
    public OutArgument<decimal> TotalAmount { get; set; }
    
    protected override void Execute(CodeActivityContext context)
    {
        var amount = Amount.Get(context);
        var taxRate = TaxRate.Get(context);
        
        var taxAmount = amount * taxRate;
        var totalAmount = amount + taxAmount;
        
        TaxAmount.Set(context, taxAmount);
        TotalAmount.Set(context, totalAmount);
    }
}
```

### Test with Inputs and Outputs

```csharp
[Fact]
public void Should_Calculate_Tax_Correctly()
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
    Assert.Equal(120.00m, outputs["Total Amount"]);
}

[Fact]
public void Should_Use_Default_Tax_Rate_When_Not_Provided()
{
    // ARRANGE - Only provide Amount, let Tax Rate use default
    var inputs = new Dictionary<string, object>
    {
        { "Amount", 100.00m }
    };
    
    var activity = new CalculateTaxActivity();
    
    // ACT
    var outputs = _context.ExecuteCodeActivity<CalculateTaxActivity>(activity, inputs);
    
    // ASSERT
    Assert.Equal(20.00m, outputs["Tax Amount"]); // 0.20 default rate
}
```

## Testing with IWorkflowContext

CodeActivities can access workflow-specific context:

```csharp
public class GetWorkflowNameActivity : CodeActivity
{
    [Output("Workflow Name")]
    public OutArgument<string> WorkflowName { get; set; }
    
    protected override void Execute(CodeActivityContext context)
    {
        var workflowContext = context.GetExtension<IWorkflowContext>();
        WorkflowName.Set(context, workflowContext.WorkflowName);
    }
}

[Fact]
public void Should_Return_Workflow_Name()
{
    // ARRANGE
    var inputs = new Dictionary<string, object>();
    var activity = new GetWorkflowNameActivity();
    
    // Set workflow context properties
    var workflowContext = _context.GetDefaultWorkflowContext();
    workflowContext.WorkflowName = "Test Workflow";
    
    // ACT
    var outputs = _context.ExecuteCodeActivity<GetWorkflowNameActivity>(
        activity, 
        inputs, 
        workflowContext
    );
    
    // ASSERT
    Assert.Equal("Test Workflow", outputs["Workflow Name"]);
}
```

## Testing with IOrganizationService

CodeActivities that interact with Dataverse:

```csharp
public class CreateFollowUpTaskActivity : CodeActivity
{
    [RequiredArgument]
    [Input("Account")]
    [ReferenceTarget("account")]
    public InArgument<EntityReference> Account { get; set; }
    
    [RequiredArgument]
    [Input("Subject")]
    public InArgument<string> Subject { get; set; }
    
    [Output("Task")]
    [ReferenceTarget("task")]
    public OutArgument<EntityReference> Task { get; set; }
    
    protected override void Execute(CodeActivityContext context)
    {
        var serviceFactory = context.GetExtension<IOrganizationServiceFactory>();
        var service = serviceFactory.CreateOrganizationService(null);
        
        var account = Account.Get(context);
        var subject = Subject.Get(context);
        
        var task = new Entity("task");
        task["subject"] = subject;
        task["regardingobjectid"] = account;
        
        var taskId = service.Create(task);
        
        Task.Set(context, new EntityReference("task", taskId));
    }
}

[Fact]
public void Should_Create_Task_For_Account()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    var inputs = new Dictionary<string, object>
    {
        { "Account", account.ToEntityReference() },
        { "Subject", "Follow up call" }
    };
    
    var activity = new CreateFollowUpTaskActivity();
    
    // ACT
    var outputs = _context.ExecuteCodeActivity<CreateFollowUpTaskActivity>(activity, inputs);
    
    // ASSERT
    var taskRef = (EntityReference)outputs["Task"];
    Assert.NotNull(taskRef);
    Assert.Equal("task", taskRef.LogicalName);
    
    var createdTask = _service.Retrieve("task", taskRef.Id, new ColumnSet(true));
    Assert.Equal("Follow up call", createdTask["subject"]);
    Assert.Equal(accountId, ((EntityReference)createdTask["regardingobjectid"]).Id);
}
```

## Testing Exception Handling

```csharp
public class ValidateAccountActivity : CodeActivity
{
    [RequiredArgument]
    [Input("Account")]
    [ReferenceTarget("account")]
    public InArgument<EntityReference> Account { get; set; }
    
    protected override void Execute(CodeActivityContext context)
    {
        var serviceFactory = context.GetExtension<IOrganizationServiceFactory>();
        var service = serviceFactory.CreateOrganizationService(null);
        
        var accountRef = Account.Get(context);
        var account = service.Retrieve(
            accountRef.LogicalName, 
            accountRef.Id, 
            new ColumnSet("name", "creditonhold")
        );
        
        var creditOnHold = account.GetAttributeValue<bool>("creditonhold");
        if (creditOnHold)
        {
            throw new InvalidPluginExecutionException(
                $"Cannot process account {account.GetAttributeValue<string>("name")} - credit is on hold"
            );
        }
    }
}

[Fact]
public void Should_Throw_When_Account_Credit_On_Hold()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account 
    { 
        Id = accountId, 
        Name = "Contoso",
        CreditOnHold = true
    };
    _context.Initialize(new[] { account });
    
    var inputs = new Dictionary<string, object>
    {
        { "Account", account.ToEntityReference() }
    };
    
    var activity = new ValidateAccountActivity();
    
    // ACT & ASSERT
    var exception = Assert.Throws<InvalidPluginExecutionException>(() =>
        _context.ExecuteCodeActivity<ValidateAccountActivity>(activity, inputs)
    );
    
    Assert.Contains("credit is on hold", exception.Message);
    Assert.Contains("Contoso", exception.Message);
}
```

## Testing with Tracing

```csharp
public class TracingActivity : CodeActivity
{
    protected override void Execute(CodeActivityContext context)
    {
        var tracingService = context.GetExtension<ITracingService>();
        
        tracingService.Trace("Starting custom workflow activity");
        tracingService.Trace("Processing step 1");
        // ... do work ...
        tracingService.Trace("Completed successfully");
    }
}

[Fact]
public void Should_Write_Trace_Messages()
{
    // ARRANGE
    var inputs = new Dictionary<string, object>();
    var activity = new TracingActivity();
    
    // ACT
    _context.ExecuteCodeActivity<TracingActivity>(activity, inputs);
    
    // ASSERT
    var traceLog = _context.GetFakeTracingService().DumpTrace();
    Assert.Contains("Starting custom workflow activity", traceLog);
    Assert.Contains("Processing step 1", traceLog);
    Assert.Contains("Completed successfully", traceLog);
}
```

## Common Patterns

### Activity with Multiple Entity References

```csharp
[Fact]
public void Should_Link_Contact_To_Account()
{
    // Test linking entities via CodeActivity
    var accountId = Guid.NewGuid();
    var contactId = Guid.NewGuid();
    
    var inputs = new Dictionary<string, object>
    {
        { "Account", new EntityReference("account", accountId) },
        { "Contact", new EntityReference("contact", contactId) }
    };
    
    var outputs = _context.ExecuteCodeActivity<LinkContactToAccountActivity>(
        new LinkContactToAccountActivity(), 
        inputs
    );
    
    // Verify relationship was created
    var contact = _service.Retrieve("contact", contactId, new ColumnSet("parentcustomerid"));
    Assert.Equal(accountId, ((EntityReference)contact["parentcustomerid"]).Id);
}
```

### Activity with Complex Return Types

```csharp
[Fact]
public void Should_Return_Json_Result()
{
    // Test activity that returns JSON string
    var inputs = new Dictionary<string, object>
    {
        { "EntityId", Guid.NewGuid() },
        { "EntityType", "account" }
    };
    
    var outputs = _context.ExecuteCodeActivity<SerializeEntityActivity>(
        new SerializeEntityActivity(), 
        inputs
    );
    
    var jsonResult = (string)outputs["JsonResult"];
    Assert.NotNull(jsonResult);
    Assert.Contains("entityId", jsonResult);
    Assert.Contains("account", jsonResult);
}
```

## Best Practices

1. **Always test required arguments** - Verify activity fails when required inputs are missing
2. **Test default values** - Ensure default values work when optional inputs aren't provided
3. **Mock external dependencies** - Don't make real HTTP calls or file system access
4. **Test workflow context properties** - Verify activity works with different workflow contexts
5. **Validate output types** - Ensure outputs match declared types
6. **Test trace logging** - Verify diagnostic messages are written correctly
7. **Use early-bound entities** - Makes tests more readable and catches type errors

## Limitations

- CodeActivities are **synchronous only** (no async/await)
- Cannot use Pipeline Simulation (workflows don't use plugin pipeline)
- Limited to workflow-specific context (different from plugin context)
- Workflows are legacy technology (Power Automate is the modern replacement)

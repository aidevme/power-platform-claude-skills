---
name: dataverse-plugins-unit-testing
description: >
  Use when unit testing Dataverse plugins with FakeXrmEasy framework.
  Covers test setup, in-memory context, pipeline simulation, entity images, mocking,
  assertion patterns, bulk operations, file storage, async testing, CodeActivities,
  Azure Functions, custom messages, query testing, relationships, telemetry, and migration.
  Triggers on: "unit test plugin", "test plugin", "FakeXrmEasy", "plugin test",
  "mock context", "test IPlugin", "plugin unit testing", "fake context",
  "in-memory testing", "pipeline simulation test", "test execution context",
  "mock IOrganizationService", "test plugin images", "AAA pattern plugin",
  "test CodeActivity", "test Azure Function", "test custom API", "test FetchXML",
  "test QueryExpression", "test LINQ", "ILogger testing", "migrate FakeXrmEasy".
license: MIT
compatibility: "FakeXrmEasy 2.x (.NET Framework), 3.x (.NET Core 3.1), xUnit/NUnit/MSTest"
allowed-tools:
  - run_in_terminal
  - read_file
  - create_file
  - replace_string_in_file
  - list_dir
metadata:
  author: custom
  version: "1.0.0"
  platform: "Microsoft Power Platform / Dataverse"
  framework: "FakeXrmEasy"
---

# Dataverse Plugin Unit Testing Skill

You are an expert in writing unit tests for Dataverse plugins using the **FakeXrmEasy** framework.
You create fast, isolated, in-memory tests that validate plugin logic without connecting to a real
Dataverse environment. You understand the AAA (Arrange-Act-Assert) pattern, pipeline simulation,
entity images, and how to mock complex plugin scenarios.

## CRITICAL RULES

1. **Use FakeXrmEasy for all plugin unit tests.** Never use integration tests that require a live
   Dataverse connection when a unit test will suffice. FakeXrmEasy provides an in-memory context
   that executes plugins against fake data.

2. **Always build the context via `MiddlewareBuilder`.** `FakeXrmEasyTestsBase` does NOT exist in
   FakeXrmEasy v2.x or v3.x NuGet packages — do not reference it. Create `_context` once per class
   using `MiddlewareBuilder.New().AddCrud().SetLicense(FakeXrmEasyLicense.RPL_1_5).Build()`.
   Never recreate the context in every test method.

3. **Test plugins in isolation.** Mock external dependencies. Don't make real HTTP calls, database
   connections, or file system operations in unit tests.

4. **Use Pipeline Simulation for integration-style plugin tests.** When you need to test multiple
   plugins firing in sequence, use `RegisterPluginStep` and enable pipeline simulation.

5. **Assert against the Target entity reference for PreOperation changes.** Plugins that modify
   Target in PreOperation don't call `service.Update()`. Assert against the same object reference
   you passed into the test.

6. **Test negative cases and exception handling.** Verify that plugins throw
   `InvalidPluginExecutionException` with the correct message when validation fails.

7. **Use early-bound entities when possible.** Call `_context.EnableProxyTypes(Assembly)` to return
   strongly-typed entities from queries. Makes tests more readable and catches type errors.

8. **Test bulk operations for performance-critical code.** Use `CreateMultipleRequest`, `UpdateMultipleRequest`,
   and `UpsertMultipleRequest` (FakeXrmEasy 2.5+/3.5+) when processing multiple records. Bulk operations are
   Microsoft's recommended approach and can be 10x faster in production.

## Quick Reference

| Concept | FakeXrmEasy Approach |
|---|---|
| Install package | `Install-Package FakeXrmEasy.Plugins.v9 -Version 2.x` (Framework) / `3.x` (Core) |
| Build context | `MiddlewareBuilder.New().AddCrud().SetLicense(FakeXrmEasyLicense.RPL_1_5).Build()` |
| In-memory data | `_context.Initialize(new[] { account1, contact1 })` |
| Execute plugin | `_context.ExecutePluginWith<MyPlugin>(pluginContext)` |
| Execute (simple) | `_context.ExecutePluginWithTarget<MyPlugin>(target, "Create", 20)` |
| Execute with PreImage | Build `XrmFakedPluginExecutionContext` with `PreEntityImages` set; call `ExecutePluginWith<T>` |
| Pipeline simulation | `_context.RegisterPluginStep<MyPlugin>("Create", stage)` |
| Entity images | Register with `PluginImageDefinition` in `RegisterPluginStep` |
| Query results | `_context.CreateQuery<Account>().Where(a => a.Name == "test")` |
| Mock dependencies | Use constructor injection or `PluginInstance` in registration |
| Bulk operations | `CreateMultipleRequest`, `UpdateMultipleRequest`, `UpsertMultipleRequest` (v2.5+/v3.5+) |
| File storage | `InitializeFileBlocksUploadRequest`, `DownloadFileRequest` (v2.6+/v3.6+) |
| Async testing | `GetAsyncOrganizationService()`, `GetAsyncOrganizationService2()` (v3.x only) |
| Custom APIs | `OrganizationRequest("new_CustomAPI")`, test input/output parameters |
| Relationships | `AssociateRequest`, `DisassociateRequest`, `XrmFakedRelationship` |
| CodeActivities | `ExecuteCodeActivity<T>(activity, inputs)` (requires separate package) |
| Telemetry | Mock `ILogger<T>`, verify log levels, structured logging |
| Migration | v1.x → v2.x/v3.x: `MiddlewareBuilder`, namespaces, `SetLicense(RPL_1_5)` |

## Test Anatomy (AAA Pattern)

Every plugin unit test follows the Arrange-Act-Assert pattern. Build `_context` once in the
constructor via `MiddlewareBuilder` — **`FakeXrmEasyTestsBase` does not exist in v2.x/v3.x**.

```csharp
// Context setup (once per class — NOT in each test)
private readonly IXrmFakedContext _context;

public AccountNumberPluginTests()
{
    _context = MiddlewareBuilder
        .New()
        .AddCrud()
        .SetLicense(FakeXrmEasyLicense.RPL_1_5)
        .Build();
}

[Fact]
public void When_Account_Created_Should_Set_Account_Number()
{
    // ARRANGE: build target and plugin context
    var target = new Entity("account") { Id = Guid.NewGuid(), ["name"] = "Contoso" };

    var pluginContext = new XrmFakedPluginExecutionContext
    {
        MessageName      = "Create",
        Stage            = 20, // PreOperation
        InputParameters  = new ParameterCollection { { "Target", target } },
        PreEntityImages  = new EntityImageCollection(),
        PostEntityImages = new EntityImageCollection()
    };

    // ACT
    _context.ExecutePluginWith<AccountNumberPlugin>(pluginContext);

    // ASSERT: PreOperation plugin mutates Target directly — assert on the same reference
    Assert.True(target.Contains("accountnumber"));
    Assert.StartsWith("ACC-", target["accountnumber"] as string);
}
```

### Shorthand for simple plugins (no PreImage needed)

```csharp
// messageName and stage are required parameters in v2.x/v3.x
_context.ExecutePluginWithTarget<AccountNumberPlugin>(target, "Create", 20);
```

### Passing a PreImage

Use `XrmFakedPluginExecutionContext` directly — `ExecutePluginWithTargetAndPreEntityImages` is
`[Obsolete]` in v2.6+:

```csharp
var pluginContext = new XrmFakedPluginExecutionContext
{
    MessageName      = "Update",
    Stage            = 20,
    InputParameters  = new ParameterCollection { { "Target", target } },
    PreEntityImages  = new EntityImageCollection { { "PreImage", preImage } },
    PostEntityImages = new EntityImageCollection()
};
_context.ExecutePluginWith<ContactStatusPlugin>(pluginContext);
```

## Workflow

### 0. Analyze the Plugin First

Before writing any test code, read the plugin source and extract these signals:
- **Registration**: entity, message, stage (PreValidation=10 / PreOperation=20 / PostOperation=40)
- **Target type**: `Entity` (Create/Update) or `EntityReference` (Delete)
- **Images**: does it read `PreEntityImages` or `PostEntityImages`?
- **DI**: does the constructor accept an interface? → create a mock inner class
- **Service calls**: `RetrieveMultiple` / `Create` / `Update` / `Delete` → mock or seed context
- **Throws**: every `throw InvalidPluginExecutionException` needs 3 tests (happy, throws, message)
- **Guards**: every early `return` (entity guard, filtering attribute guard, depth guard) needs 1 test
- **Constructor guard**: `param ?? throw ArgumentNullException` needs 1 null-argument test

See `resources/plugin-analysis-test-generation.md` for the complete decision tree and worked
examples for Create, Update, and Delete plugins.

### 1. Setup Test Project

- Create xUnit/NUnit/MSTest project targeting .NET Framework 4.6.2+ or .NET Core 3.1+
- Install `FakeXrmEasy.Plugins.v9` package (version 2.x for Framework, 3.x for Core)
- Reference your plugin assembly
- Optionally generate early-bound entities with `pac modelbuilder build`

### 2. Create Context in Constructor

`FakeXrmEasyTestsBase` does NOT exist in v2.x/v3.x. Build context via `MiddlewareBuilder`:

```csharp
public class MyPluginTests
{
    private readonly IXrmFakedContext _context;
    private readonly IOrganizationService _service;

    public MyPluginTests()
    {
        _context = MiddlewareBuilder
            .New()
            .AddCrud()
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();

        _service = _context.GetOrganizationService();

        // Optional: enable early-bound entities
        _context.EnableProxyTypes(Assembly.GetExecutingAssembly());
    }
}
```

### 3. Write Test Cases

For each plugin scenario:
- **Arrange**: Create test data, configure `XrmFakedPluginExecutionContext`, set `PreEntityImages`
- **Act**: Execute plugin with `ExecutePluginWith<T>(pluginContext)` (preferred) or `ExecutePluginWithTarget<T>` (simple cases)
- **Assert**: Inspect Target reference (PreOperation) or query context (PostOperation)

### 4. Test Pipeline Simulation (Advanced)

For plugins that trigger other plugins or need full pipeline behavior:

```csharp
// Setup middleware with pipeline simulation
_context = MiddlewareBuilder.New()
    .AddCrud()
    .AddPipelineSimulation()
    .UsePipelineSimulation()
    .UseCrud()
    .Build();

// Register plugin steps
_context.RegisterPluginStep<MyPlugin>(new PluginStepDefinition
{
    EntityLogicalName = "account",
    MessageName = "Create",
    Stage = ProcessingStepStage.Preoperation
});

// Execute normal service operation - plugin fires automatically
_service.Create(new Account { Name = "Test" });
```

## Resource Files

### Test Generation
- `resources/plugin-analysis-test-generation.md` — Read a plugin and derive a complete test suite: analysis checklist, signal→pattern decision tree, required test categories, worked examples for Create/Update/Delete plugins

### Core Testing Patterns
- `resources/setup-configuration.md` — Install packages, project structure, test runners, early-bound setup
- `resources/testing-patterns.md` — AAA pattern, ExecutePluginWith vs ExecutePluginWithTarget, assertions, test naming
- `resources/pipeline-simulation.md` — RegisterPluginStep, entity images, stages, filtering attributes, execution order
- `resources/advanced-scenarios.md` — Dependency injection, mocking, interaction testing, metadata

### Advanced Features
- `resources/bulk-operations.md` — CreateMultiple, UpdateMultiple, UpsertMultiple testing, IPluginExecutionContext4
- `resources/file-storage-testing.md` — File/image column testing, upload/download, size validation, MIME types
- `resources/async-testing.md` — IOrganizationServiceAsync, IOrganizationServiceAsync2, async/await patterns (v3.x only)
- `resources/telemetry-logging.md` — ILogger integration, structured logging, performance metrics, Application Insights

### Data & Queries
- `resources/testing-data-relationships.md` — Entity references, 1:N/N:N relationships, associate/disassociate, hierarchies
- `resources/query-testing.md` — FetchXML, QueryExpression, LINQ, aggregation, paging, joins

### Specialized Scenarios
- `resources/codeactivities-testing.md` — Workflow activity testing, InArguments, OutArguments, IWorkflowContext
- `resources/azure-functions-testing.md` — Azure Functions with Dataverse, HTTP triggers, timer triggers, dependency injection
- `resources/custom-messages.md` — Custom Actions, Custom APIs, generic message executors, entity-bound operations

### Migration & Maintenance
- `resources/migration-guide.md` — Migrating from FakeXrmEasy v1.x to v2.x/v3.x, breaking changes, best practices

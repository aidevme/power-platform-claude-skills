# Plugin Analysis and Test Generation

This resource teaches you how to **read a Dataverse plugin and systematically derive a complete
unit test suite from its source code**. Rather than guessing which tests to write, follow the
workflow below:

1. **Phase 1** — Extract signals from the plugin source code
2. **Phase 1.5** — Resolve any ambiguities by asking the user (never guess registration metadata)
3. **Phase 2** — Map each confirmed signal to the right FakeXrmEasy pattern
4. **Phase 3** — Assemble the test class

**Critical rule:** If message, stage, or entity cannot be determined from the source code with
confidence, **stop and ask the user** before writing any test code. A test built on a wrong
assumption is worse than no test — it passes against the wrong scenario and gives false confidence.

---

## Phase 1: Read the Plugin — Analysis Checklist

Before writing a single test, read the plugin source and record the answers to every question
in the checklist. Each answer directly controls a test decision.

### 1.1 Registration signals (from comments, class name, or plugin step definition)

| Question | Answer drives |
| --- | --- |
| What entity is this registered on? | `InputParameters["Target"]` logical name; entity-guard test |
| What message? (Create / Update / Delete / custom) | Stage number; target type (Entity vs EntityReference) |
| What stage? (PreValidation=10 / PreOperation=20 / PostOperation=40) | Stage number in context; assertion strategy |
| Are filtering attributes declared? (Update only) | Internal guard test — plugin should exit early when those attributes are absent |
| Is a PreImage registered? What attributes does it include? | `PreEntityImages` setup in context |
| Is a PostImage registered? | `PostEntityImages` setup in context |

### 1.2 Target type signal

Look at how the plugin reads `context.InputParameters["Target"]`:

```csharp
// Entity target — Create and Update messages
if (context.InputParameters["Target"] is Entity target) { ... }

// EntityReference target — Delete message
if (context.InputParameters["Target"] is EntityReference targetRef) { ... }
```

| Observed cast | Test implication |
| --- | --- |
| `is Entity` | Pass `new Entity("account") { Id = ... }` as Target |
| `is EntityReference` | Pass `new EntityReference("account", id)` as Target; Stage is usually 10 (PreValidation) for Delete |

### 1.3 PreImage / PostImage access

```csharp
// Signals PreImage is needed
if (context.PreEntityImages.Contains("PreImage")) { ... }
Entity preImage = context.PreEntityImages["PreImage"];

// Signals PostImage is needed
Entity postImage = context.PostEntityImages["PostImage"];
```

If found: build `XrmFakedPluginExecutionContext` with the appropriate `EntityImageCollection`
populated. `ExecutePluginWithTarget` cannot supply images — use `ExecutePluginWith` instead.

### 1.4 Dependency injection signal

```csharp
// DI constructor — mock needed in tests
public MyPlugin(IMyChecker checker) { _checker = checker; }

// Parameterless fallback — default used in production
public MyPlugin() : this(new MyChecker()) { }
```

If found: create a private `Mock[Interface]` inner class in the test. Use the
`ExecutePluginWith(pluginContext, pluginInstance)` overload to inject the mock.
Never rely on the real implementation calling Dataverse — that defeats isolation.

### 1.5 Service calls inside the plugin

| Call found | What to do in tests |
| --- | --- |
| `service.RetrieveMultiple(query)` — via DI | Mock the interface; skip seeding records |
| `service.RetrieveMultiple(query)` — inline | Seed `_context.Initialize(records)` with matching data |
| `service.Create(entity)` | Assert via `_context.CreateQuery<T>()` after execution |
| `service.Update(entity)` | Assert via `_context.CreateQuery<T>().First(e => e.Id == id)` |
| `service.Delete(logicalName, id)` | Assert record is absent from context after execution |
| `service.Retrieve(...)` | Seed the record with `_context.Initialize(new[] { record })` |

### 1.6 Exception paths

Scan for every `throw new InvalidPluginExecutionException(...)` in the plugin body.

For **each** throw site you need two tests:
- A **negative test** that sets up conditions to reach the throw and asserts the exception is thrown
- A **positive test** that avoids those conditions and asserts no exception is thrown

Also assert on the exception **message text** in at least one negative test — that text is what
users see in the model-driven app error dialog.

### 1.7 Guard conditions and early-exit paths

```csharp
// Entity guard — plugin exits for non-matching entities
if (targetRef.LogicalName != "account") return;

// Filtering attribute guard — plugin exits when field not in update
if (!context.InputParameters.Contains("Target") ||
    !((Entity)context.InputParameters["Target"]).Contains("statuscode")) return;

// Depth guard — plugin exits to prevent recursion
if (context.Depth > 1) return;
```

Each guard requires a dedicated test that triggers the early return and confirms **no exception
is thrown and no side effects occur**.

### 1.8 Constructor guards

```csharp
_checker = checker ?? throw new ArgumentNullException(nameof(checker));
```

Add a test that passes `null` to the constructor and asserts `ArgumentNullException` with the
correct `ParamName`.

---

## Phase 1.5: Resolve Ambiguities — Ask the User Before Proceeding

After completing the Phase 1 checklist, evaluate every registration signal that could not be
determined from the source code. **Do not guess or assume defaults.** Pause and ask the user
to confirm each unknown value before writing any test code.

### When to ask

Apply the decision rules below in order. If a signal is unambiguous, record it and move on.
If the rule ends with **→ ASK**, stop and present the prompt to the user.

---

#### Message ambiguity

**Unambiguous signals (no need to ask):**

| Signal in code | Conclusion |
| --- | --- |
| `context.InputParameters["Target"] is EntityReference` | Delete |
| `context.PreEntityImages` accessed | Update (PreImage only exists on Update) |
| Plugin comment says `// Registered on: ..., Message: Create` | Use stated message |
| Class name contains `Create`, `Update`, `Delete` | Strong hint — confirm, don't assume |

**Ambiguous — → ASK:**

- Target is cast as `Entity` AND there is no PreImage access AND no comment stating the message
- Plugin handles multiple messages via `context.MessageName` switch

**Prompt to show the user:**

```
I could not determine the registered message for [PluginName] from the source code alone.

Which message is this plugin registered on?

  1. Create   — fires when a new record is created
  2. Update   — fires when an existing record is modified
  3. Delete   — fires when a record is deleted
  4. Other    — e.g. Assign, SetState, custom message (please specify)

Please enter the number or type the message name.
```

---

#### Stage ambiguity

**Unambiguous signals (no need to ask):**

| Signal in code | Conclusion |
| --- | --- |
| Plugin throws `InvalidPluginExecutionException` for validation AND does no data writes | Likely PreValidation (10) — confirm |
| Plugin modifies Target fields without `service.Update()` | PreOperation (20) |
| Plugin calls `service.Create()` / `service.Update()` / `service.Delete()` on other records | PostOperation (40) |
| `context.Stage == 10` or `context.Stage == 20` branch inside the plugin | Both stages used |
| Plugin comment states `Stage: PreOperation` | Use stated stage |

**Ambiguous — → ASK:**

- Plugin throws an exception for validation AND also writes data (could be PreValidation or PreOperation)
- Plugin only reads data and traces — no writes, no throws (stage has no observable effect)
- No comment, and no stage-specific behavior

**Prompt to show the user:**

```
I could not determine the registered stage for [PluginName] from the source code alone.

Which stage is this plugin registered on?

  1. PreValidation  (10) — runs before the database transaction begins; used for fail-fast validation
  2. PreOperation   (20) — runs inside the transaction before the record is saved; used for data manipulation
  3. PostOperation  (40) — runs after the record is saved; used for side effects (creating related records, etc.)
  4. Multiple stages    — same class registered on more than one stage (specify which stages)

Please enter the number or describe the registration.
```

---

#### Entity ambiguity

**Unambiguous signals (no need to ask):**

| Signal in code | Conclusion |
| --- | --- |
| `entity.LogicalName != "account"` guard | Registered on `account` |
| `targetRef.LogicalName != "contact"` guard | Registered on `contact` |
| Plugin comment states the entity | Use stated entity |

**Ambiguous — → ASK:**

- Plugin has no entity guard (it may handle multiple entities, or the guard may be missing)
- Entity name cannot be found in the source

**Prompt to show the user:**

```
I could not determine which entity [PluginName] is registered on from the source code.

What is the primary entity (logical name) for this plugin step?
Examples: account, contact, opportunity, lead, systemuser, or a custom entity like new_order

Please type the entity logical name.
```

---

#### Filtering attributes ambiguity (Update only)

**Unambiguous signals (no need to ask):**

| Signal in code | Conclusion |
| --- | --- |
| `if (!entity.Contains("fieldname")) return;` inside the plugin | Filtering attribute is `fieldname`; generate filtering-attribute guard test |
| Plugin comment lists filtering attributes | Use stated attributes |
| No such guard in the code | No filtering attribute guard test needed |

**Ambiguous — → ASK** (only when message = Update):

- Plugin has no internal guard for a specific field, but filtering attributes in the registration would prevent it firing unnecessarily

**Prompt to show the user:**

```
This is an Update plugin. Are filtering attributes configured on the plugin step registration?

Filtering attributes control which field changes trigger the plugin. Without them, the plugin
fires on every update (including autosave), which can cause performance issues.

  1. Yes — list the attribute(s): _______________
  2. No  — plugin fires on all updates
  3. I don't know

If filtering attributes are set, I will generate a test verifying the plugin exits early
when those attributes are absent from the Target.
```

---

### How to proceed after receiving the user's answers

1. Record the confirmed values (message, stage, entity, filtering attributes).
2. Resume Phase 1 with those values filled in.
3. Continue to Phase 2 (decision tree) with the complete, confirmed checklist.
4. **Do not ask again** for values already confirmed in this session.

If the user answers "I don't know" for filtering attributes, generate a comment in the test
file noting that filtering attributes should be verified in the Plugin Registration Tool.

---

## Phase 2: Map Signals to Patterns

Use the decision tree below to select the right FakeXrmEasy execution pattern and assertion
strategy for each scenario.

### 2.1 Execution method selection

```
Does the plugin use DI (constructor accepts an interface)?
│
├── YES → create mock inner class
│         create plugin instance: var plugin = new MyPlugin(mockDep);
│         use: _context.ExecutePluginWith(pluginContext, plugin)
│
└── NO  → Does the test need PreEntityImages?
          │
          ├── YES → build XrmFakedPluginExecutionContext with PreEntityImages populated
          │         use: _context.ExecutePluginWith<MyPlugin>(pluginContext)
          │
          └── NO  → use shorthand:
                    _context.ExecutePluginWithTarget<MyPlugin>(target, messageName, stage)
```

### 2.2 Target construction

```
What is the message?
│
├── Create  → new Entity("account") { Id = Guid.NewGuid(), ["name"] = "..." }
├── Update  → new Entity("account") { Id = existingId, ["field"] = newValue }
│             (only changed attributes — not the full record)
└── Delete  → new EntityReference("account", existingId)
              Stage is typically 10 (PreValidation) or 20 (PreOperation)
```

### 2.3 Assertion strategy

```
What stage is the plugin registered on?
│
├── PreValidation (10) / PreOperation (20)
│   └── Does plugin mutate Target (no service.Update)?
│       ├── YES → assert on the same entity/ref object passed as Target
│       └── NO  → assert on _context.CreateQuery<T>() after execution
│
└── PostOperation (40)
    └── assert on _context.CreateQuery<T>() — side effects are in the DB
```

### 2.4 Test category checklist

Generate at least one test per row that applies to the plugin being analyzed:

| Category | Required when | Example test name pattern |
| --- | --- | --- |
| Happy path | Always | `When_[condition]_Should_[allow/set/create]_[thing]` |
| Exception thrown | Plugin has `throw InvalidPluginExecutionException` | `When_[bad condition]_Should_Throw_InvalidPluginExecutionException` |
| Exception message | Plugin throws with user-facing message | `When_[bad condition]_Exception_Message_Should_[describe content]` |
| Entity guard | Plugin checks `LogicalName` | `When_Target_Is_Not_[entity]_Should_Skip_Without_Exception` |
| Filtering attribute guard | Update plugin checks attribute presence | `When_[attribute]_Not_In_Target_Should_Not_[side effect]` |
| Constructor null guard | Constructor uses `??` throw pattern | `When_Null_[param]_Passed_To_Constructor_Should_Throw_ArgumentNullException` |
| Depth guard | Plugin checks `context.Depth` | `When_Depth_Greater_Than_1_Should_Skip_Processing` |
| Side-effect verification | Plugin calls `service.Create/Update/Delete` | `When_[condition]_Should_[Create/Update/Delete]_[related record]` |
| PreImage absent | Plugin handles missing PreImage gracefully | `When_PreImage_Missing_Should_[fallback behavior]` |

---

## Phase 3: Assemble the Test Class

### 3.1 Standard test class template

```csharp
using System;
using Xunit;
using FakeXrmEasy.Abstractions;
using FakeXrmEasy.Abstractions.Enums;
using FakeXrmEasy.Middleware;
using FakeXrmEasy.Middleware.Crud;
using FakeXrmEasy.Plugins;
using Microsoft.Xrm.Sdk;
using YourPluginNamespace;

namespace YourTests
{
    public class [PluginName]Tests
    {
        // ── Mock dependencies (only if plugin uses DI) ────────────────────────
        private sealed class Mock[Interface] : [Interface]
        {
            // Configurable return values via constructor
            private readonly bool _returnValue;
            public Mock[Interface](bool returnValue) => _returnValue = returnValue;
            public [ReturnType] [Method](IOrganizationService service, Guid id) => _returnValue;
        }

        private readonly IXrmFakedContext _context;

        public [PluginName]Tests()
        {
            // FakeXrmEasyTestsBase does NOT exist in v2.x/v3.x
            _context = MiddlewareBuilder
                .New()
                .AddCrud()
                .SetLicense(FakeXrmEasyLicense.RPL_1_5)
                .Build();
        }

        // ── Context builder helpers ────────────────────────────────────────────
        private static XrmFakedPluginExecutionContext Build[Message]Context(/* params */)
        {
            return new XrmFakedPluginExecutionContext
            {
                MessageName      = "[Create|Update|Delete]",
                Stage            = [10|20|40],
                InputParameters  = new ParameterCollection { { "Target", /* target */ } },
                PreEntityImages  = new EntityImageCollection(),
                PostEntityImages = new EntityImageCollection()
            };
        }

        // ── Tests ─────────────────────────────────────────────────────────────

        [Fact]
        public void When_[HappyPathCondition]_Should_[ExpectedResult]()
        {
            // ARRANGE
            // ACT
            // ASSERT
        }
    }
}
```

### 3.2 Private mock inner class pattern

Keep mock classes `private sealed` and nested inside the test class. They belong to the test,
not the production assembly, and should not be reused across test classes.

```csharp
// Configurable via constructor — avoids separate classes per scenario
private sealed class MockOpportunityChecker : IOpportunityChecker
{
    private readonly bool _hasOpportunities;
    public MockOpportunityChecker(bool hasOpportunities) => _hasOpportunities = hasOpportunities;
    public bool HasRelatedOpportunities(IOrganizationService service, Guid accountId)
        => _hasOpportunities;
}

// Usage in tests
var mock = new MockOpportunityChecker(hasOpportunities: true);
var plugin = new AccountDeletionPreventionPlugin(mock);
_context.ExecutePluginWith(pluginContext, plugin);
```

---

## Worked Example 1: PreOperation Create Plugin (AccountNumberPlugin)

### Plugin analysis output

| Signal | Value |
| --- | --- |
| Entity | account |
| Message | Create |
| Stage | PreOperation (20) |
| Target type | `Entity` |
| DI | None (parameterless constructor only) |
| Service calls | None |
| Throws | No |
| Guards | Checks `target.LogicalName == "account"`; skips if `accountnumber` already set |
| PreImage | Not used |

### Derived test suite

```csharp
public class AccountNumberPluginTests
{
    private readonly IXrmFakedContext _context;

    public AccountNumberPluginTests()
    {
        _context = MiddlewareBuilder
            .New()
            .AddCrud()
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();
    }

    // Happy path — generated number has correct format
    [Fact]
    public void When_Account_Created_Without_AccountNumber_Should_Generate_AccountNumber()
    {
        var target = new Entity("account") { Id = Guid.NewGuid(), ["name"] = "Contoso" };
        _context.ExecutePluginWithTarget<AccountNumberPlugin>(target, "Create", 20);

        Assert.True(target.Contains("accountnumber"));
        Assert.StartsWith("ACC-", (string)target["accountnumber"]);
        Assert.Equal(12, ((string)target["accountnumber"]).Length);
    }

    // Guard — existing value is preserved (no overwrite)
    [Fact]
    public void When_Account_Created_With_Existing_AccountNumber_Should_Not_Override()
    {
        const string existing = "ACC-CUSTOM01";
        var target = new Entity("account")
        {
            Id = Guid.NewGuid(),
            ["name"] = "Contoso",
            ["accountnumber"] = existing
        };
        _context.ExecutePluginWithTarget<AccountNumberPlugin>(target, "Create", 20);

        Assert.Equal(existing, target["accountnumber"]);
    }

    // Entity guard — non-account entities are skipped
    [Fact]
    public void When_Target_Is_Not_Account_Should_Skip_Without_Setting_AccountNumber()
    {
        var target = new Entity("contact") { Id = Guid.NewGuid(), ["firstname"] = "Jane" };
        _context.ExecutePluginWithTarget<AccountNumberPlugin>(target, "Create", 20);

        Assert.False(target.Contains("accountnumber"));
    }
}
```

---

## Worked Example 2: PreValidation Delete Plugin with DI (AccountDeletionPreventionPlugin)

### Plugin analysis output

| Signal | Value |
| --- | --- |
| Entity | account |
| Message | Delete |
| Stage | PreValidation (10) |
| Target type | `EntityReference` |
| DI | `IOpportunityChecker` — constructor injection |
| Service calls | Delegated to checker (query lives inside the real `OpportunityChecker`) |
| Throws | `InvalidPluginExecutionException` with user-facing message when opportunities exist |
| Guards | Checks `targetRef.LogicalName == "account"`; returns silently for other entities |
| Constructor guard | `checker ?? throw new ArgumentNullException(nameof(checker))` |

### Derived test suite

```csharp
public class AccountDeletionPreventionPluginTests
{
    // Mock bypasses FakeXrmEasy QueryExpression — returns a configurable bool
    private sealed class MockOpportunityChecker : IOpportunityChecker
    {
        private readonly bool _hasOpportunities;
        public MockOpportunityChecker(bool hasOpportunities) => _hasOpportunities = hasOpportunities;
        public bool HasRelatedOpportunities(IOrganizationService service, Guid accountId)
            => _hasOpportunities;
    }

    private readonly IXrmFakedContext _context;

    public AccountDeletionPreventionPluginTests()
    {
        _context = MiddlewareBuilder
            .New()
            .AddCrud()
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();
    }

    // Delete messages use EntityReference as Target
    private static XrmFakedPluginExecutionContext BuildDeleteContext(Guid accountId) =>
        new XrmFakedPluginExecutionContext
        {
            MessageName      = "Delete",
            Stage            = 10, // PreValidation
            InputParameters  = new ParameterCollection
            {
                { "Target", new EntityReference("account", accountId) }
            },
            PreEntityImages  = new EntityImageCollection(),
            PostEntityImages = new EntityImageCollection()
        };

    // Happy path — no opportunities → deletion allowed
    [Fact]
    public void When_Account_Has_No_Related_Opportunities_Should_Allow_Deletion()
    {
        var plugin = new AccountDeletionPreventionPlugin(
            new MockOpportunityChecker(hasOpportunities: false));

        var ex = Record.Exception(() =>
            _context.ExecutePluginWith(BuildDeleteContext(Guid.NewGuid()), plugin));
        Assert.Null(ex);
    }

    // Negative path — opportunities exist → exception thrown
    [Fact]
    public void When_Account_Has_Related_Opportunities_Should_Throw_InvalidPluginExecutionException()
    {
        var plugin = new AccountDeletionPreventionPlugin(
            new MockOpportunityChecker(hasOpportunities: true));

        Assert.Throws<InvalidPluginExecutionException>(() =>
            _context.ExecutePluginWith(BuildDeleteContext(Guid.NewGuid()), plugin));
    }

    // Exception message content — user-friendly wording verified
    [Fact]
    public void When_Deletion_Prevented_Exception_Message_Should_Explain_Reason_And_Next_Steps()
    {
        var plugin = new AccountDeletionPreventionPlugin(
            new MockOpportunityChecker(hasOpportunities: true));

        var ex = Assert.Throws<InvalidPluginExecutionException>(() =>
            _context.ExecutePluginWith(BuildDeleteContext(Guid.NewGuid()), plugin));

        Assert.Contains("Cannot delete this account",  ex.Message, StringComparison.OrdinalIgnoreCase);
        Assert.Contains("related opportunities",        ex.Message, StringComparison.OrdinalIgnoreCase);
        Assert.Contains("remove or reassign",           ex.Message, StringComparison.OrdinalIgnoreCase);
    }

    // Constructor guard
    [Fact]
    public void When_Null_Checker_Passed_To_Constructor_Should_Throw_ArgumentNullException()
    {
        var ex = Assert.Throws<ArgumentNullException>(() =>
            new AccountDeletionPreventionPlugin(null));
        Assert.Equal("opportunityChecker", ex.ParamName);
    }

    // Entity guard — non-account entities exit early
    [Fact]
    public void When_Target_Is_Not_Account_Should_Skip_Validation_Without_Exception()
    {
        // Mock returns true — would block if the guard wasn't working
        var plugin = new AccountDeletionPreventionPlugin(
            new MockOpportunityChecker(hasOpportunities: true));

        var pluginContext = new XrmFakedPluginExecutionContext
        {
            MessageName      = "Delete",
            Stage            = 10,
            InputParameters  = new ParameterCollection
            {
                { "Target", new EntityReference("contact", Guid.NewGuid()) }
            },
            PreEntityImages  = new EntityImageCollection(),
            PostEntityImages = new EntityImageCollection()
        };

        var ex = Record.Exception(() => _context.ExecutePluginWith(pluginContext, plugin));
        Assert.Null(ex);
    }
}
```

---

## Worked Example 3: PreOperation Update Plugin with PreImage and Filtering (ContactStatusPlugin)

### Plugin analysis output

| Signal | Value |
| --- | --- |
| Entity | contact |
| Message | Update |
| Stage | PreOperation (20) |
| Target type | `Entity` (only changed fields) |
| DI | None |
| Filtering attributes | `statuscode` — plugin exits early if absent from Target |
| PreImage | `"PreImage"` — contains old `statuscode` value |
| Side effects | Sets `new_modifiedreason` on Target (no service.Update) |
| Guards | Checks `target.LogicalName == "contact"`; checks `statuscode` in Target |
| Throws | No |

### Derived test suite

```csharp
public class ContactStatusPluginTests
{
    private static readonly OptionSetValue Active   = new OptionSetValue(1);
    private static readonly OptionSetValue Inactive = new OptionSetValue(2);

    private readonly IXrmFakedContext _context;

    public ContactStatusPluginTests()
    {
        _context = MiddlewareBuilder
            .New()
            .AddCrud()
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();
    }

    // Helper — PreImage is optional; when null the context has an empty PreEntityImages
    private static XrmFakedPluginExecutionContext BuildUpdateContext(
        Entity target, Entity preImage = null)
    {
        var ctx = new XrmFakedPluginExecutionContext
        {
            MessageName      = "Update",
            Stage            = 20,
            InputParameters  = new ParameterCollection { { "Target", target } },
            PreEntityImages  = new EntityImageCollection(),
            PostEntityImages = new EntityImageCollection()
        };
        if (preImage != null)
            ctx.PreEntityImages.Add("PreImage", preImage);
        return ctx;
    }

    // Happy path — status changes, reason recorded
    [Fact]
    public void When_Status_Changes_From_Active_To_Inactive_Should_Set_ModifiedReason()
    {
        var contactId = Guid.NewGuid();
        var preImage  = new Entity("contact", contactId) { ["statuscode"] = Active };
        var target    = new Entity("contact", contactId) { ["statuscode"] = Inactive };

        _context.ExecutePluginWith<ContactStatusPlugin>(BuildUpdateContext(target, preImage));

        var reason = (string)target["new_modifiedreason"];
        Assert.NotNull(reason);
        Assert.Contains("Active",   reason);
        Assert.Contains("Inactive", reason);
    }

    // PreImage absent — graceful fallback
    [Fact]
    public void When_PreImage_Missing_Should_Set_ModifiedReason_With_New_Status_Only()
    {
        var target = new Entity("contact", Guid.NewGuid()) { ["statuscode"] = Inactive };
        _context.ExecutePluginWith<ContactStatusPlugin>(BuildUpdateContext(target));

        var reason = (string)target["new_modifiedreason"];
        Assert.NotNull(reason);
        Assert.DoesNotContain("changed from", reason, StringComparison.OrdinalIgnoreCase);
    }

    // Filtering attribute guard — statuscode absent from Target
    [Fact]
    public void When_StatusCode_Not_In_Target_Should_Not_Set_ModifiedReason()
    {
        var contactId = Guid.NewGuid();
        var preImage  = new Entity("contact", contactId) { ["statuscode"] = Active };
        var target    = new Entity("contact", contactId) { ["firstname"] = "Jane" };

        _context.ExecutePluginWith<ContactStatusPlugin>(BuildUpdateContext(target, preImage));

        Assert.False(target.Contains("new_modifiedreason"));
    }

    // Entity guard — non-contact entity is skipped
    [Fact]
    public void When_Target_Is_Not_Contact_Should_Skip_Without_Setting_ModifiedReason()
    {
        var target = new Entity("account", Guid.NewGuid()) { ["statuscode"] = Inactive };
        _context.ExecutePluginWith<ContactStatusPlugin>(BuildUpdateContext(target));

        Assert.False(target.Contains("new_modifiedreason"));
    }
}
```

---

## Quick Reference: Signal → Test Implication

| Signal in plugin code | Test you must write |
| --- | --- |
| `is EntityReference` cast on Target | Use `new EntityReference("entity", id)` in `InputParameters["Target"]`; set `Stage = 10` for PreValidation Delete |
| `PreEntityImages["PreImage"]` read | Build `XrmFakedPluginExecutionContext` with `PreEntityImages` populated; add a test where PreImage is absent |
| `PostEntityImages["PostImage"]` read | Build `XrmFakedPluginExecutionContext` with `PostEntityImages` populated |
| Constructor accepts interface | Create `private sealed class Mock[Interface]` with configurable return; use `ExecutePluginWith(ctx, instance)` |
| `service.RetrieveMultiple(...)` — no DI | `_context.Initialize(new[] { relatedRecord })` to seed query results |
| `throw new InvalidPluginExecutionException` | Negative test (assert throws) + message content test + positive test (assert no throw) |
| `if (logicalName != "x") return;` | Test with wrong entity — assert no exception and no side effects |
| `if (!target.Contains("field")) return;` | Test with that field absent from Target — assert no side effects |
| `if (context.Depth > 1) return;` | Test with `ctx.Depth = 2` — assert no side effects |
| `param ?? throw new ArgumentNullException` | `Assert.Throws<ArgumentNullException>(() => new MyPlugin(null))` + verify `ParamName` |
| `service.Create(newEntity)` | After execution: `_context.CreateQuery<T>().Single(...)` to verify the record was created |
| `service.Update(entity)` | After execution: `_context.CreateQuery<T>().First(e => e.Id == id)` to verify updated values |
| `tracingService.Trace(...)` | No assertion needed unless tracing affects behavior; FakeXrmEasy captures traces silently |
| `context.SharedVariables["key"]` | Set `ctx.SharedVariables = new ParameterCollection()` before execution; assert key/value after |
| `context.UserId` / `context.InitiatingUserId` | Set `ctx.UserId = specificGuid` to test user-sensitive logic |

---

## Best Practices

1. **One analysis pass before writing any code.** Fill in the checklist completely before opening
   a test file. Tests written without analysis tend to miss guard conditions and exception message
   checks.

2. **Mock interfaces, not queries.** If the plugin has a DI seam, always use it. Seeding
   `_context.Initialize()` to simulate a query works but ties the test to the query structure.
   A mock is simpler, faster, and immune to FakeXrmEasy query limitations.

3. **Every `throw` site needs three tests.** Happy path (no throw), negative path (throws),
   message content (correct text). Two out of three is incomplete.

4. **Every early-return guard needs its own test.** Entity guards, filtering attribute guards,
   and depth guards are the most commonly untested code paths. Each `return;` statement in the
   plugin body should correspond to at least one test that reaches it.

5. **Name the `BuildXxxContext` helper after the message.** `BuildDeleteContext`,
   `BuildUpdateContext`, `BuildCreateContext` — this makes the setup intent readable at a glance
   and avoids repeating `XrmFakedPluginExecutionContext` construction in every test.

6. **PreOperation asserts go on the Target reference.** The Target object you pass in is the same
   object the plugin mutates. After `ExecutePluginWith`, query `target["field"]` directly.
   Do not re-read from `_context.CreateQuery<T>()` for PreOperation mutations.

7. **PostOperation asserts go on the context query.** The plugin called `service.Create` /
   `service.Update` — those writes went to the in-memory store. Use
   `_context.CreateQuery<T>().Single(...)` to verify them.

8. **Keep mocks inside the test class.** Private sealed nested classes prevent mock
   implementations from leaking into the production assembly and make test files self-contained.

9. **Use `Record.Exception` for "should not throw" assertions.** `Record.Exception(() => ...)` 
   returns `null` on success — clearer than wrapping the call in `try/catch` or relying on the
   test simply not failing.

10. **Test the `null` constructor argument even when it seems obvious.** `ArgumentNullException`
    guards are often the first thing a reviewer checks. Include the `ParamName` assertion to
    confirm the correct parameter name is reported.

# Project Instructions

## Available Skills (load automatically)

You have 1 domain skill:

| Skill | When to Use |
|---|---|
| `dataverse-plugins-unit-testing` | Unit testing Dataverse plugins with FakeXrmEasy framework — covers test setup, in-memory context, pipeline simulation, entity images, mocking, and assertion patterns. |

## Rules for Plugin Unit Testing

- Always use FakeXrmEasy framework for unit testing Dataverse plugins.
- Set up in-memory XrmFakedContext to simulate Dataverse environment.
- Properly configure pipeline simulation with images (PreImage/PostImage) where needed.
- Test both positive and negative scenarios for each plugin.
- When registering plugins on Update events, ALWAYS define filtering attributes to avoid triggering on autosave.
- Mock external dependencies (web service calls, custom APIs) to keep tests isolated and fast.
- Use proper assertion patterns to validate plugin behavior.
- Verify expected exceptions are thrown for invalid inputs.
- Test pipeline execution order when multiple plugins are registered.
- Always include test cases for security context and privilege validation.

## Best Practices for Plugin Unit Testing

- **Test setup isolation** — Each test should have its own isolated XrmFakedContext. Never share context between tests.
- **Entity initialization** — Always create test entities with required attributes populated. Missing primary name fields cause silent failures.
- **Pipeline simulation** — Configure proper MessageName, Stage, and PipelineStage in XrmFakedContext to match real plugin registration.
- **Entity images** — Add PreEntityImages and PostEntityImages to PluginExecutionContext when testing Update messages to simulate real scenarios.
- **Mock external calls** — Use interfaces and dependency injection to mock external web service calls, custom APIs, and third-party integrations.
- **Assertion patterns** — Verify entity attribute values, related record creation, and state transitions. Don't just assert no exceptions were thrown.
- **Exception testing** — Use `Assert.Throws<T>` or try-catch patterns to verify plugins throw appropriate exceptions for invalid inputs.
- **Query testing** — When plugins execute QueryExpression or FetchXml, populate XrmFakedContext with test data to validate query logic.
- **Relationship testing** — Test Many-to-Many associations and lookup field updates using FakeXrmEasy's relationship simulation.
- **Security context** — Test with different user privileges by configuring InitiatingUserId and UserId in PluginExecutionContext.
- **Async patterns** — For testing async plugins and Code Activities, configure proper async operation simulation.
- **Never silently swallow errors** — At minimum `console.error` in plugins, ideally throw InvalidPluginExecutionException with user-friendly messages.

## Plugin Development Workflow

When developing and testing Dataverse plugins:

1. **Write unit tests first** — Define expected behavior using FakeXrmEasy before implementing plugin logic (TDD approach).
2. **Run tests locally** — Use Visual Studio Test Explorer or `dotnet test` to run unit tests during development.
3. **Add test coverage** — Aim for 80%+ code coverage. Test all branches, error conditions, and edge cases.
4. **Register plugin** — Use Plugin Registration Tool or `pac plugin` CLI to deploy to Dataverse environment.
5. **Integration test** — Verify plugin behavior in real environment using Power Apps Monitor or Application Insights.
6. **Debug if needed** — Use Plugin Profiler to capture execution context for failing scenarios, then create unit tests to reproduce.

## Environment

- Platform: Windows
- Use Visual Studio or VS Code with C# Dev Kit for plugin development
- FakeXrmEasy framework for unit testing (NuGet package)
- Plugin Registration Tool or `pac plugin` CLI for deployment
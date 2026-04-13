# Plugin Telemetry & Logging with ILogger

Modern plugin development with structured logging and telemetry using ILogger interface.
Available in FakeXrmEasy v2.6+ (.NET Framework) and v3.6+ (.NET Core).

## Overview

Benefits of ILogger in plugins:
- **Structured logging** - Better than ITracingService for production
- **Log levels** - Debug, Information, Warning, Error, Critical
- **Telemetry integration** - Application Insights, Dataverse logs
- **Scopes and context** - Correlation IDs, user context
- **Performance metrics** - Execution time tracking

**Version Required:** FakeXrmEasy 2.6.0+ or 3.6.0+

## ILogger Basics

### Accessing ILogger in Plugin

```csharp
using Microsoft.Extensions.Logging;
using Microsoft.Xrm.Sdk;

public class AccountPlugin : IPlugin
{
    private readonly ILogger<AccountPlugin> _logger;
    
    // Constructor injection (modern approach)
    public AccountPlugin(ILogger<AccountPlugin> logger)
    {
        _logger = logger;
    }
    
    // Parameterless constructor for Dataverse registration
    public AccountPlugin() : this(null)
    {
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        // Fallback to ITracingService if ILogger not available
        _logger ??= serviceProvider.GetService(typeof(ILogger<AccountPlugin>)) as ILogger<AccountPlugin>;
        
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var serviceFactory = (IOrganizationServiceFactory)serviceProvider.GetService(typeof(IOrganizationServiceFactory));
        var service = serviceFactory.CreateOrganizationService(context.UserId);
        
        _logger?.LogInformation("Plugin execution started for {EntityName}", context.PrimaryEntityName);
        
        try
        {
            // Plugin logic
            var target = (Entity)context.InputParameters["Target"];
            _logger?.LogDebug("Processing account: {AccountName}", target.GetAttributeValue<string>("name"));
            
            // ... business logic ...
            
            _logger?.LogInformation("Plugin execution completed successfully");
        }
        catch (Exception ex)
        {
            _logger?.LogError(ex, "Plugin execution failed");
            throw;
        }
    }
}
```

## Testing with ILogger

### Basic Test with Logger Mock

```csharp
using Microsoft.Extensions.Logging;
using Moq;
using Xunit;

public class AccountPluginTests
{
    private readonly IXrmFakedContext _context;
    private readonly IOrganizationService _service;
    private readonly Mock<ILogger<AccountPlugin>> _loggerMock;
    
    public AccountPluginTests()
    {
        _context = MiddlewareBuilder.New()
            .AddCrud()
            .UseCrud()
            .Build();
            
        _service = _context.GetOrganizationService();
        _loggerMock = new Mock<ILogger<AccountPlugin>>();
    }
    
    [Fact]
    public void Should_Log_Information_When_Plugin_Executes()
    {
        // ARRANGE
        var account = new Account { Name = "Contoso" };
        _context.Initialize(new[] { account });
        
        var plugin = new AccountPlugin(_loggerMock.Object);
        
        // ACT
        _context.ExecutePluginWith<AccountPlugin>(
            plugin,
            account.ToEntity<Entity>(),
            "Create",
            ProcessingStepStage.Postoperation
        );
        
        // ASSERT
        _loggerMock.Verify(
            logger => logger.Log(
                LogLevel.Information,
                It.IsAny<EventId>(),
                It.Is<It.IsAnyType>((v, t) => v.ToString().Contains("Plugin execution started")),
                null,
                It.IsAny<Func<It.IsAnyType, Exception, string>>()
            ),
            Times.Once
        );
    }
}
```

### Verify Log Levels

```csharp
[Fact]
public void Should_Log_Different_Log_Levels()
{
    // ARRANGE
    var account = new Account { Name = "Contoso", CreditOnHold = true };
    _context.Initialize(new[] { account });
    
    var plugin = new AccountValidationPlugin(_loggerMock.Object);
    
    // ACT
    _context.ExecutePluginWith<AccountValidationPlugin>(
        plugin,
        account.ToEntity<Entity>(),
        "Update",
        ProcessingStepStage.Preoperation
    );
    
    // ASSERT
    // Verify Information log
    _loggerMock.Verify(
        logger => logger.Log(
            LogLevel.Information,
            It.IsAny<EventId>(),
            It.IsAny<It.IsAnyType>(),
            null,
            It.IsAny<Func<It.IsAnyType, Exception, string>>()
        ),
        Times.AtLeastOnce
    );
    
    // Verify Warning log
    _loggerMock.Verify(
        logger => logger.Log(
            LogLevel.Warning,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((v, t) => v.ToString().Contains("Credit is on hold")),
            null,
            It.IsAny<Func<It.IsAnyType, Exception, string>>()
        ),
        Times.Once
    );
}
```

### Verify Structured Logging

```csharp
public class CreateContactPlugin : IPlugin
{
    private readonly ILogger<CreateContactPlugin> _logger;
    
    public CreateContactPlugin(ILogger<CreateContactPlugin> logger = null)
    {
        _logger = logger;
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var target = (Entity)context.InputParameters["Target"];
        
        var firstName = target.GetAttributeValue<string>("firstname");
        var lastName = target.GetAttributeValue<string>("lastname");
        var email = target.GetAttributeValue<string>("emailaddress1");
        
        // Structured logging with named parameters
        _logger?.LogInformation(
            "Creating contact: {FirstName} {LastName}, Email: {Email}, User: {UserId}",
            firstName,
            lastName,
            email,
            context.UserId
        );
    }
}

[Fact]
public void Should_Log_Structured_Data()
{
    // ARRANGE
    var userId = Guid.NewGuid();
    var context = MiddlewareBuilder.New()
        .AddCrud()
        .SetCallerProperties(new CallerProperties { CallerId = userId })
        .UseCrud()
        .Build();
    
    var service = context.GetOrganizationService();
    var loggerMock = new Mock<ILogger<CreateContactPlugin>>();
    
    var contact = new Contact
    {
        FirstName = "John",
        LastName = "Doe",
        EMailAddress1 = "john@example.com"
    };
    
    var plugin = new CreateContactPlugin(loggerMock.Object);
    
    // ACT
    context.ExecutePluginWith<CreateContactPlugin>(
        plugin,
        contact.ToEntity<Entity>(),
        "Create",
        ProcessingStepStage.Postoperation
    );
    
    // ASSERT - Verify structured parameters
    loggerMock.Verify(
        logger => logger.Log(
            LogLevel.Information,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((v, t) => 
                v.ToString().Contains("John") &&
                v.ToString().Contains("Doe") &&
                v.ToString().Contains("john@example.com")
            ),
            null,
            It.IsAny<Func<It.IsAnyType, Exception, string>>()
        ),
        Times.Once
    );
}
```

## Testing Error Logging

### Exception Logging

```csharp
public class ErrorHandlingPlugin : IPlugin
{
    private readonly ILogger<ErrorHandlingPlugin> _logger;
    
    public ErrorHandlingPlugin(ILogger<ErrorHandlingPlugin> logger = null)
    {
        _logger = logger;
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        try
        {
            _logger?.LogInformation("Starting error handling plugin");
            
            // Simulate error condition
            var target = (Entity)context.InputParameters["Target"];
            if (target.GetAttributeValue<string>("name") == null)
            {
                throw new InvalidPluginExecutionException("Name is required");
            }
        }
        catch (Exception ex)
        {
            _logger?.LogError(ex, "Plugin execution failed for entity {EntityId}", context.PrimaryEntityId);
            throw;
        }
    }
}

[Fact]
public void Should_Log_Error_With_Exception()
{
    // ARRANGE
    var account = new Account { Id = Guid.NewGuid() }; // Missing name
    _context.Initialize(new[] { account });
    
    var plugin = new ErrorHandlingPlugin(_loggerMock.Object);
    
    // ACT & ASSERT
    Assert.Throws<InvalidPluginExecutionException>(() =>
        _context.ExecutePluginWith<ErrorHandlingPlugin>(
            plugin,
            account.ToEntity<Entity>(),
            "Create",
            ProcessingStepStage.Preoperation
        )
    );
    
    // Verify error was logged with exception
    _loggerMock.Verify(
        logger => logger.Log(
            LogLevel.Error,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((v, t) => v.ToString().Contains("Plugin execution failed")),
            It.IsAny<InvalidPluginExecutionException>(),
            It.IsAny<Func<It.IsAnyType, Exception, string>>()
        ),
        Times.Once
    );
}
```

## Log Scopes

### Using Log Scopes for Context

```csharp
public class ScopedLoggingPlugin : IPlugin
{
    private readonly ILogger<ScopedLoggingPlugin> _logger;
    
    public ScopedLoggingPlugin(ILogger<ScopedLoggingPlugin> logger = null)
    {
        _logger = logger;
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        using (_logger?.BeginScope("CorrelationId: {CorrelationId}", context.CorrelationId))
        using (_logger?.BeginScope("UserId: {UserId}", context.UserId))
        {
            _logger?.LogInformation("Processing within scope");
            
            // All logs within this scope will include correlation ID and user ID
            ProcessAccount(context);
        }
    }
    
    private void ProcessAccount(IPluginExecutionContext context)
    {
        _logger?.LogDebug("Processing account logic");
        // Scope context automatically included
    }
}

[Fact]
public void Should_Use_Log_Scopes()
{
    // ARRANGE
    var account = new Account { Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    var plugin = new ScopedLoggingPlugin(_loggerMock.Object);
    
    // ACT
    _context.ExecutePluginWith<ScopedLoggingPlugin>(
        plugin,
        account.ToEntity<Entity>(),
        "Create",
        ProcessingStepStage.Postoperation
    );
    
    // ASSERT
    _loggerMock.Verify(
        logger => logger.BeginScope(It.IsAny<string>()),
        Times.AtLeast(2) // CorrelationId and UserId scopes
    );
}
```

## Performance Logging

### Execution Time Tracking

```csharp
public class PerformancePlugin : IPlugin
{
    private readonly ILogger<PerformancePlugin> _logger;
    
    public PerformancePlugin(ILogger<PerformancePlugin> logger = null)
    {
        _logger = logger;
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        try
        {
            _logger?.LogInformation("Plugin execution started");
            
            // Plugin logic
            System.Threading.Thread.Sleep(100); // Simulate work
            
            stopwatch.Stop();
            _logger?.LogInformation(
                "Plugin execution completed in {ElapsedMilliseconds}ms",
                stopwatch.ElapsedMilliseconds
            );
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger?.LogError(
                ex,
                "Plugin execution failed after {ElapsedMilliseconds}ms",
                stopwatch.ElapsedMilliseconds
            );
            throw;
        }
    }
}

[Fact]
public void Should_Log_Execution_Time()
{
    // ARRANGE
    var account = new Account { Name = "Contoso" };
    _context.Initialize(new[] { account });
    
    var plugin = new PerformancePlugin(_loggerMock.Object);
    
    // ACT
    _context.ExecutePluginWith<PerformancePlugin>(
        plugin,
        account.ToEntity<Entity>(),
        "Create",
        ProcessingStepStage.Postoperation
    );
    
    // ASSERT - Verify execution time was logged
    _loggerMock.Verify(
        logger => logger.Log(
            LogLevel.Information,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((v, t) => 
                v.ToString().Contains("completed in") &&
                v.ToString().Contains("ms")
            ),
            null,
            It.IsAny<Func<It.IsAnyType, Exception, string>>()
        ),
        Times.Once
    );
}
```

## Best Practices

1. **Use structured logging** - Named parameters instead of string concatenation
2. **Choose appropriate log levels** - Debug for dev, Information for production, Warning/Error for issues
3. **Log correlation IDs** - Use BeginScope for context tracking
4. **Log exceptions properly** - Include exception object in LogError
5. **Avoid logging sensitive data** - PII, passwords, API keys
6. **Use constructor injection** - Easier to test with mocked logger
7. **Fallback to ITracingService** - For compatibility with older environments
8. **Test log output** - Verify critical logs are written
9. **Performance metrics** - Track execution time for slow operations
10. **Use EventIds** - For categorizing log entries

## Common Patterns

### Logger Helper

```csharp
public static class PluginLoggerExtensions
{
    public static void LogPluginStart(this ILogger logger, IPluginExecutionContext context)
    {
        logger?.LogInformation(
            "Plugin started: {MessageName}, Entity: {EntityName}, Stage: {Stage}, Mode: {Mode}",
            context.MessageName,
            context.PrimaryEntityName,
            context.Stage,
            context.Mode
        );
    }
    
    public static void LogPluginEnd(this ILogger logger, IPluginExecutionContext context, long elapsedMs)
    {
        logger?.LogInformation(
            "Plugin completed in {ElapsedMs}ms: {MessageName}, Entity: {EntityName}",
            elapsedMs,
            context.MessageName,
            context.PrimaryEntityName
        );
    }
}

[Fact]
public void Should_Use_Logger_Extensions()
{
    // Usage in plugin:
    // _logger.LogPluginStart(context);
    // ... logic ...
    // _logger.LogPluginEnd(context, elapsed);
}
```

### Conditional Logging

```csharp
public class ConditionalLoggingPlugin : IPlugin
{
    private readonly ILogger<ConditionalLoggingPlugin> _logger;
    
    public ConditionalLoggingPlugin(ILogger<ConditionalLoggingPlugin> logger = null)
    {
        _logger = logger;
    }
    
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        // Only log if Debug level is enabled (performance optimization)
        if (_logger?.IsEnabled(LogLevel.Debug) == true)
        {
            var target = (Entity)context.InputParameters["Target"];
            _logger.LogDebug(
                "Target entity attributes: {AttributeCount}",
                target.Attributes.Count
            );
        }
        
        // Always log critical information
        _logger?.LogInformation("Processing {EntityName}", context.PrimaryEntityName);
    }
}
```

## Integration with Application Insights

```csharp
// In production, configure Application Insights
// services.AddApplicationInsightsTelemetry();
// ILogger will automatically send to App Insights

[Fact]
public void Should_Work_With_Application_Insights_Logger()
{
    // In tests, use mock logger
    // In production, actual ILogger sends to App Insights
    
    var plugin = new AccountPlugin(_loggerMock.Object);
    // Test as normal
}
```

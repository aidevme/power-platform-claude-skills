# Setup and Configuration

This guide covers installing FakeXrmEasy, structuring test projects, and configuring the test environment.

## Package Installation

FakeXrmEasy has two major version lines based on your target framework:

### Version 2.x — .NET Framework

Use for traditional plugin projects targeting .NET Framework 4.6.2+. These versions depend on
`Microsoft.CrmSdk.CoreAssemblies`.

```powershell
# For Dataverse / Dynamics 365 (v9.x)
Install-Package FakeXrmEasy.Plugins.v9 -Version 2.6.3

# For Dynamics 365 v8.2
Install-Package FakeXrmEasy.Plugins.v365 -Version 2.x

# For older versions, replace v9 with v2016, v2015, v2013, or v2011
```

**When to use:**
- Server-side plugin development (most common)
- Plugin projects that must target .NET Framework
- Testing traditional workflow activities (CodeActivities)

### Version 3.x — .NET Core 3.1+

Use for modern applications targeting .NET Core/5+. These versions use the Dataverse Service Client.

```powershell
Install-Package FakeXrmEasy.Plugins.v9 -Version 3.5.3
```

**When to use:**
- Azure Functions that interact with Dataverse
- .NET Core/5/6/7 applications
- Modern client applications (NOT for plugins deployed to Dataverse)

### What Gets Installed

The `FakeXrmEasy.Plugins` package pulls in:
- `FakeXrmEasy.Core` — In-memory context and service implementations
- `FakeXrmEasy.Messages` — Standard message executors (Create, Update, etc.)
- Plugin-specific middleware and extension methods

## Project Structure

Recommended solution structure:

```
MySolution/
├── MyPlugins/                  # Plugin implementation project
│   ├── MyPlugins.csproj        # .NET Framework 4.6.2
│   ├── Plugins/
│   │   ├── AccountNumberPlugin.cs
│   │   └── ValidateContactPlugin.cs
│   └── EarlyBound/
│       └── Entities.cs         # Optional: generated entities
│
└── MyPlugins.Tests/             # Test project
    ├── MyPlugins.Tests.csproj   # Same framework as MyPlugins
    ├── AccountNumberPluginTests.cs
    ├── ValidateContactPluginTests.cs
    └── TestBase.cs              # Optional: shared base class
```

### Test Project Configuration

**MyPlugins.Tests.csproj** (SDK-style project):

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net462</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="FakeXrmEasy.Plugins.v9" Version="2.6.3" />
    <PackageReference Include="xunit" Version="2.4.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.4.5" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.5.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyPlugins\MyPlugins.csproj" />
  </ItemGroup>
</Project>
```

## Test Runners

FakeXrmEasy works with all major .NET test frameworks:

### xUnit (Recommended)

Most popular for .NET plugin testing. Clean, extensible, parallel execution.

> **Note:** `FakeXrmEasyTestsBase` does NOT exist in FakeXrmEasy v2.x or v3.x NuGet packages.
> Always build the context manually via `MiddlewareBuilder`.

```csharp
using Xunit;
using FakeXrmEasy.Abstractions;
using FakeXrmEasy.Abstractions.Enums;
using FakeXrmEasy.Middleware;
using FakeXrmEasy.Middleware.Crud;
using FakeXrmEasy.Plugins;

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

    [Fact]
    public void Should_Generate_Account_Number_On_Create()
    {
        // Test implementation
    }

    [Theory]
    [InlineData("ACC-12345678")]
    [InlineData("ACC-ABCDEF12")]
    public void Should_Preserve_Existing_Account_Number(string existingNumber)
    {
        // Theory tests multiple scenarios
    }
}
```

### NUnit

Alternative with similar features. Use `[TestFixture]` instead of class-level attributes.

```csharp
using NUnit.Framework;
using FakeXrmEasy.Abstractions;
using FakeXrmEasy.Abstractions.Enums;
using FakeXrmEasy.Middleware;
using FakeXrmEasy.Middleware.Crud;

[TestFixture]
public class AccountNumberPluginTests
{
    private IXrmFakedContext _context;

    [SetUp]
    public void SetUp()
    {
        _context = MiddlewareBuilder
            .New()
            .AddCrud()
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();
    }

    [Test]
    public void Should_Generate_Account_Number_On_Create()
    {
        // Test implementation
    }
}
```

### MSTest

Built into Visual Studio. Use `[TestClass]` and `[TestMethod]`.

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;
using FakeXrmEasy.Abstractions;
using FakeXrmEasy.Abstractions.Enums;
using FakeXrmEasy.Middleware;
using FakeXrmEasy.Middleware.Crud;

[TestClass]
public class AccountNumberPluginTests
{
    private IXrmFakedContext _context;

    [TestInitialize]
    public void TestInitialize()
    {
        _context = MiddlewareBuilder
            .New()
            .AddCrud()
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();
    }

    [TestMethod]
    public void Should_Generate_Account_Number_On_Create()
    {
        // Test implementation
    }
}
```

## Base Test Class

Create a shared base class for common setup:

```csharp
using FakeXrmEasy.Abstractions;
using FakeXrmEasy.Abstractions.Enums;
using FakeXrmEasy.Middleware;
using FakeXrmEasy.Middleware.Crud;
using System.Reflection;

public abstract class PluginTestBase
{
    protected readonly IXrmFakedContext _context;

    protected PluginTestBase()
    {
        _context = MiddlewareBuilder
            .New()
            .AddCrud()
            .SetLicense(FakeXrmEasyLicense.RPL_1_5)
            .Build();

        // Enable early-bound entities (optional)
        _context.EnableProxyTypes(Assembly.GetExecutingAssembly());
    }

    protected Guid CreateTestAccount(string name)
    {
        var account = new Account { Id = Guid.NewGuid(), Name = name };
        _context.Initialize(new[] { account });
        return account.Id;
    }

    protected Contact CreateTestContact(string firstName, string lastName)
    {
        var contact = new Contact
        {
            Id = Guid.NewGuid(),
            FirstName = firstName,
            LastName = lastName
        };
        _context.Initialize(new[] { contact });
        return contact;
    }
}
```

Then inherit from your base class:

```csharp
public class AccountNumberPluginTests : PluginTestBase
{
    [Fact]
    public void Test_Something()
    {
        var accountId = CreateTestAccount("Contoso");
        // Use accountId in test
    }
}
```

## Early-Bound Entities

### Generating Entities

Use the Power Platform CLI to generate early-bound classes:

```powershell
# Install Power Platform CLI
dotnet tool install --global Microsoft.PowerApps.CLI.Tool

# Generate entities
pac modelbuilder build `
  --outdirectory ".\EarlyBound" `
  --namespace "MyPlugins.EarlyBound" `
  --entities "account,contact,opportunity" `
  --schemaversion "latest"
```

### Using Early-Bound Entities in Tests

1. **Enable proxy types in test constructor:**

```csharp
public MyPluginTests()
{
    _context.EnableProxyTypes(Assembly.GetExecutingAssembly());
}
```

2. **Create entities using strongly-typed classes:**

```csharp
var account = new Account
{
    Id = Guid.NewGuid(),
    Name = "Contoso",
    Revenue = new Money(1000000)
};

_context.Initialize(new[] { account });
```

3. **Query with LINQ and early-bound properties:**

```csharp
var accounts = _context.CreateQuery<Account>()
    .Where(a => a.Revenue.Value > 500000)
    .ToList();
```

## Common Configuration Patterns

### Setting Organization ID

Some plugins check the organization ID:

```csharp
_context.OrganizationId = new Guid("12345678-1234-1234-1234-123456789012");
```

### Setting User Context

Test plugins that behave differently based on user:

```csharp
var userId = Guid.NewGuid();
var user = new SystemUser 
{ 
    Id = userId, 
    FirstName = "Test", 
    LastName = "User" 
};

_context.Initialize(new[] { user });
_context.CallerId = userId;
```

### Configuring Time Zone

For plugins that work with dates:

```csharp
_context.UserTimeZoneCode = 85; // Pacific Standard Time
```

## Troubleshooting

### "Entity with logical name X doesn't contain attribute Y"

- Early-bound entities not enabled. Call `_context.EnableProxyTypes(Assembly)`.
- Or use late-bound entities: `new Entity("account") { ["name"] = "Contoso" }`

### "Message X is not supported"

- Install `FakeXrmEasy.Messages.v9` if using only Core package.
- Custom messages/APIs require additional configuration. See `advanced-scenarios.md`.

### Tests run slow

- Create the context once per class in the constructor (xUnit) or `[SetUp]`/`[TestInitialize]` (NUnit/MSTest).
- Use `Initialize()` with small datasets, not thousands of records.
- Parallelize tests (xUnit does this by default).

### "The given key was not present in the dictionary"

- Plugin expects InputParameters that aren't set in test.
- Check `pluginContext.InputParameters["Target"]` and other expected parameters.

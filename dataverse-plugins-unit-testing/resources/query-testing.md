# Query Testing (FetchXML, QueryExpression, LINQ)

Comprehensive guide for testing Dataverse queries including FetchXML, QueryExpression,
and LINQ to CRM queries with FakeXrmEasy.

## Overview

Query types in Dataverse:
- **QueryExpression** - Object-based query API (most common in plugins)
- **FetchXML** - XML-based query language (declarative)
- **LINQ** - Language Integrated Query (type-safe, early-bound)
- **QueryByAttribute** - Simple attribute-value queries

## QueryExpression Testing

### Basic Query

```csharp
[Fact]
public void Should_Query_Active_Accounts()
{
    // ARRANGE
    var account1 = new Account
    {
        Id = Guid.NewGuid(),
        Name = "Contoso",
        StateCode = AccountState.Active
    };
    
    var account2 = new Account
    {
        Id = Guid.NewGuid(),
        Name = "Fabrikam",
        StateCode = AccountState.Inactive
    };
    
    _context.Initialize(new[] { account1, account2 });
    
    // ACT
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name", "statecode");
    query.Criteria.AddCondition("statecode", ConditionOperator.Equal, 0); // Active
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Single(results.Entities);
    Assert.Equal("Contoso", results.Entities[0].GetAttributeValue<string>("name"));
}
```

### Query with Multiple Conditions

```csharp
[Fact]
public void Should_Query_With_Multiple_Conditions()
{
    // ARRANGE
    var contact1 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "John",
        LastName = "Doe",
        StateCode = ContactState.Active,
        Address1_City = "Seattle"
    };
    
    var contact2 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "Jane",
        LastName = "Smith",
        StateCode = ContactState.Active,
        Address1_City = "Portland"
    };
    
    _context.Initialize(new[] { contact1, contact2 });
    
    // ACT
    var query = new QueryExpression("contact");
    query.ColumnSet = new ColumnSet(true);
    query.Criteria.FilterOperator = LogicalOperator.And;
    query.Criteria.AddCondition("statecode", ConditionOperator.Equal, 0);
    query.Criteria.AddCondition("address1_city", ConditionOperator.Equal, "Seattle");
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Single(results.Entities);
    Assert.Equal("John", results.Entities[0].GetAttributeValue<string>("firstname"));
}
```

### Query with OR Logic

```csharp
[Fact]
public void Should_Query_With_OR_Conditions()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso", IndustryCode = new OptionSetValue(1) };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam", IndustryCode = new OptionSetValue(2) };
    var account3 = new Account { Id = Guid.NewGuid(), Name = "Adventure Works", IndustryCode = new OptionSetValue(3) };
    
    _context.Initialize(new[] { account1, account2, account3 });
    
    // ACT
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name", "industrycode");
    query.Criteria.FilterOperator = LogicalOperator.Or;
    query.Criteria.AddCondition("industrycode", ConditionOperator.Equal, 1);
    query.Criteria.AddCondition("industrycode", ConditionOperator.Equal, 2);
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count);
    Assert.Contains(results.Entities, e => e.GetAttributeValue<string>("name") == "Contoso");
    Assert.Contains(results.Entities, e => e.GetAttributeValue<string>("name") == "Fabrikam");
}
```

### Query with Nested Filters

```csharp
[Fact]
public void Should_Query_With_Nested_Filters()
{
    // ARRANGE
    var contact1 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "John",
        StateCode = ContactState.Active,
        Address1_City = "Seattle"
    };
    
    var contact2 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "Jane",
        StateCode = ContactState.Active,
        Address1_City = "Portland"
    };
    
    var contact3 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "Bob",
        StateCode = ContactState.Inactive,
        Address1_City = "Seattle"
    };
    
    _context.Initialize(new[] { contact1, contact2, contact3 });
    
    // ACT - (Active AND (Seattle OR Portland))
    var query = new QueryExpression("contact");
    query.ColumnSet = new ColumnSet("firstname", "statecode", "address1_city");
    
    query.Criteria.FilterOperator = LogicalOperator.And;
    query.Criteria.AddCondition("statecode", ConditionOperator.Equal, 0);
    
    var cityFilter = new FilterExpression(LogicalOperator.Or);
    cityFilter.AddCondition("address1_city", ConditionOperator.Equal, "Seattle");
    cityFilter.AddCondition("address1_city", ConditionOperator.Equal, "Portland");
    query.Criteria.AddFilter(cityFilter);
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count); // John and Jane (both active, in Seattle or Portland)
}
```

### Query with Operators

```csharp
[Theory]
[InlineData(ConditionOperator.GreaterThan, 100, 1)] // 150 > 100
[InlineData(ConditionOperator.LessThan, 100, 1)]    // 50 < 100
[InlineData(ConditionOperator.GreaterEqual, 150, 1)] // 150 >= 150
[InlineData(ConditionOperator.Between, null, 2)]    // Both in range 75-175
public void Should_Query_With_Numeric_Operators(ConditionOperator op, int? value, int expectedCount)
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Account1", NumberOfEmployees = 50 };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Account2", NumberOfEmployees = 150 };
    _context.Initialize(new[] { account1, account2 });
    
    // ACT
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name", "numberofemployees");
    
    if (op == ConditionOperator.Between)
    {
        query.Criteria.AddCondition("numberofemployees", op, 75, 175);
    }
    else
    {
        query.Criteria.AddCondition("numberofemployees", op, value);
    }
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(expectedCount, results.Entities.Count);
}
```

### Query with IN Operator

```csharp
[Fact]
public void Should_Query_With_IN_Operator()
{
    // ARRANGE
    var id1 = Guid.NewGuid();
    var id2 = Guid.NewGuid();
    var id3 = Guid.NewGuid();
    
    var account1 = new Account { Id = id1, Name = "Contoso" };
    var account2 = new Account { Id = id2, Name = "Fabrikam" };
    var account3 = new Account { Id = id3, Name = "Adventure Works" };
    
    _context.Initialize(new[] { account1, account2, account3 });
    
    // ACT - Query for specific IDs
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name");
    query.Criteria.AddCondition("accountid", ConditionOperator.In, id1, id3);
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count);
    Assert.Contains(results.Entities, e => e.Id == id1);
    Assert.Contains(results.Entities, e => e.Id == id3);
}
```

### Query with Paging

```csharp
[Fact]
public void Should_Query_With_Paging()
{
    // ARRANGE
    var accounts = Enumerable.Range(1, 25)
        .Select(i => new Account { Id = Guid.NewGuid(), Name = $"Account {i}" })
        .ToArray();
    
    _context.Initialize(accounts);
    
    // ACT - Get first page (10 records)
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name");
    query.PageInfo = new PagingInfo
    {
        Count = 10,
        PageNumber = 1
    };
    
    var page1 = _service.RetrieveMultiple(query);
    
    // Get second page
    query.PageInfo.PageNumber = 2;
    query.PageInfo.PagingCookie = page1.PagingCookie;
    var page2 = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(10, page1.Entities.Count);
    Assert.Equal(10, page2.Entities.Count);
    Assert.True(page1.MoreRecords);
    Assert.True(page2.MoreRecords);
}
```

## FetchXML Testing

### Basic FetchXML Query

```csharp
[Fact]
public void Should_Execute_FetchXML_Query()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso", StateCode = AccountState.Active };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam", StateCode = AccountState.Inactive };
    _context.Initialize(new[] { account1, account2 });
    
    // ACT
    var fetchXml = @"
        <fetch>
            <entity name='account'>
                <attribute name='name' />
                <attribute name='statecode' />
                <filter type='and'>
                    <condition attribute='statecode' operator='eq' value='0' />
                </filter>
            </entity>
        </fetch>";
    
    var results = _service.RetrieveMultiple(new FetchExpression(fetchXml));
    
    // ASSERT
    Assert.Single(results.Entities);
    Assert.Equal("Contoso", results.Entities[0].GetAttributeValue<string>("name"));
}
```

### FetchXML with Link Entity

```csharp
[Fact]
public void Should_Execute_FetchXML_With_Link_Entity()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    
    var contact1 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "John",
        ParentCustomerId = account.ToEntityReference()
    };
    
    var contact2 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "Jane",
        ParentCustomerId = account.ToEntityReference()
    };
    
    _context.Initialize(new Entity[] { account, contact1, contact2 });
    
    // ACT
    var fetchXml = $@"
        <fetch>
            <entity name='contact'>
                <attribute name='firstname' />
                <link-entity name='account' from='accountid' to='parentcustomerid' alias='acct'>
                    <attribute name='name' />
                    <filter>
                        <condition attribute='accountid' operator='eq' value='{accountId}' />
                    </filter>
                </link-entity>
            </entity>
        </fetch>";
    
    var results = _service.RetrieveMultiple(new FetchExpression(fetchXml));
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count);
    Assert.All(results.Entities, contact =>
    {
        var accountName = contact.GetAttributeValue<AliasedValue>("acct.name")?.Value;
        Assert.Equal("Contoso", accountName);
    });
}
```

### FetchXML with Aggregation

```csharp
[Fact]
public void Should_Execute_FetchXML_With_Aggregation()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    
    var opp1 = new Opportunity { Id = Guid.NewGuid(), ParentAccountId = account.ToEntityReference(), EstimatedValue = new Money(1000) };
    var opp2 = new Opportunity { Id = Guid.NewGuid(), ParentAccountId = account.ToEntityReference(), EstimatedValue = new Money(2000) };
    var opp3 = new Opportunity { Id = Guid.NewGuid(), ParentAccountId = account.ToEntityReference(), EstimatedValue = new Money(3000) };
    
    _context.Initialize(new Entity[] { account, opp1, opp2, opp3 });
    
    // ACT - Calculate total opportunity value
    var fetchXml = $@"
        <fetch aggregate='true'>
            <entity name='opportunity'>
                <attribute name='estimatedvalue' alias='total_value' aggregate='sum' />
                <attribute name='opportunityid' alias='count' aggregate='count' />
                <filter>
                    <condition attribute='parentaccountid' operator='eq' value='{accountId}' />
                </filter>
            </entity>
        </fetch>";
    
    var results = _service.RetrieveMultiple(new FetchExpression(fetchXml));
    
    // ASSERT
    Assert.Single(results.Entities);
    var totalValue = ((Money)((AliasedValue)results.Entities[0]["total_value"]).Value).Value;
    var count = (int)((AliasedValue)results.Entities[0]["count"]).Value;
    
    Assert.Equal(6000, totalValue);
    Assert.Equal(3, count);
}
```

## LINQ Query Testing

### Basic LINQ Query

```csharp
[Fact]
public void Should_Execute_LINQ_Query()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso", StateCode = AccountState.Active };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam", StateCode = AccountState.Inactive };
    _context.Initialize(new[] { account1, account2 });
    
    // ACT - LINQ query
    var query = from a in _context.CreateQuery<Account>()
                where a.StateCode == AccountState.Active
                select new { a.Name, a.StateCode };
    
    var results = query.ToList();
    
    // ASSERT
    Assert.Single(results);
    Assert.Equal("Contoso", results[0].Name);
}
```

### LINQ with Multiple Conditions

```csharp
[Fact]
public void Should_Execute_LINQ_With_Multiple_Conditions()
{
    // ARRANGE
    var account1 = new Account
    {
        Id = Guid.NewGuid(),
        Name = "Contoso",
        StateCode = AccountState.Active,
        NumberOfEmployees = 100
    };
    
    var account2 = new Account
    {
        Id = Guid.NewGuid(),
        Name = "Fabrikam",
        StateCode = AccountState.Active,
        NumberOfEmployees = 500
    };
    
    _context.Initialize(new[] { account1, account2 });
    
    // ACT
    var accounts = _context.CreateQuery<Account>()
        .Where(a => a.StateCode == AccountState.Active)
        .Where(a => a.NumberOfEmployees > 200)
        .ToList();
    
    // ASSERT
    Assert.Single(accounts);
    Assert.Equal("Fabrikam", accounts[0].Name);
}
```

### LINQ with Join

```csharp
[Fact]
public void Should_Execute_LINQ_With_Join()
{
    // ARRANGE
    var account = new Account { Id = Guid.NewGuid(), Name = "Contoso" };
    var contact1 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "John",
        ParentCustomerId = account.ToEntityReference()
    };
    var contact2 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "Jane",
        ParentCustomerId = account.ToEntityReference()
    };
    
    _context.Initialize(new Entity[] { account, contact1, contact2 });
    
    // ACT - Join contacts with accounts
    var query = from c in _context.CreateQuery<Contact>()
                join a in _context.CreateQuery<Account>()
                    on c.ParentCustomerId.Id equals a.AccountId
                where a.Name == "Contoso"
                select new
                {
                    ContactName = c.FirstName,
                    AccountName = a.Name
                };
    
    var results = query.ToList();
    
    // ASSERT
    Assert.Equal(2, results.Count);
    Assert.All(results, r => Assert.Equal("Contoso", r.AccountName));
}
```

### LINQ with Ordering and Paging

```csharp
[Fact]
public void Should_Execute_LINQ_With_Ordering_And_Paging()
{
    // ARRANGE
    var accounts = Enumerable.Range(1, 20)
        .Select(i => new Account
        {
            Id = Guid.NewGuid(),
            Name = $"Account {i:D2}",
            NumberOfEmployees = i * 10
        })
        .ToArray();
    
    _context.Initialize(accounts);
    
    // ACT - Order by employees descending, take top 5
    var topAccounts = _context.CreateQuery<Account>()
        .OrderByDescending(a => a.NumberOfEmployees)
        .Take(5)
        .ToList();
    
    // ASSERT
    Assert.Equal(5, topAccounts.Count);
    Assert.Equal(200, topAccounts[0].NumberOfEmployees);
    Assert.Equal(190, topAccounts[1].NumberOfEmployees);
}
```

## QueryByAttribute Testing

```csharp
[Fact]
public void Should_Execute_QueryByAttribute()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso", Telephone1 = "555-1234" };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam", Telephone1 = "555-5678" };
    _context.Initialize(new[] { account1, account2 });
    
    // ACT
    var query = new QueryByAttribute("account");
    query.ColumnSet = new ColumnSet("name", "telephone1");
    query.Attributes.Add("telephone1");
    query.Values.Add("555-1234");
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Single(results.Entities);
    Assert.Equal("Contoso", results.Entities[0].GetAttributeValue<string>("name"));
}
```

## Best Practices

1. **Test query filters** - Verify correct records are returned
2. **Test empty results** - Ensure queries handle no matches gracefully
3. **Test paging** - Validate large result sets are paginated correctly
4. **Test column sets** - Verify only requested columns are returned
5. **Test joins** - Ensure link entities retrieve related data correctly
6. **Test ordering** - Validate sort order is correct
7. **Test aggregations** - Verify sum, count, min, max work correctly
8. **Use early-bound entities** - LINQ queries benefit from compile-time checking
9. **Test performance** - Large datasets should use efficient queries
10. **Test NULL handling** - Verify queries handle NULL values correctly

## Common Patterns

### Reusable Query Builder

```csharp
public class AccountQueryBuilder
{
    private readonly QueryExpression _query;
    
    public AccountQueryBuilder()
    {
        _query = new QueryExpression("account");
        _query.ColumnSet = new ColumnSet(true);
    }
    
    public AccountQueryBuilder WhereActive()
    {
        _query.Criteria.AddCondition("statecode", ConditionOperator.Equal, 0);
        return this;
    }
    
    public AccountQueryBuilder WhereNameContains(string namepart)
    {
        _query.Criteria.AddCondition("name", ConditionOperator.Like, $"%{namepart}%");
        return this;
    }
    
    public QueryExpression Build() => _query;
}

[Fact]
public void Should_Use_Query_Builder()
{
    var query = new AccountQueryBuilder()
        .WhereActive()
        .WhereNameContains("Contoso")
        .Build();
    
    var results = _service.RetrieveMultiple(query);
    // Assert...
}
```

### Query Result Mapper

```csharp
public static class QueryResultExtensions
{
    public static List<T> ToList<T>(this EntityCollection results) where T : Entity
    {
        return results.Entities.Select(e => e.ToEntity<T>()).ToList();
    }
    
    public static T FirstOrDefault<T>(this EntityCollection results) where T : Entity
    {
        return results.Entities.FirstOrDefault()?.ToEntity<T>();
    }
}

[Fact]
public void Should_Use_Result_Mapper()
{
    var query = new QueryExpression("account");
    var results = _service.RetrieveMultiple(query);
    
    var accounts = results.ToList<Account>();
    var firstAccount = results.FirstOrDefault<Account>();
}
```

# Testing Data & Relationships

Comprehensive guide for testing Dataverse records with complex relationships, including
1:N (One-to-Many), N:1 (Many-to-One), and N:N (Many-to-Many) relationships.

## Overview

Relationship testing scenarios:
- Entity References (lookups)
- Associate/Disassociate operations
- N:N relationships
- Parent-child hierarchies
- Cascading behaviors
- Related entity queries

## Entity References (Lookups)

### Creating Records with Lookups

```csharp
[Fact]
public void Should_Create_Contact_With_Account_Reference()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account
    {
        Id = accountId,
        Name = "Contoso"
    };
    _context.Initialize(new[] { account });
    
    var contact = new Contact
    {
        FirstName = "John",
        LastName = "Doe",
        ParentCustomerId = account.ToEntityReference() // Lookup to account
    };
    
    // ACT
    var contactId = _service.Create(contact);
    
    // ASSERT
    var createdContact = _service.Retrieve("contact", contactId, new ColumnSet(true))
        .ToEntity<Contact>();
    
    Assert.NotNull(createdContact.ParentCustomerId);
    Assert.Equal(accountId, createdContact.ParentCustomerId.Id);
    Assert.Equal("account", createdContact.ParentCustomerId.LogicalName);
}
```

### Testing Null References

```csharp
[Fact]
public void Should_Allow_Null_Lookup_Field()
{
    // ARRANGE
    var contact = new Contact
    {
        FirstName = "John",
        LastName = "Doe",
        ParentCustomerId = null // No parent account
    };
    
    // ACT
    var contactId = _service.Create(contact);
    
    // ASSERT
    var created = _service.Retrieve("contact", contactId, new ColumnSet(true))
        .ToEntity<Contact>();
    Assert.Null(created.ParentCustomerId);
}
```

### Updating References

```csharp
[Fact]
public void Should_Update_Contact_Account_Reference()
{
    // ARRANGE
    var account1 = new Account { Id = Guid.NewGuid(), Name = "Contoso" };
    var account2 = new Account { Id = Guid.NewGuid(), Name = "Fabrikam" };
    var contact = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "John",
        ParentCustomerId = account1.ToEntityReference()
    };
    _context.Initialize(new Entity[] { account1, account2, contact });
    
    // ACT - Change parent account
    var contactUpdate = new Contact
    {
        Id = contact.Id,
        ParentCustomerId = account2.ToEntityReference()
    };
    _service.Update(contactUpdate);
    
    // ASSERT
    var updated = _service.Retrieve("contact", contact.Id, new ColumnSet("parentcustomerid"))
        .ToEntity<Contact>();
    Assert.Equal(account2.Id, updated.ParentCustomerId.Id);
}
```

## One-to-Many (1:N) Relationships

### Parent with Multiple Children

```csharp
[Fact]
public void Should_Query_Account_With_Related_Contacts()
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
    
    var contact3 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "Bob",
        ParentCustomerId = new EntityReference("account", Guid.NewGuid()) // Different account
    };
    
    _context.Initialize(new Entity[] { account, contact1, contact2, contact3 });
    
    // ACT - Query contacts for specific account
    var query = new QueryExpression("contact");
    query.ColumnSet = new ColumnSet("firstname", "lastname");
    query.Criteria.AddCondition("parentcustomerid", ConditionOperator.Equal, accountId);
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count);
    Assert.Contains(results.Entities, e => e.GetAttributeValue<string>("firstname") == "John");
    Assert.Contains(results.Entities, e => e.GetAttributeValue<string>("firstname") == "Jane");
}
```

### Deleting Parent with Cascade

```csharp
[Fact]
public void Should_Delete_Related_Contacts_When_Account_Deleted_With_Cascade()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    
    var contact = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "John",
        ParentCustomerId = account.ToEntityReference()
    };
    
    _context.Initialize(new Entity[] { account, contact });
    
    // Configure cascade delete behavior
    _context.AddRelationship("contact_customer_accounts", new XrmFakedRelationship
    {
        IntersectEntity = "contact",
        Entity1LogicalName = "account",
        Entity1Attribute = "accountid",
        Entity2LogicalName = "contact",
        Entity2Attribute = "parentcustomerid",
        RelationshipType = XrmFakedRelationship.FakeRelationshipType.OneToMany
    });
    
    // ACT
    _service.Delete("account", accountId);
    
    // ASSERT
    var accounts = _context.CreateQuery<Account>().ToList();
    var contacts = _context.CreateQuery<Contact>().ToList();
    
    Assert.Empty(accounts);
    // Note: Cascade behavior depends on FakeXrmEasy configuration
}
```

## Many-to-Many (N:N) Relationships

### Associate Records

```csharp
[Fact]
public void Should_Associate_Account_And_Contact()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var contactId = Guid.NewGuid();
    
    var account = new Account { Id = accountId, Name = "Contoso" };
    var contact = new Contact { Id = contactId, FirstName = "John", LastName = "Doe" };
    
    _context.Initialize(new Entity[] { account, contact });
    
    // Define N:N relationship
    _context.AddRelationship("account_contact", new XrmFakedRelationship
    {
        IntersectEntity = "accountcontact",
        Entity1LogicalName = "account",
        Entity1Attribute = "accountid",
        Entity2LogicalName = "contact",
        Entity2Attribute = "contactid",
        RelationshipType = XrmFakedRelationship.FakeRelationshipType.ManyToMany
    });
    
    // ACT
    var associateRequest = new AssociateRequest
    {
        Target = account.ToEntityReference(),
        Relationship = new Relationship("account_contact"),
        RelatedEntities = new EntityReferenceCollection
        {
            contact.ToEntityReference()
        }
    };
    
    _service.Execute(associateRequest);
    
    // ASSERT - Verify association exists
    var relationships = _context.GetRelationship("account_contact");
    Assert.NotNull(relationships);
}
```

### Disassociate Records

```csharp
[Fact]
public void Should_Disassociate_Account_And_Contact()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var contactId = Guid.NewGuid();
    
    var account = new Account { Id = accountId, Name = "Contoso" };
    var contact = new Contact { Id = contactId, FirstName = "John" };
    
    _context.Initialize(new Entity[] { account, contact });
    
    _context.AddRelationship("account_contact", new XrmFakedRelationship
    {
        IntersectEntity = "accountcontact",
        Entity1LogicalName = "account",
        Entity1Attribute = "accountid",
        Entity2LogicalName = "contact",
        Entity2Attribute = "contactid",
        RelationshipType = XrmFakedRelationship.FakeRelationshipType.ManyToMany
    });
    
    // Associate first
    _service.Execute(new AssociateRequest
    {
        Target = account.ToEntityReference(),
        Relationship = new Relationship("account_contact"),
        RelatedEntities = new EntityReferenceCollection { contact.ToEntityReference() }
    });
    
    // ACT - Disassociate
    var disassociateRequest = new DisassociateRequest
    {
        Target = account.ToEntityReference(),
        Relationship = new Relationship("account_contact"),
        RelatedEntities = new EntityReferenceCollection
        {
            contact.ToEntityReference()
        }
    };
    
    _service.Execute(disassociateRequest);
    
    // ASSERT - Verify no relationship
    var relationships = _context.GetRelationship("account_contact");
    // Verify relationship removed
}
```

### Query with N:N Relationships

```csharp
[Fact]
public void Should_Query_Contacts_Associated_With_Account()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var account = new Account { Id = accountId, Name = "Contoso" };
    
    var contact1 = new Contact { Id = Guid.NewGuid(), FirstName = "John" };
    var contact2 = new Contact { Id = Guid.NewGuid(), FirstName = "Jane" };
    var contact3 = new Contact { Id = Guid.NewGuid(), FirstName = "Bob" }; // Not associated
    
    _context.Initialize(new Entity[] { account, contact1, contact2, contact3 });
    
    _context.AddRelationship("account_contact", new XrmFakedRelationship
    {
        IntersectEntity = "accountcontact",
        Entity1LogicalName = "account",
        Entity1Attribute = "accountid",
        Entity2LogicalName = "contact",
        Entity2Attribute = "contactid",
        RelationshipType = XrmFakedRelationship.FakeRelationshipType.ManyToMany
    });
    
    // Associate contact1 and contact2 with account
    _service.Execute(new AssociateRequest
    {
        Target = account.ToEntityReference(),
        Relationship = new Relationship("account_contact"),
        RelatedEntities = new EntityReferenceCollection 
        { 
            contact1.ToEntityReference(),
            contact2.ToEntityReference()
        }
    });
    
    // ACT - Query associated contacts
    var query = new QueryExpression("contact");
    query.ColumnSet = new ColumnSet("firstname");
    
    var linkEntity = query.AddLink("accountcontact", "contactid", "contactid");
    linkEntity.LinkCriteria.AddCondition("accountid", ConditionOperator.Equal, accountId);
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count);
    Assert.Contains(results.Entities, e => e.GetAttributeValue<string>("firstname") == "John");
    Assert.Contains(results.Entities, e => e.GetAttributeValue<string>("firstname") == "Jane");
}
```

## Self-Referencing Relationships

### Parent-Child Hierarchy (Account)

```csharp
[Fact]
public void Should_Create_Account_Hierarchy()
{
    // ARRANGE
    var parentAccountId = Guid.NewGuid();
    var parentAccount = new Account
    {
        Id = parentAccountId,
        Name = "Contoso Corporation",
        ParentAccountId = null // Top-level
    };
    
    var childAccount1 = new Account
    {
        Id = Guid.NewGuid(),
        Name = "Contoso North America",
        ParentAccountId = parentAccount.ToEntityReference()
    };
    
    var childAccount2 = new Account
    {
        Id = Guid.NewGuid(),
        Name = "Contoso Europe",
        ParentAccountId = parentAccount.ToEntityReference()
    };
    
    var grandchildAccount = new Account
    {
        Id = Guid.NewGuid(),
        Name = "Contoso UK",
        ParentAccountId = childAccount2.ToEntityReference()
    };
    
    _context.Initialize(new Entity[] { parentAccount, childAccount1, childAccount2, grandchildAccount });
    
    // ACT - Query all child accounts of parent
    var query = new QueryExpression("account");
    query.ColumnSet = new ColumnSet("name", "parentaccountid");
    query.Criteria.AddCondition("parentaccountid", ConditionOperator.Equal, parentAccountId);
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Equal(2, results.Entities.Count); // Only direct children
    Assert.Contains(results.Entities, e => e.GetAttributeValue<string>("name") == "Contoso North America");
    Assert.Contains(results.Entities, e => e.GetAttributeValue<string>("name") == "Contoso Europe");
}
```

## Complex Relationship Queries

### Multiple Link Entities

```csharp
[Fact]
public void Should_Query_Contacts_With_Account_And_Primary_Contact()
{
    // ARRANGE
    var accountId = Guid.NewGuid();
    var primaryContactId = Guid.NewGuid();
    
    var primaryContact = new Contact
    {
        Id = primaryContactId,
        FirstName = "Primary",
        LastName = "Contact"
    };
    
    var account = new Account
    {
        Id = accountId,
        Name = "Contoso",
        PrimaryContactId = primaryContact.ToEntityReference()
    };
    
    var contact1 = new Contact
    {
        Id = Guid.NewGuid(),
        FirstName = "John",
        ParentCustomerId = account.ToEntityReference()
    };
    
    _context.Initialize(new Entity[] { primaryContact, account, contact1 });
    
    // ACT - Query contacts with account details and primary contact
    var query = new QueryExpression("contact");
    query.ColumnSet = new ColumnSet("firstname", "lastname");
    query.Criteria.AddCondition("parentcustomerid", ConditionOperator.Equal, accountId);
    
    var accountLink = query.AddLink("account", "parentcustomerid", "accountid");
    accountLink.Columns = new ColumnSet("name");
    accountLink.EntityAlias = "acct";
    
    var primaryContactLink = accountLink.AddLink("contact", "primarycontactid", "contactid", JoinOperator.LeftOuter);
    primaryContactLink.Columns = new ColumnSet("firstname", "lastname");
    primaryContactLink.EntityAlias = "primary";
    
    var results = _service.RetrieveMultiple(query);
    
    // ASSERT
    Assert.Single(results.Entities);
    var contact = results.Entities[0];
    Assert.Equal("John", contact.GetAttributeValue<string>("firstname"));
    Assert.Equal("Contoso", contact.GetAttributeValue<AliasedValue>("acct.name").Value);
    Assert.Equal("Primary", contact.GetAttributeValue<AliasedValue>("primary.firstname").Value);
}
```

## Best Practices

1. **Initialize related records** - Always create parent records before children
2. **Define relationships explicitly** - Use AddRelationship for N:N scenarios
3. **Test cascade behaviors** - Verify delete, assign, share cascades
4. **Use EntityReferences correctly** - Include LogicalName and Id
5. **Query with LinkEntity** - Test complex multi-table queries
6. **Validate relationship existence** - Check associations before assuming they exist
7. **Test NULL references** - Ensure optional lookups work correctly
8. **Use early-bound entities** - Better type safety for relationships

## Common Patterns

### Bulk Associate

```csharp
public void AssociateMultipleContacts(Guid accountId, List<Guid> contactIds)
{
    var associateRequest = new AssociateRequest
    {
        Target = new EntityReference("account", accountId),
        Relationship = new Relationship("account_contact"),
        RelatedEntities = new EntityReferenceCollection(
            contactIds.Select(id => new EntityReference("contact", id)).ToList()
        )
    };
    
    _service.Execute(associateRequest);
}
```

### Retrieve with Related Entities

```csharp
public Entity RetrieveAccountWithContacts(Guid accountId)
{
    var retrieveRequest = new RetrieveRequest
    {
        Target = new EntityReference("account", accountId),
        ColumnSet = new ColumnSet(true),
        RelatedEntitiesQuery = new RelationshipQueryCollection
        {
            {
                new Relationship("contact_customer_accounts"),
                new QueryExpression("contact")
                {
                    ColumnSet = new ColumnSet("firstname", "lastname", "emailaddress1")
                }
            }
        }
    };
    
    var response = (RetrieveResponse)_service.Execute(retrieveRequest);
    return response.Entity;
}
```

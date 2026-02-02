---
name: create-entity
description: Use this skill when creating entities and database migrations in .NET applications using the Shesha framework. This skill guides you through implementing Domain-Driven Design (DDD) patterns with comprehensive knowledge of Shesha best practices, entity modeling and migrations.
---

# Shesha Entity Creator

You are an elite .NET software developer with deep expertise in enterprise application development using the Shesha framework. You specialize in creating robust, well-architected entities and domain models following Domain-Driven Design (DDD) principles and Shesha best practices.

## When to Use This Skill

Use this skill when:
- Creating new entities and domain models in a Shesha application
- Implementing entity relationships and aggregates
- Creating database migrations for new or modified entities
- Implementing reference lists (code-based and data-based)

# Design thinking
Always consider how to leverage Shesha's built-in capabilitiies before implementing domain changes. Ask yourself: 'How does Shesha handle this scenario?' and 'What framework features or resuable modules can I utilize? 

Whenever making changes to domain model entity classes ALWAYS create the corresponding database migration classes to effect the changes at database level. 

## Framework-First Approach
Always leverage Shesha's built-in capabilities before implementing custom solutions:
- Use existing entities (`Person`, `Organisation`, `Account`, `Address`, etc.)
- Utilize framework's configuration and metadata-driven approach

## Domain-Driven Design Excellence
- Follow DDD principles with proper aggregate boundaries
- Implement entities with appropriate base classes
- Use proper naming conventions and folder structures
- Apply separation of concerns consistently

## Quality and Maintainability
- Write idiomatic, well-documented code
- Follow Shesha naming conventions
- Ensure database migrations match domain model changes

## Module Manifests - Domain Entity Reference

The `reference/manifests/` folder contains JSON manifest files documenting all available Shesha modules, their domain entities, and reference lists. Use these manifests to:
- Discover existing entities to extend or reference
- Find the correct namespaces and base classes
- Identify available reference lists and their enum names
- Understand module dependencies

### Loading and Using Manifests

When creating new entities, first check the relevant manifests to:
1. **Identify reusable entities** - Extend existing entities like `Person`, `Organisation`, `Account` instead of creating new ones
2. **Find correct base classes** - Use appropriate base classes from Shesha.Core
3. **Reference existing entities** - Link to existing entities in relationships
4. **Reuse reference lists** - Use existing enums/reference lists where applicable

### Manifest JSON Structure

Each manifest file follows this structure:
```json
{
  "module": {
    "name": "Module.Name",
    "className": "Full.Qualified.ModuleClass",
    "dbPrefix": "Prefix_",
    "publisher": "Publisher",
    "dependencies": ["Dependency1", "Dependency2"]
  },
  "entities": [
    {
      "name": "EntityName",
      "namespace": "Full.Namespace",
      "description": "Entity description",
      "baseEntity": "BaseEntity",
      "isAggregate": true,
      "properties": [
        { "name": "PropertyName", "type": "PropertyType", "description": "Property description" }
      ]
    }
  ],
  "referenceLists": [
    {
      "name": "RefListName",
      "description": "Description",
      "values": ["Value1", "Value2"],
      "isEnum": true,
      "enumFullName": "Full.Qualified.EnumName"
    }
  ]
}
```

### Key Manifests for Entity Creation

| Manifest | Purpose | Key Entities |
|----------|---------|--------------|
| `shesha-core-manifest.json` | Core Shesha entities | Person, Organisation, Account, Address, Site, Notification |
| `shesha-enterprise-domain-manifest.json` | Enterprise business entities | FinancialAccount, Order, Invoice, Product, Employee, Contract |
| `shesha-crm-domain-manifest.json` | CRM domain entities | Contact, Lead, Opportunity, Campaign, Territory |
| `boxfusion-service-management-manifest.json` | Service/Case management | Case, CaseInteraction, SlaPolicy, Article |

### Using Manifests in Practice

**Example: Creating an entity that references Person**
```csharp
// From shesha-core-manifest.json, Person is in Shesha.Domain namespace
using Shesha.Domain;

public class MyEntity : FullAuditedEntity<Guid>
{
    // Reference the existing Person entity
    public virtual Person AssignedTo { get; set; }
}
```

**Example: Using an existing reference list**
```csharp
// From shesha-core-manifest.json, Gender enum is in Shesha.Domain.Enums
using Shesha.Domain.Enums;

public class MyEntity : FullAuditedEntity<Guid>
{
    public virtual RefListGender Gender { get; set; }
}
```

## Resources and Documentation

All Shesha backend development best practices and patterns are documented in:
- `backend-developer/DomainModelling.md`
- `backend-developer/DomainService.md`
- `backend-developer/StringProperties.md`
- `backend-developer/ReferenceLists.md`
- `backend-developer/DatabaseMigrations.md`

Always consult these documents for detailed implementation patterns and examples. 
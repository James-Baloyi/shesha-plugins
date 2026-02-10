# Domain Entities and Module Classes

## §1. Domain Entity

**File:** `{EntityName}.cs` in `Domain/{EntityNamePlural}/` (Domain project)

```csharp
using Abp.Domain.Entities.Auditing;
using Shesha.Domain;
using Shesha.Domain.Attributes;
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace {ModuleNamespace}.Domain.{EntityNamePlural}
{
    [Entity(TypeShortAlias = "{ModuleNamespace}.Domain.{EntityNamePlural}.{EntityName}")]
    public class {EntityName} : FullAuditedEntity<Guid>
    {
        [StringLength(200)]
        public virtual string Name { get; set; }

        public virtual DateTime? StartDate { get; set; }
        public virtual DateTime? EndDate { get; set; }
        public virtual decimal? Amount { get; set; }
        public virtual bool IsActive { get; set; }

        [ReferenceList("{RefListName}")]
        public virtual RefList{PropertyName}? Status { get; set; }

        // Navigation (single)
        public virtual Person Person { get; set; }
        public virtual StoredFile Attachment { get; set; }

        // Navigation (collection)
        [InverseProperty("PartOfId")]
        public virtual IList<{ChildEntity}> Items { get; set; }

        [ReadonlyProperty]
        public virtual decimal TotalAmount { get; set; }
    }
}
```

**Extending a framework entity (discriminator pattern):**

```csharp
[Entity(TypeShortAlias = "{ModuleNamespace}.Domain.{EntityNamePlural}.{EntityName}")]
[Discriminator]
public class {EntityName} : {BaseFrameworkEntity}
{
    [StringLength(20)]
    public virtual string CustomField { get; set; }

    public virtual OrganisationUnit Branch { get; set; }
}
```

**Key rules:**
- All properties `virtual` (NHibernate)
- `[Entity(TypeShortAlias)]` on every entity
- `[Discriminator]` when extending framework entities
- `[ReferenceList]` for lookups, `[StringLength]` on strings
- `[ReadonlyProperty]` for computed, `[InverseProperty]` for collections
- Base class: `FullAuditedEntity<Guid>`

---

## §2. Application Module

**File:** `{ModuleName}ApplicationModule.cs` at root of Application project

```csharp
using Abp.AspNetCore;
using Abp.AutoMapper;
using Abp.Modules;
using Shesha;
using Shesha.Startup;
using System.Reflection;

namespace {ModuleNamespace}.Application
{
    [DependsOn(
        typeof(SheshaCoreModule),
        typeof({DomainModuleName}),
        typeof(AbpAspNetCoreModule)
    )]
    public class {ModuleName}ApplicationModule : SheshaSubModule<{DomainModuleName}>
    {
        public override void Initialize()
        {
            IocManager.RegisterAssemblyByConvention(Assembly.GetExecutingAssembly());

            var thisAssembly = Assembly.GetExecutingAssembly();
            Configuration.Modules.AbpAutoMapper().Configurators.Add(
                cfg => cfg.AddMaps(thisAssembly)
            );
        }

        public override void PreInitialize()
        {
            base.PreInitialize();

            Configuration.Modules.AbpAspNetCore().CreateControllersForAppServices(
                typeof({ModuleName}ApplicationModule).Assembly,
                moduleName: "{ModulePrefix}",
                useConventionalHttpVerbs: true);
        }
    }
}
```

**Key rules:**
- Inherits `SheshaSubModule<TDomainModule>`
- `Initialize()`: register assembly + AutoMapper scanning
- `PreInitialize()`: auto-generate API controllers
- `[DependsOn]` for module dependencies

---

## §3. Domain Module

**File:** `{ModuleName}Module.cs` at root of Domain project

```csharp
using Abp.Modules;
using Shesha;
using Shesha.Modules;
using System.Reflection;

namespace {ModuleNamespace}.Domain
{
    [DependsOn(
        typeof(SheshaCoreModule),
        typeof(SheshaApplicationModule)
    )]
    public class {ModuleName}Module : SheshaModule
    {
        public static string Name = "{ModuleNamespace}";

        public override SheshaModuleInfo ModuleInfo => new SheshaModuleInfo(Name)
        {
            FriendlyName = "{FriendlyModuleName}",
            Publisher = "{PublisherName}",
        };

        public override void Initialize()
        {
            var thisAssembly = Assembly.GetExecutingAssembly();
            IocManager.RegisterAssemblyByConvention(thisAssembly);
        }
    }
}
```

# Distribution (Export/Import)

Three classes enable configuration item portability across environments.

## §1 Distribution DTO

Serializable representation of the configuration item. Only includes serializable primitives — no entity references.

```csharp
using Shesha.ConfigurationItems.Distribution;

namespace {Namespace}.Application.{ConfigName}s.Distribution.Dto
{
    public class Distributed{ConfigName} : DistributedConfigurableItemBase
    {
        // Mirror each custom property from the entity.
        // Use primitive types only (no entity references).
        // FK references should be serialized as Guid? if needed.

        // public int {IntProp} { get; set; }
        // public bool {BoolProp} { get; set; }
        // public string {StringProp} { get; set; }
    }
}
```

The base `DistributedConfigurableItemBase` already includes: `Id`, `OriginId`, `Name`, `Label`, `ItemType`, `Description`, `ModuleName`, `FrontEndApplication`, `VersionNo`, `VersionStatus`, `ParentVersionId`, `Suppress`, `BaseItem`.

## §2 Exporter

Converts from entity to distribution DTO and serializes to JSON.

### Interface

```csharp
using Shesha.ConfigurationItems.Distribution;
using {EntityNamespace};

namespace {Namespace}.Application.{ConfigName}s.Distribution
{
    public interface I{ConfigName}Export : IConfigurableItemExport<{ConfigName}>
    {
    }
}
```

### Implementation

```csharp
using Abp.Dependency;
using Abp.Domain.Repositories;
using Newtonsoft.Json;
using Shesha.ConfigurationItems.Distribution;
using Shesha.Domain;
using {EntityNamespace};
using {Namespace}.Application.{ConfigName}s.Distribution.Dto;
using System;
using System.IO;
using System.Threading.Tasks;

namespace {Namespace}.Application.{ConfigName}s.Distribution
{
    public class {ConfigName}Export : I{ConfigName}Export, ITransientDependency
    {
        private readonly IRepository<{ConfigName}, Guid> _repository;

        public {ConfigName}Export(IRepository<{ConfigName}, Guid> repository)
        {
            _repository = repository;
        }

        public string ItemType => {ConfigName}.ItemTypeName;

        public async Task<DistributedConfigurableItemBase> ExportItemAsync(Guid id)
        {
            var item = await _repository.GetAsync(id);
            return await ExportItemAsync(item);
        }

        public Task<DistributedConfigurableItemBase> ExportItemAsync(
            ConfigurationItemBase item)
        {
            if (item is not {ConfigName} config)
                throw new ArgumentException(
                    $"Expected {nameof({ConfigName})}, got {item.GetType().FullName}");

            var result = new Distributed{ConfigName}
            {
                // Base properties (always include all of these)
                Id = config.Id,
                Name = config.Name,
                ModuleName = config.Module?.Name,
                FrontEndApplication = config.Application?.AppKey,
                ItemType = config.ItemType,
                Label = config.Label,
                Description = config.Description,
                OriginId = config.Origin?.Id,
                BaseItem = config.BaseItem?.Id,
                VersionNo = config.VersionNo,
                VersionStatus = config.VersionStatus,
                ParentVersionId = config.ParentVersion?.Id,
                Suppress = config.Suppress,

                // Custom properties
                // {CustomProp} = config.{CustomProp},
            };

            return Task.FromResult<DistributedConfigurableItemBase>(result);
        }

        public async Task WriteToJsonAsync(
            DistributedConfigurableItemBase item, Stream jsonStream)
        {
            var json = JsonConvert.SerializeObject(item, Formatting.Indented);
            using var writer = new StreamWriter(jsonStream);
            await writer.WriteAsync(json);
        }
    }
}
```

## §3 Importer

Reads JSON and creates or updates entities in the database.

### Interface

```csharp
using Shesha.ConfigurationItems.Distribution;
using {EntityNamespace};

namespace {Namespace}.Application.{ConfigName}s.Distribution
{
    public interface I{ConfigName}Import : IConfigurableItemImport<{ConfigName}>
    {
    }
}
```

### Implementation

```csharp
using Abp.Dependency;
using Abp.Domain.Repositories;
using Newtonsoft.Json;
using Shesha.ConfigurationItems.Distribution;
using Shesha.Domain;
using Shesha.Domain.ConfigurationItems;
using Shesha.Services.ConfigurationItems;
using {EntityNamespace};
using {Namespace}.Application.{ConfigName}s.Distribution.Dto;
using System;
using System.IO;
using System.Threading.Tasks;

namespace {Namespace}.Application.{ConfigName}s.Distribution
{
    public class {ConfigName}Import : ConfigurationItemImportBase,
        I{ConfigName}Import, ITransientDependency
    {
        private readonly IRepository<{ConfigName}, Guid> _repository;

        public {ConfigName}Import(
            IRepository<Module, Guid> moduleRepo,
            IRepository<FrontEndApp, Guid> frontEndAppRepo,
            IRepository<{ConfigName}, Guid> repository
        ) : base(moduleRepo, frontEndAppRepo)
        {
            _repository = repository;
        }

        public string ItemType => {ConfigName}.ItemTypeName;

        public async Task<ConfigurationItemBase> ImportItemAsync(
            DistributedConfigurableItemBase item,
            IConfigurationItemsImportContext context)
        {
            if (item is not Distributed{ConfigName} distributed)
                throw new NotSupportedException(
                    $"Expected {nameof(Distributed{ConfigName})}, " +
                    $"got {item.GetType().FullName}");

            var statusToImport = context.ImportStatusAs ?? item.VersionStatus;

            // Match existing item by Name + Module + IsLast
            var dbItem = await _repository.FirstOrDefaultAsync(x =>
                x.Name == item.Name
                && (x.Module == null && item.ModuleName == null
                    || x.Module != null && x.Module.Name == item.ModuleName)
                && x.IsLast);

            if (dbItem != null)
            {
                await MapPropertiesAsync(distributed, dbItem, context);
                await _repository.UpdateAsync(dbItem);
            }
            else
            {
                dbItem = new {ConfigName}();
                await MapPropertiesAsync(distributed, dbItem, context);

                dbItem.VersionNo = 1;
                dbItem.Module = await GetModuleAsync(item.ModuleName, context);
                dbItem.VersionStatus = statusToImport;
                dbItem.CreatedByImport = context.ImportResult;

                dbItem.Normalize();
                await _repository.InsertAsync(dbItem);
            }

            return dbItem;
        }

        private async Task MapPropertiesAsync(
            Distributed{ConfigName} source,
            {ConfigName} target,
            IConfigurationItemsImportContext context)
        {
            // Base properties
            target.Name = source.Name;
            target.Module = await GetModuleAsync(source.ModuleName, context);
            target.Application = await GetFrontEndAppAsync(
                source.FrontEndApplication, context);
            target.Label = source.Label;
            target.Description = source.Description;
            target.VersionNo = source.VersionNo;
            target.VersionStatus = source.VersionStatus;
            target.Suppress = source.Suppress;

            // Custom properties
            // target.{CustomProp} = source.{CustomProp};
        }

        public async Task<DistributedConfigurableItemBase> ReadFromJsonAsync(
            Stream jsonStream)
        {
            using var reader = new StreamReader(jsonStream);
            var json = await reader.ReadToEndAsync();

            var result = JsonConvert.DeserializeObject<Distributed{ConfigName}>(json)
                ?? throw new Exception(
                    $"Failed to deserialize {nameof({ConfigName})} from JSON");

            return result;
        }
    }
}
```

## Key Points

- **`ITransientDependency`** — both exporter and importer must implement this.
- **Match by Name + Module + IsLast** — this is how the importer finds existing items.
- **`Normalize()`** — call on new items only; sets Origin to self-reference.
- **`context.ImportStatusAs`** — allows the import caller to override the version status.
- **`GetModuleAsync` / `GetFrontEndAppAsync`** — inherited from `ConfigurationItemImportBase`; resolves or creates modules/apps as needed.

## Exported Package Structure

Items are packaged into `.shaconfig` zip files with this folder structure:

```
{module-name}/
  {item-type-name}/            ← matches ItemTypeName
    {item-name}.json           ← one JSON file per item
```

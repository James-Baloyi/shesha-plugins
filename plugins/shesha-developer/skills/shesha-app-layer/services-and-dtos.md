# Services, DTOs, and Mapping Profiles

## §1. Application Service

**File:** `{EntityName}AppService.cs` in `Services/{EntityNamePlural}/`

```csharp
using Abp.Domain.Repositories;
using Abp.Runtime.Validation;
using Microsoft.AspNetCore.Mvc;
using Shesha;
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.Linq;
using System.Threading.Tasks;

namespace {ModuleNamespace}.Application.Services.{EntityNamePlural}
{
    public class {EntityName}AppService : SheshaAppServiceBase
    {
        private readonly IRepository<{EntityName}, Guid> _{entityName}Repository;

        public {EntityName}AppService(
            IRepository<{EntityName}, Guid> {entityName}Repository)
        {
            _{entityName}Repository = {entityName}Repository;
        }

        [HttpPost]
        public async Task CreateAsync({EntityName}Dto input)
        {
            var validationResults = new List<ValidationResult>();

            if (input == null)
                throw new UserFriendlyException("Input cannot be null.");

            // Add field-level validations:
            // if (input.RequiredField == null)
            //     validationResults.Add(new ValidationResult("Field is required", new[] { nameof(input.RequiredField) }));

            if (validationResults.Any())
                throw new AbpValidationException("Please correct the errors and try again", validationResults);

            var entity = new {EntityName}
            {
                // Map properties from DTO
            };

            await _{entityName}Repository.InsertAsync(entity);
        }

        [HttpPut]
        public async Task UpdateAsync({EntityName}Dto input)
        {
            if (input.Id == null)
                throw new UserFriendlyException("Id is required.");

            var entity = await _{entityName}Repository.GetAsync((Guid)input.Id);
            // Update entity properties from DTO
            await _{entityName}Repository.UpdateAsync(entity);
        }

        [HttpGet]
        public async Task<{EntityName}Dto> GetAsync(Guid id)
        {
            var entity = await _{entityName}Repository.GetAsync(id);
            return new {EntityName}Dto
            {
                Id = entity.Id,
                // Map properties
            };
        }
    }
}
```

**Key rules:**
- Inherits `SheshaAppServiceBase` (provides `Logger`, `GetCurrentPersonAsync()`, `AbpSession`, `MapJObjectToEntityAsync`)
- Constructor injection for repositories and domain managers
- `[HttpPost]` for create, `[HttpPut]` for update, `[HttpGet]` for read
- Validation: `List<ValidationResult>` + `AbpValidationException` (NOT FluentValidation)
- `UserFriendlyException` for display-safe business errors

**DynamicDto pattern** (partial JSON from frontend):

```csharp
[HttpPut]
public async Task SubmitAsync(DynamicDto<{EntityName}, Guid> request)
{
    var validationResults = new List<ValidationResult>();
    var entity = await _{entityName}Repository.GetAsync(request.Id);

    var result = await MapJObjectToEntityAsync<{EntityName}, Guid>(
        request._jObject, entity, validationResults);

    if (!result)
        throw new AbpValidationException("Please correct the errors and try again", validationResults);

    await _{entityName}Repository.UpdateAsync(entity);
}
```

---

## §2. DTO

**File:** `{EntityName}Dto.cs` in `Services/{EntityNamePlural}/`

```csharp
using System;

namespace {ModuleNamespace}.Application.Services.{EntityNamePlural}
{
    public class {EntityName}Dto
    {
        public Guid? Id { get; set; }

        // Simple value properties
        public string Name { get; set; }
        public DateTime? StartDate { get; set; }
        public decimal? Amount { get; set; }
        public bool IsActive { get; set; }

        // Reference entity IDs (NOT navigation properties)
        public Guid? PersonId { get; set; }
        public Guid? CategoryId { get; set; }

        // RefList values as nullable long
        public long? Status { get; set; }
    }
}
```

**Key rules:**
- `Guid?` for Id and FK references
- `long?` for RefList/enum properties
- Nullable types for optional fields
- No navigation properties — use `Guid?` references

---

## §3. AutoMapper Profile

**File:** `{EntityName}MappingProfile.cs` in `Services/{EntityNamePlural}/`

```csharp
using AutoMapper;
using Shesha.AutoMapper;

namespace {ModuleNamespace}.Application.Services.{EntityNamePlural}
{
    public class {EntityName}MappingProfile : ShaProfile
    {
        public {EntityName}MappingProfile()
        {
            CreateMap<{EntityName}Dto, {EntityName}>()
                .ForMember(dest => dest.NavigationProperty, opt => opt.Ignore());
                // Ignore all navigation/complex properties — resolved by ID
        }
    }
}
```

**Key rules:**
- Inherit `ShaProfile`
- `.ForMember(dest => dest.X, opt => opt.Ignore())` for navigation properties
- Module's `Initialize()` auto-scans profiles via `cfg.AddMaps(thisAssembly)`

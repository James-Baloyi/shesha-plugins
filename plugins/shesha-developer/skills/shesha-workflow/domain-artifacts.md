# Domain Layer Workflow Artifacts

## §1. Workflow Instance

**File:** `{WorkflowName}Workflow.cs` in `Domain/{WorkflowName}Workflows/`

```csharp
using Shesha.Domain.Attributes;
using Shesha.Workflow.Domain;
using System.ComponentModel.DataAnnotations;

namespace {ModuleNamespace}.Domain.{WorkflowName}Workflows
{
    [JoinedProperty("{TablePrefix}_{WorkflowName}Workflows")]
    [Prefix(UsePrefixes = false)]
    [Entity(TypeShortAlias = "{ModuleNamespace}.Domain.{WorkflowName}Workflow")]
    public class {WorkflowName}Workflow : WorkflowInstanceWithTypedDefinition<{WorkflowName}WorkflowDefinition>
    {
        public virtual {ModelEntity} Model { get; set; }
    }
}
```

**With extra state** (e.g. cancellation context):

```csharp
[JoinedProperty("{TablePrefix}_{WorkflowName}Workflows")]
[Prefix(UsePrefixes = false)]
[Entity(TypeShortAlias = "{ModuleNamespace}.Domain.{WorkflowName}Workflow")]
public class {WorkflowName}Workflow : WorkflowInstanceWithTypedDefinition<{WorkflowName}WorkflowDefinition>
{
    public virtual {ModelEntity} Model { get; set; }
    public virtual bool ApplicationWasRecommended { get; set; }
    public virtual bool ApplicationWasApproved { get; set; }

    [StringLength(4000)]
    public virtual string Comments { get; set; }
}
```

**Key rules:**
- `Model` links to the domain entity being processed
- `[JoinedProperty]` maps to a dedicated DB table (joined subclass)
- `[Prefix(UsePrefixes = false)]` disables column prefix generation
- All properties must be `virtual`

---

## §2. Workflow Definition

**File:** `{WorkflowName}WorkflowDefinition.cs` in `Domain/{WorkflowName}Workflows/`

Always generated alongside a Workflow Instance — they are a mandatory pair.

```csharp
using Shesha.Domain.Attributes;
using Shesha.Workflow.Domain;
using System.ComponentModel.DataAnnotations;

namespace {ModuleNamespace}.Domain.{WorkflowName}Workflows
{
    [DiscriminatorValue("{discriminator-slug}")]
    [JoinedProperty("{TablePrefix}_{WorkflowName}WorkflowDefinitions")]
    [Display(Name = "{Friendly Name} workflow definition")]
    public class {WorkflowName}WorkflowDefinition : WorkflowDefinition
    {
        public override WorkflowInstance CreateInstance()
        {
            return new {WorkflowName}Workflow
            {
                WorkflowDefinition = this,
                RefNumber = AssignReferenceNumber(),
                SubStatus = (long){RefListStatusEnum}.Draft,
                Model = new {ModelEntity}()
            };
        }
    }
}
```

**With processor-based initialization** (complex model setup):

```csharp
public override WorkflowInstance CreateInstance()
{
    var processor = IocManager.Instance.Resolve<{ModelEntity}Processor>();
    var model = AsyncHelper.RunSync(() => processor.InitialiseAsync<{ModelEntity}>());

    return new {WorkflowName}Workflow
    {
        WorkflowDefinition = this,
        RefNumber = AssignReferenceNumber(),
        SubStatus = (long){RefListStatusEnum}.Draft,
        Model = model
    };
}
```

**`CreateInstance()` must:**
1. Create the matching Workflow Instance type
2. Set `WorkflowDefinition = this`
3. Assign reference number via `AssignReferenceNumber()`
4. Set `SubStatus` to initial status (typically Draft)
5. Create a new `Model` entity

---

## §3. Workflow Manager

**File:** `{WorkflowName}WorkflowManager.cs` in `Domain/{WorkflowName}Workflows/`

Generate when user needs workflow orchestration, status management, or cross-workflow coordination.

```csharp
using Abp.Domain.Repositories;
using Abp.Domain.Services;
using Abp.Domain.Uow;
using Abp.Runtime.Session;
using Abp.UI;
using Shesha.NHibernate;
using Shesha.Workflow.Domain;
using Shesha.Workflow.Domain.Enums;
using Shesha.Workflow.DomainServices;
using System;
using System.Linq;
using System.Threading.Tasks;

namespace {ModuleNamespace}.Domain.{WorkflowName}Workflows
{
    public class {WorkflowName}WorkflowManager : DomainService
    {
        private readonly IRepository<{WorkflowName}Workflow, Guid> _workflowRepository;
        private readonly IRepository<{ModelEntity}, Guid> _modelRepository;
        private readonly IAbpSession _abpSession;
        private readonly ISessionProvider _sessionProvider;
        private readonly IProcessDomainService _processDomainService;

        public {WorkflowName}WorkflowManager(
            IRepository<{WorkflowName}Workflow, Guid> workflowRepository,
            IRepository<{ModelEntity}, Guid> modelRepository,
            IAbpSession abpSession,
            ISessionProvider sessionProvider,
            IProcessDomainService processDomainService)
        {
            _workflowRepository = workflowRepository;
            _modelRepository = modelRepository;
            _abpSession = abpSession;
            _sessionProvider = sessionProvider;
            _processDomainService = processDomainService;
        }

        public async Task<string> CreateSubjectAsync(Guid workflowId)
        {
            var workflow = await _workflowRepository.GetAsync(workflowId);
            var model = workflow.Model
                ?? throw new ArgumentNullException(nameof(workflow.Model));
            return $"{model.Name} — {model.Person?.FullName}";
        }

        public async Task UpdateStatusesAsync(
            Guid workflowId,
            {RefListStatusEnum} subStatus,
            {RefListStatusEnum} modelStatus)
        {
            try
            {
                var workflow = await _workflowRepository.GetAsync(workflowId);
                workflow.SubStatus = (long?)subStatus;
                workflow.Model.Status = modelStatus;
                await _workflowRepository.UpdateAsync(workflow);
            }
            catch (Exception ex)
            {
                Logger.Error($"Failed to update statuses for workflow {workflowId}", ex);
                throw;
            }
        }

        public void RefreshWorkflow({WorkflowName}Workflow workflow)
        {
            _sessionProvider.Session.Refresh(workflow);
        }
    }
}
```

**For generic/reusable managers** (extending a framework base):

```csharp
public class {WorkflowName}WorkflowManager
    : FrameworkWorkflowManager<{WorkflowName}WorkflowDefinition, {WorkflowName}Workflow, {ModelEntity}>
{
    public {WorkflowName}WorkflowManager(/* base params */) : base(/* pass through */) { }
}
```

**Manager responsibilities:** subject generation, status sync between `SubStatus` and model `Status`, NHibernate session refresh, cross-workflow coordination, completing user tasks via `IProcessDomainService`.

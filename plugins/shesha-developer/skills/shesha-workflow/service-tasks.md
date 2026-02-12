# Application Layer Workflow Artifacts

## §1. Service Task

**File:** `{TaskName}ServiceTask.cs` in `Workflows/{WorkflowNamePlural}/`

A single automated step in a workflow. The engine instantiates and executes it when reaching the matching node.

```csharp
using Abp.Dependency;
using Abp.Domain.Repositories;
using Shesha.Workflow.Tasks;
using System;
using System.ComponentModel.DataAnnotations;
using System.Threading.Tasks;

namespace {ModuleNamespace}.Application.Workflows.{WorkflowNamePlural}
{
    [Display(Name = "{Friendly Task Name}",
             Description = "{Brief description}")]
    public class {TaskName}ServiceTask : AsyncServiceTask<{WorkflowName}Workflow>, ITransientDependency
    {
        private readonly IRepository<{WorkflowName}Workflow, Guid> _workflowRepository;

        public {TaskName}ServiceTask(
            IRepository<{WorkflowName}Workflow, Guid> workflowRepository)
        {
            _workflowRepository = workflowRepository;
        }

        public override async Task RunAsync(ServiceTaskExecutionContext<{WorkflowName}Workflow> context)
        {
            var workflow = context.WorkflowInstance;

            // Task logic here

            await _workflowRepository.UpdateAsync(workflow);
        }
    }
}
```

### Common task patterns

**(a) Update Model Status:**
```csharp
public override async Task RunAsync(ServiceTaskExecutionContext<{WorkflowName}Workflow> context)
{
    var workflow = context.WorkflowInstance;
    RefreshWorkflow(workflow);
    workflow.Model.Status = ({RefListStatusEnum}?)workflow.SubStatus;
    await _workflowRepository.UpdateAsync(workflow);
}
```

**(b) Update Subject:**
```csharp
public override async Task RunAsync(ServiceTaskExecutionContext<{WorkflowName}Workflow> context)
{
    var workflow = context.WorkflowInstance;
    RefreshWorkflow(workflow);
    if (workflow.Subject == null)
        workflow.Subject = await _workflowManager.CreateSubjectAsync(workflow.Id);
    await _workflowRepository.UpdateAsync(workflow);
}
```

**(c) Business Logic Delegation:**
```csharp
public override async Task RunAsync(ServiceTaskExecutionContext<{WorkflowName}Workflow> context)
{
    var model = context.WorkflowInstance.Model
        ?? throw new InvalidOperationException("Workflow model is null");
    await _businessManager.ProcessAsync(model.Id);
}
```

**(d) Cross-Workflow Action:**
```csharp
public override async Task RunAsync(ServiceTaskExecutionContext<{CancellationWorkflow}> context)
{
    var cancellationWorkflow = context.WorkflowInstance;
    var originalWorkflow = await _originalWorkflowRepository.GetAll()
        .Where(w => w.Model.Id == cancellationWorkflow.Model.Parent.Id
                  && w.Model.RequestType == RefListRequestType.Application)
        .FirstOrDefaultAsync()
        ?? throw new ArgumentNullException("Original workflow not found");

    await _manager.CancelAsync(originalWorkflow.Id);
    await _workflowManager.UpdateStatusesAsync(
        originalWorkflow.Id, RefListStatus.Cancelled, RefListStatus.Cancelled);
}
```

**(e) Conditional Logic:**
```csharp
public override async Task RunAsync(ServiceTaskExecutionContext<{CancellationWorkflow}> context)
{
    var workflow = context.WorkflowInstance;
    var originalWorkflow = await FindOriginalWorkflowAsync(workflow);

    if (originalWorkflow.Status == RefListWorkflowStatus.Completed)
    {
        originalWorkflow.Status = RefListWorkflowStatus.Cancelled;
        await _workflowRepository.UpdateAsync(originalWorkflow);
        await _unitOfWorkManager.Current.SaveChangesAsync();
    }
    else if (originalWorkflow.Status == RefListWorkflowStatus.InProgress)
    {
        await _manager.SuspendAsync(originalWorkflow.Id);
    }
}
```

---

## §2. Generic Base Service Task

**File:** `{TaskName}ServiceTaskBase.cs` in `Workflows/Common/`

Abstract base for tasks sharing logic across multiple workflow types (Template Method pattern).

```csharp
using Abp.Dependency;
using Abp.Domain.Repositories;
using Shesha.Workflow.Domain;
using Shesha.Workflow.Tasks;
using System;
using System.Threading.Tasks;

namespace {ModuleNamespace}.Application.Workflows.Common
{
    public abstract class {TaskName}ServiceTaskBase<TWorkflow> : AsyncServiceTask<TWorkflow>, ITransientDependency
        where TWorkflow : WorkflowInstance
    {
        private readonly IRepository<TWorkflow, Guid> _workflowRepository;
        private readonly IRepository<{ModelEntity}, Guid> _modelRepository;

        protected {TaskName}ServiceTaskBase(
            IRepository<TWorkflow, Guid> workflowRepository,
            IRepository<{ModelEntity}, Guid> modelRepository)
        {
            _workflowRepository = workflowRepository;
            _modelRepository = modelRepository;
        }

        protected abstract {ModelEntity} GetModel(TWorkflow workflow);

        public override async Task RunAsync(ServiceTaskExecutionContext<TWorkflow> context)
        {
            var model = GetModel(context.WorkflowInstance)
                ?? throw new InvalidOperationException(
                    $"Model is null on workflow {context.WorkflowInstance.Id}");

            // Shared business logic here

            await Task.CompletedTask;
        }
    }
}
```

**Concrete implementation** per workflow type:

**File:** `{TaskName}ServiceTask.cs` in `Workflows/{WorkflowNamePlural}/`

```csharp
using Abp.Domain.Repositories;
using {ModuleNamespace}.Application.Workflows.Common;
using System;

namespace {ModuleNamespace}.Application.Workflows.{WorkflowNamePlural}
{
    public class {TaskName}ServiceTask : {TaskName}ServiceTaskBase<{WorkflowName}Workflow>
    {
        public {TaskName}ServiceTask(
            IRepository<{WorkflowName}Workflow, Guid> workflowRepository,
            IRepository<{ModelEntity}, Guid> modelRepository)
            : base(workflowRepository, modelRepository)
        {
        }

        protected override {ModelEntity} GetModel({WorkflowName}Workflow workflow)
            => workflow.Model;
    }
}
```

---

## §3. Workflow Extension Methods

**File:** `{WorkflowName}WorkflowExtensions.cs` in `Workflows/{WorkflowNamePlural}/`

Static extension methods on `WorkflowInstance` used as gateway conditions in the workflow designer.

```csharp
using Abp.Dependency;
using Abp.Domain.Repositories;
using Shesha.Workflow.Domain;
using System;
using System.Linq;

namespace {ModuleNamespace}.Application.Workflows.{WorkflowNamePlural}
{
    public static class {WorkflowName}WorkflowExtensions
    {
        public static bool {ConditionName}(this WorkflowInstance workflow)
        {
            if (workflow == null)
                throw new ArgumentNullException(nameof(workflow));

            var workflowRepo = IocManager.Instance
                .Resolve<IRepository<{WorkflowName}Workflow, Guid>>();

            var typedWorkflow = workflowRepo.FirstOrDefault(workflow.Id);
            if (typedWorkflow?.Model == null)
                return false;

            var relatedRepo = IocManager.Instance
                .Resolve<IRepository<{RelatedEntity}, Guid>>();

            var items = relatedRepo.GetAll()
                .Where(x => x.PartOf != null && x.PartOf.Id == typedWorkflow.Model.Id)
                .ToList();

            return items.Any(x => /* condition logic */);
        }
    }
}
```

**Checking definition configuration:**

```csharp
public static bool {ConfigFlag}(this WorkflowInstance workflow)
{
    if (workflow == null)
        throw new ArgumentNullException(nameof(workflow));

    var workflowRepo = IocManager.Instance
        .Resolve<IRepository<{WorkflowName}Workflow, Guid>>();
    var typedWorkflow = workflowRepo.FirstOrDefault(workflow.Id)
        ?? throw new InvalidOperationException($"Workflow {workflow.Id} not found");

    var definitionRepo = IocManager.Instance
        .Resolve<IRepository<{WorkflowName}WorkflowDefinition, Guid>>();
    var definition = definitionRepo.FirstOrDefault(typedWorkflow.WorkflowDefinition.Id);

    return definition?.{ConfigFlag} ?? false;
}
```

**Key rules:**
- Extends `WorkflowInstance` (base type, not specific type) — required by the workflow engine
- Uses `IocManager.Instance.Resolve<T>()` since extension methods cannot use constructor DI
- Returns `bool` for gateway conditions
- Must be defensive: null-check workflow/model, return false on missing data
- Name as a question: `HasDisagreementInRatings`, `OriginalRequestWasSynchronized`

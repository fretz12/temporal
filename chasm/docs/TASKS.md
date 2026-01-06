# How Tasks Work in CHASM

## Overview

Tasks in CHASM are the mechanism for scheduling and executing work, either immediately within the current transaction or at a later time. They are the primary way components drive asynchronous behavior.

---

## Two Types of Tasks

### 1. **Pure Tasks** (Synchronous within Transaction)

Pure tasks execute **inside the current transaction** during `CloseTransaction()`. They:
- Run synchronously before the transaction is persisted
- Can modify component state directly
- Cannot perform side effects (no network calls, no I/O)
- Are ideal for timers, state machine transitions, and internal logic

```go
// Executor signature - receives MutableContext (can modify state)
type PureTaskExecutor[C any, T any] interface {
    Execute(MutableContext, C, TaskAttributes, T) error
}
```

**Example: BackoffTask** (from callback library)
```go
func (e *BackoffTaskExecutor) Execute(
    ctx chasm.MutableContext,
    callback *Callback,
    taskAttrs chasm.TaskAttributes,
    task *callbackspb.BackoffTask,
) error {
    // Can directly modify state and transition the state machine
    return TransitionRescheduled.Apply(callback, ctx, EventRescheduled{})
}
```

### 2. **Side-Effect Tasks** (Asynchronous, External)

Side-effect tasks execute **outside the transaction** in a separate task processor. They:
- Run asynchronously after the transaction is committed
- Can perform I/O (HTTP calls, gRPC, database access)
- Receive a regular `context.Context`, NOT a `MutableContext`
- Must call `UpdateComponent()` to persist any state changes

```go
// Executor signature - receives regular context (read-only, must explicitly persist)
type SideEffectTaskExecutor[C any, T any] interface {
    Execute(context.Context, ComponentRef, TaskAttributes, T) error
}
```

**Example: InvocationTask** (from callback library)
```go
func (e InvocationTaskExecutor) Execute(
    ctx context.Context, 
    ref chasm.ComponentRef, 
    attrs chasm.TaskAttributes, 
    task *callbackspb.InvocationTask,
) error {
    // 1. Read the component (read-only)
    invokable, _ := chasm.ReadComponent(ctx, ref, (*Callback).loadInvocationArgs, nil)
    
    // 2. Make HTTP call (side effect!)
    result := invokable.Invoke(ctx, ...)
    
    // 3. Persist result (opens new transaction)
    _, _, err := chasm.UpdateComponent(ctx, ref, (*Callback).saveResult, result)
    return err
}
```

---

## Task Attributes

Tasks are created with `TaskAttributes` that control their behavior:

```go
type TaskAttributes struct {
    ScheduledTime time.Time  // When to execute (zero = immediate)
    Destination   string     // Target for outbound tasks (e.g., "https://api.example.com")
}
```

### Key Rules:
- **Immediate tasks**: `ScheduledTime.IsZero()` → execute now (transfer category)
- **Timer tasks**: `ScheduledTime` set → execute at that time (timer category)
- **Outbound tasks**: `Destination != ""` → routed to specific endpoint (outbound category)
- **Visibility tasks**: Special internal tasks for search attribute updates

---

## Task Categories

The framework automatically assigns tasks to categories based on attributes:

```go
func taskCategory(task) tasks.Category {
    if task.TypeId == visibilityTaskTypeID {
        return tasks.CategoryVisibility  // Search attribute updates
    }
    if task.Destination != "" {
        return tasks.CategoryOutbound    // External HTTP/gRPC calls
    }
    if task.ScheduledTime.IsZero() {
        return tasks.CategoryTransfer    // Immediate execution
    }
    return tasks.CategoryTimer           // Delayed execution
}
```

---

## Task Lifecycle

### 1. **Creation** - Adding a Task

Tasks are added via `MutableContext.AddTask()` during a transaction:

```go
func (cb *Callback) OnTrigger(ctx chasm.MutableContext, event EventScheduled) error {
    // Add a side-effect task to make an HTTP call
    ctx.AddTask(cb, chasm.TaskAttributes{
        Destination: "https://api.example.com",
    }, &InvocationTask{Attempt: 1})
    return nil
}
```

Internally, `AddTask` stores the task in one of two places:
- **Immediate pure tasks**: `node.immediatePureTasks` (executed during CloseTransaction)
- **All other tasks**: `node.newTasks` (serialized and persisted)

```go
func (n *Node) AddTask(component Component, attrs TaskAttributes, task any) {
    rt := n.registry.taskFor(task)
    
    if rt.isPureTask && attrs.IsImmediate() {
        // Execute immediately during CloseTransaction
        n.immediatePureTasks[component] = append(...)
        return
    }
    
    // Persist and execute later
    n.newTasks[component] = append(...)
}
```

### 2. **Validation** - Checking Task Validity

Before execution, tasks are validated via `TaskValidator.Validate()`:

```go
type TaskValidator[C any, T any] interface {
    Validate(Context, C, TaskAttributes, T) (bool, error)
}
```

**Purpose of validation:**
1. **Deduplication in standby clusters**: Replicated tasks may already be completed
2. **Obsolescence checking**: State changes may invalidate pending tasks
3. **Gate execution**: Prevent unnecessary work

**Example: Validate InvocationTask**
```go
func (e InvocationTaskExecutor) Validate(
    ctx chasm.Context, 
    cb *Callback, 
    attrs chasm.TaskAttributes, 
    task *InvocationTask,
) (bool, error) {
    // Only valid if attempt matches AND callback is still in SCHEDULED state
    return cb.Attempt == task.Attempt && 
           cb.Status == CALLBACK_STATUS_SCHEDULED, nil
}
```

### 3. **Execution** - Running the Task

#### Pure Task Execution (during CloseTransaction)

```go
func (n *Node) CloseTransaction() (NodesMutation, error) {
    // Execute immediate pure tasks FIRST
    if err := n.executeImmediatePureTasks(); err != nil {
        return NodesMutation{}, err
    }
    
    // Then serialize, validate, and generate physical tasks
    // ...
}

func (n *Node) executeImmediatePureTasks() error {
    for component, tasks := range n.immediatePureTasks {
        taskNode := findNodeForComponent(component)
        for _, task := range tasks {
            executed, err := taskNode.ExecutePureTask(ctx, task.attrs, task.task)
            // ...
        }
    }
}
```

The `ExecutePureTask` method:
1. Validates the task
2. If valid, calls the registered executor with `MutableContext`
3. Changes are part of the current transaction

```go
func (n *Node) ExecutePureTask(ctx, attrs, task) (bool, error) {
    // 1. Validate
    valid, err := n.validateTask(validateCtx, attrs, task)
    if !valid {
        return false, nil  // Skip silently
    }
    
    // 2. Execute with MutableContext
    executionCtx := NewMutableContext(ctx, n)
    component, _ := n.Component(executionCtx, ComponentRef{})
    
    // Call the registered executor
    registrableTask.executeFn.Call([]reflect.Value{
        reflect.ValueOf(executionCtx),
        reflect.ValueOf(component),
        reflect.ValueOf(attrs),
        reflect.ValueOf(task),
    })
    
    return true, nil
}
```

#### Side-Effect Task Execution (asynchronous)

Side-effect tasks are persisted as "physical tasks" and executed by task processors:

```go
func (n *Node) closeTransactionGeneratePhysicalSideEffectTask(...) {
    n.backend.AddTasks(&tasks.ChasmTask{
        WorkflowKey:         n.backend.GetWorkflowKey(),
        VisibilityTimestamp: sideEffectTask.ScheduledTime,
        Destination:         sideEffectTask.Destination,
        Category:            taskCategory(sideEffectTask),
        Info: &ChasmTaskInfo{
            Path:    nodePath,        // Path to component in tree
            TypeId:  sideEffectTask.TypeId,  // Task type for deserialization
            Data:    sideEffectTask.Data,    // Serialized task payload
        },
    })
}
```

The task processor later:
1. Loads the execution
2. Deserializes the task
3. Calls the side-effect executor with `context.Context` and `ComponentRef`

---

## Task Registration

Tasks must be registered with a Library:

```go
func (l *Library) Tasks() []*chasm.RegistrableTask {
    return []*chasm.RegistrableTask{
        // Side-effect task: external HTTP invocation
        chasm.NewRegistrableSideEffectTask(
            "invoke",                      // Task type name
            l.InvocationTaskExecutor,      // Validator (same struct implements both)
            l.InvocationTaskExecutor,      // Executor
        ),
        
        // Pure task: internal state transition
        chasm.NewRegistrablePureTask(
            "backoff",
            l.BackoffTaskExecutor,
            l.BackoffTaskExecutor,
        ),
    }
}
```

Each registrable task has:
- **Task type**: Unique name within the library
- **Go type**: The struct type for the task payload
- **Component Go type**: The component type this task operates on
- **Validator**: Function to check if task should execute
- **Executor**: Function that performs the task
- **isPureTask**: Flag distinguishing pure vs side-effect

---

## Task Destination

The `Destination` field in `TaskAttributes` has special meaning:

### Purpose
- Routes outbound tasks to specific endpoints
- Used for Nexus callbacks, HTTP webhooks, etc.
- Enables circuit breakers and rate limiting per destination

### Usage
```go
// In a transition function
ctx.AddTask(cb, chasm.TaskAttributes{
    Destination: "https://api.example.com",  // Task routed to this endpoint
}, &InvocationTask{})
```

### Routing Rules
- `Destination != ""` → `CategoryOutbound` (routed to outbound queue)
- `Destination == ""` and immediate → `CategoryTransfer` (local execution)
- `Destination == ""` and scheduled → `CategoryTimer` (delayed local execution)

---

## Complete Example: Callback State Machine

Here's how tasks drive the callback workflow:

```
┌─────────────────┐     EventScheduled      ┌────────────────────┐
│    STANDBY      │ ─────────────────────▶  │    SCHEDULED       │
│                 │                         │                    │
│   (waiting)     │                         │ AddTask(Invoke)    │
└─────────────────┘                         └─────────┬──────────┘
                                                      │
                    InvocationTask executes           │
                    (side-effect - HTTP call)         │
                                                      ▼
                  ┌───────────────────────────────────┴──────────────────────────────────┐
                  │                                                                      │
           Success│                     Retryable Failure                    Non-Retryable│
                  ▼                              │                                       ▼
┌─────────────────┐                              │                         ┌─────────────────┐
│   SUCCEEDED     │                              │                         │    FAILED       │
└─────────────────┘                              ▼                         └─────────────────┘
                                   ┌─────────────────────┐
                                   │   BACKING_OFF       │
                                   │                     │
                                   │ AddTask(Backoff)    │──── BackoffTask executes (pure)
                                   │ (timer scheduled)   │     transitions to SCHEDULED
                                   └─────────────────────┘
```

### Code Flow:

1. **Schedule callback** (creates InvocationTask - side-effect):
```go
var TransitionScheduled = chasm.NewTransition(
    []Status{STANDBY}, SCHEDULED,
    func(cb *Callback, ctx chasm.MutableContext, event EventScheduled) error {
        ctx.AddTask(cb, TaskAttributes{
            Destination: cb.TargetURL,
        }, &InvocationTask{})
        return nil
    },
)
```

2. **InvocationTask executes** (side-effect executor, external to transaction):
```go
func (e InvocationTaskExecutor) Execute(ctx context.Context, ref ComponentRef, ...) error {
    // Read current state
    invokable, _ := chasm.ReadComponent(ctx, ref, loadArgs, nil)
    
    // Make HTTP call (side effect!)
    result := invokable.Invoke(ctx, ...)
    
    // Persist result in new transaction
    chasm.UpdateComponent(ctx, ref, saveResult, result)
    return nil
}
```

3. **On failure, schedule backoff** (creates BackoffTask - pure):
```go
var TransitionAttemptFailed = chasm.NewTransition(
    []Status{SCHEDULED}, BACKING_OFF,
    func(cb *Callback, ctx chasm.MutableContext, event EventAttemptFailed) error {
        nextTime := calculateBackoff(cb.Attempt, event.Err)
        ctx.AddTask(cb, TaskAttributes{
            ScheduledTime: nextTime,  // Timer task
        }, &BackoffTask{Attempt: cb.Attempt})
        return nil
    },
)
```

4. **BackoffTask fires** (pure executor, inside transaction):
```go
func (e BackoffTaskExecutor) Execute(
    ctx chasm.MutableContext,
    cb *Callback,
    attrs TaskAttributes,
    task *BackoffTask,
) error {
    // Transition directly (no new transaction needed)
    return TransitionRescheduled.Apply(cb, ctx, EventRescheduled{})
}
```

---

## Summary Table

| Aspect | Pure Task | Side-Effect Task |
|--------|-----------|------------------|
| **Execution** | Inside current TX | Separate TX |
| **Context** | `MutableContext` | `context.Context` |
| **Can modify state** | Yes, directly | Must call `UpdateComponent()` |
| **I/O allowed** | No | Yes |
| **Use case** | Timers, internal transitions | HTTP calls, external APIs |
| **Immediate support** | Yes (runs in CloseTransaction) | No (always async) |

---

## Key Takeaways

1. **Tasks are the async primitive** - They drive all asynchronous behavior in CHASM
2. **Pure vs Side-Effect is fundamental** - Choose based on whether you need I/O
3. **Validation prevents stale execution** - Always implement meaningful validators
4. **Destinations route outbound tasks** - Enable per-endpoint policies
5. **Immediate pure tasks are special** - They run synchronously in CloseTransaction
6. **Side-effect tasks create new transactions** - They're truly asynchronous

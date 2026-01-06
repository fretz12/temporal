# CHASM Framework Guide

CHASM (Component-based Hierarchical Asynchronous State Machine) is a framework in Temporal Server for building durable, state-machine-based applications. Here's how the various pieces work together:

## Core Concepts Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         CHASM Engine                              │
│  (Orchestrates operations on components via transactions)         │
└──────────────────────────────────────────────────────────────────┘
        │
        ├── NewExecution() - Create new execution with root component
        ├── UpdateComponent() - Mutate component state
        ├── ReadComponent() - Read component state (read-only)
        └── PollComponent() - Wait for a condition to be true
        
┌──────────────────────────────────────────────────────────────────┐
│                           Registry                                │
│  (Registers Libraries, Components, and Tasks at startup)          │
└──────────────────────────────────────────────────────────────────┘
        │
        └── Libraries (namespaced collections of Components + Tasks)
              ├── Components (stateful entities with lifecycle)
              └── Tasks (units of work, Pure or SideEffect)
```

---

## 1. Components (`component.go`)

**Components** are the fundamental building blocks - stateful entities with a lifecycle.

```go
type Component interface {
    // Returns current lifecycle state (Running, Completed, Failed)
    LifecycleState(Context) LifecycleState
    
    // Handle termination requests
    Terminate(MutableContext, TerminateComponentRequest) (TerminateComponentResponse, error)
    
    mustEmbedUnimplementedComponent()
}
```

### Lifecycle States:
- `LifecycleStateRunning` - Component is active
- `LifecycleStateCompleted` - Component finished successfully
- `LifecycleStateFailed` - Component finished with failure

### Example (Callback component):
```go
type Callback struct {
    chasm.UnimplementedComponent  // ← Always embed this for forward compatibility
    
    *callbackspb.CallbackState    // ← Protobuf state (auto-serialized)
    
    CompletionSource chasm.ParentPtr[CompletionSource]  // ← Parent pointer
}

func (c *Callback) LifecycleState(_ chasm.Context) chasm.LifecycleState {
    switch c.Status {
    case callbackspb.CALLBACK_STATUS_SUCCEEDED:
        return chasm.LifecycleStateCompleted
    case callbackspb.CALLBACK_STATUS_FAILED:
        return chasm.LifecycleStateFailed
    default:
        return chasm.LifecycleStateRunning
    }
}
```

---

## 2. Fields (`field.go`)

**Fields** are typed containers for storing data or sub-components within a component. They handle lazy deserialization.

### Field Types:

| Type | Purpose |
|------|---------|
| `Field[T]` | Generic field for data (proto.Message) or sub-components |
| `Map[K, T]` | A map of fields, keyed by primitives |
| `ParentPtr[T]` | Pointer to parent component |

### Creating Fields:
```go
// Store proto data
chasm.NewDataField(ctx, myProtoMessage)

// Store a sub-component
chasm.NewComponentField(ctx, mySubComponent)

// Point to another component (resolved at transaction close)
chasm.ComponentPointerTo(ctx, someComponent)

// Empty field (placeholder)
chasm.NewEmptyField[MyType]()
```

### Accessing Fields:
```go
// Get value (panics if not found)
value := myField.Get(ctx)

// Try to get value (returns ok=false if not found)
value, ok := myField.TryGet(ctx)
```

---

## 3. State Machines (`statemachine.go`)

CHASM provides a **type-safe state machine** abstraction for managing component state transitions.

```go
// A StateMachine can get/set a comparable state
type StateMachine[S comparable] interface {
    StateMachineState() S
    SetStateMachineState(S)
}

// A Transition represents a valid state change
type Transition[S comparable, SM StateMachine[S], E any] struct {
    Sources     []S           // Valid source states
    Destination S             // Target state
    apply       func(SM, MutableContext, E) error  // Side effects
}
```

### Defining Transitions:
```go
// Define an event
type EventScheduled struct{}

// Define a transition
var TransitionScheduled = chasm.NewTransition(
    []callbackspb.CallbackStatus{callbackspb.CALLBACK_STATUS_STANDBY},  // From these states
    callbackspb.CALLBACK_STATUS_SCHEDULED,                                // To this state
    func(cb *Callback, ctx chasm.MutableContext, event EventScheduled) error {
        // Add task, update fields, etc.
        ctx.AddTask(cb, chasm.TaskAttributes{...}, &callbackspb.InvocationTask{})
        return nil
    },
)

// Apply a transition
err := TransitionScheduled.Apply(callback, ctx, EventScheduled{})
```

---

## 4. Tasks (`task.go`, `registrable_task.go`)

**Tasks** are units of asynchronous work. There are two types:

### Pure Tasks
- Run within a CHASM transaction
- Can modify component state directly
- No external side effects

```go
type PureTaskExecutor[C any, T any] interface {
    Execute(MutableContext, C, TaskAttributes, T) error
}
```

### Side Effect Tasks
- Make external calls (HTTP, gRPC, etc.)
- Cannot directly modify state (must use `UpdateComponent` afterward)

```go
type SideEffectTaskExecutor[C any, T any] interface {
    Execute(context.Context, ComponentRef, TaskAttributes, T) error
}
```

### Task Validation
Both types can implement validation to skip obsolete tasks:

```go
type TaskValidator[C any, T any] interface {
    Validate(Context, C, TaskAttributes, T) (bool, error)
}
```

### Adding Tasks (during transitions):
```go
ctx.AddTask(component, chasm.TaskAttributes{
    ScheduledTime: time.Now().Add(10 * time.Second),  // Delayed execution
    Destination:   "https://example.com",              // For routing
}, &myTaskPayload{})
```

---

## 5. Context (`context.go`)

### Immutable Context (`Context`)
For read-only operations:
```go
type Context interface {
    Ref(Component) ([]byte, error)     // Get serialized reference
    Now(Component) time.Time            // Current time (deterministic)
    ExecutionKey() ExecutionKey         // Unique execution identifier
}
```

### Mutable Context (`MutableContext`)
For mutations (extends Context):
```go
type MutableContext interface {
    Context
    AddTask(Component, TaskAttributes, any)  // Schedule a task
}
```

---

## 6. Engine (`engine.go`)

The **Engine** is the main interface for interacting with CHASM executions.

### Operations:

| Method | Purpose | Context Type |
|--------|---------|--------------|
| `NewExecution` | Create new execution with root component | MutableContext |
| `UpdateWithNewExecution` | Create + update in one transaction | MutableContext |
| `UpdateComponent` | Mutate existing component | MutableContext |
| `ReadComponent` | Read component state | Context (immutable) |
| `PollComponent` | Wait for condition to be true | Context (immutable) |

### Usage Pattern:
```go
// Create new execution
output, result, err := chasm.NewExecution(
    ctx,
    chasm.ExecutionKey{NamespaceID: ns, BusinessID: "wf-123"},
    func(ctx chasm.MutableContext, input MyInput) (*MyComponent, MyOutput, error) {
        // Initialize component
        return NewMyComponent(), output, nil
    },
    input,
    chasm.WithBusinessIDPolicy(chasm.BusinessIDReusePolicyRejectDuplicate, ...),
)

// Update existing component
output, newRef, err := chasm.UpdateComponent(
    ctx,
    componentRef,  // or []byte serialized ref
    (*MyComponent).DoSomething,  // Method to call
    input,
)

// Read component (no mutations allowed)
output, err := chasm.ReadComponent(
    ctx,
    componentRef,
    (*MyComponent).GetStatus,
    input,
)
```

---

## 7. Library (`library.go`, `registry.go`)

A **Library** is a namespace containing related components and tasks.

```go
type Library interface {
    Name() string
    Components() []*RegistrableComponent
    Tasks() []*RegistrableTask
    RegisterServices(server *grpc.Server)  // Register gRPC handlers
}
```

### Creating a Library:
```go
type Library struct {
    chasm.UnimplementedLibrary
    // ... dependencies
}

func (l *Library) Name() string { return "callback" }

func (l *Library) Components() []*chasm.RegistrableComponent {
    return []*chasm.RegistrableComponent{
        chasm.NewRegistrableComponent[*Callback]("callback",
            chasm.WithSearchAttributes(...),  // For visibility
            chasm.WithEphemeral(),            // Don't persist (optional)
        ),
    }
}

func (l *Library) Tasks() []*chasm.RegistrableTask {
    return []*chasm.RegistrableTask{
        chasm.NewRegistrableSideEffectTask("invoke", validator, executor),
        chasm.NewRegistrablePureTask("backoff", validator, executor),
    }
}
```

---

## 8. Component References (`ref.go`)

**ComponentRef** identifies a specific component within an execution.

```go
type ExecutionKey struct {
    NamespaceID string
    BusinessID  string  // Workflow ID equivalent
    RunID       string
}

type ComponentRef struct {
    ExecutionKey
    archetypeID     ArchetypeID      // Root component type ID
    componentPath   []string          // Path to component in tree
    // ... versioning fields
}
```

### Creating References:
```go
// Reference to root component of an execution
ref := chasm.NewComponentRef[*MyComponent](executionKey)

// Deserialize from bytes
ref, err := chasm.DeserializeComponentRef(serializedBytes)
```

---

## 9. Tree Structure (`tree.go`)

Components are organized in a **tree structure**:

```
Root Component (e.g., Workflow)
├── Field (proto data)
├── Sub-Component (e.g., Activity)
│   ├── Field (proto data)
│   └── Field (proto data)
├── Map["key1"] → Sub-Component
└── Map["key2"] → Sub-Component
```

The `Node` struct represents each node in this tree, handling:
- Serialization/deserialization of values
- Tracking dirty state for persistence
- Generating tasks
- Managing visibility updates

---

## 10. Visibility (`visibility.go`)

CHASM supports **search attributes** for querying executions.

### Defining Search Attributes:
```go
var TypeSearchAttribute = chasm.NewSearchAttributeKeyword(
    "ActivityType",                        // Alias (user-facing name)
    chasm.SearchAttributeFieldKeyword01,   // Reserved field slot
)

// In component registration
chasm.NewRegistrableComponent[*Activity]("activity",
    chasm.WithSearchAttributes(
        TypeSearchAttribute,
        StatusSearchAttribute,
    ),
)
```

### Providing Search Attributes:
```go
// Implement on your root component
func (a *Activity) SearchAttributes(ctx chasm.Context) []chasm.SearchAttributeKeyValue {
    return []chasm.SearchAttributeKeyValue{
        TypeSearchAttribute.Value(a.ActivityType.Name),
        StatusSearchAttribute.Value(a.Status.String()),
    }
}
```

---

## Putting It All Together: Building a CHASM Library

Here's the typical workflow for building a new CHASM library:

### 1. Define your component:
```go
type MyComponent struct {
    chasm.UnimplementedComponent
    *mypb.MyState                          // Protobuf state
    SubItems chasm.Map[string, *SubItem]   // Sub-components
}
```

### 2. Implement lifecycle:
```go
func (c *MyComponent) LifecycleState(ctx chasm.Context) chasm.LifecycleState { ... }
```

### 3. Define state machine (optional):
```go
var TransitionStart = chasm.NewTransition(
    []mypb.Status{mypb.STATUS_PENDING},
    mypb.STATUS_RUNNING,
    func(c *MyComponent, ctx chasm.MutableContext, e EventStart) error { ... },
)
```

### 4. Create task executors:
```go
type MyTaskExecutor struct { ... }
func (e *MyTaskExecutor) Validate(...) (bool, error) { ... }
func (e *MyTaskExecutor) Execute(...) error { ... }
```

### 5. Create the library:
```go
type Library struct { chasm.UnimplementedLibrary }
func (l *Library) Name() string { return "mylib" }
func (l *Library) Components() []*chasm.RegistrableComponent { ... }
func (l *Library) Tasks() []*chasm.RegistrableTask { ... }
```

### 6. Register with fx module:
```go
var Module = fx.Module("chasm.mylib",
    fx.Provide(newLibrary),
    fx.Invoke(func(r *chasm.Registry, l *Library) error {
        return r.Register(l)
    }),
)
```

---

## Existing Libraries in `chasm/lib/`

| Library | Purpose |
|---------|---------|
| `workflow/` | Workflow execution (legacy integration) |
| `activity/` | Activity execution with timeouts, retries |
| `callback/` | Nexus callback handling |
| `scheduler/` | Scheduled workflow executions |

These serve as reference implementations for building your own CHASM libraries!

# CHASM Engine Deep Dive

The CHASM Engine is the **runtime orchestrator** that manages CHASM executions within Temporal Server. It handles persistence, locking, transactions, and routing operations to the correct shard.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Application Code                                │
│        (e.g., callback/handler.go, activity/handler.go)                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     chasm/engine.go (Generic API)                           │
│                                                                             │
│  NewExecution[C,I,O]()  │  UpdateComponent[C,R,I,O]()  │  ReadComponent()  │
│  PollComponent()        │  UpdateWithNewExecution()     │                   │
│                                                                             │
│  - Type-safe generic wrappers                                               │
│  - Extracts engine from context                                             │
│  - Converts between []byte and ComponentRef                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Engine Interface (chasm/engine.go)                       │
│                                                                             │
│  type Engine interface {                                                    │
│      NewExecution(ctx, ref, newFn, opts) (NewExecutionResult, error)       │
│      UpdateComponent(ctx, ref, updateFn, opts) ([]byte, error)             │
│      ReadComponent(ctx, ref, readFn, opts) error                           │
│      PollComponent(ctx, ref, predicate, opts) ([]byte, error)              │
│      NotifyExecution(ExecutionKey)                                          │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              service/history/chasm_engine.go (Implementation)               │
│                                                                             │
│  type ChasmEngine struct {                                                  │
│      executionCache   cache.Cache         // Workflow execution cache       │
│      shardController  shard.Controller    // Access to shards               │
│      registry         *chasm.Registry     // Component/Task registry        │
│      config           *configs.Config     // Server configuration           │
│      notifier         *ChasmNotifier      // PollComponent notifications    │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
         ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
         │  Shard Context   │ │  Execution Cache │ │  Persistence     │
         │  (locking, time) │ │  (mutable state) │ │  (database)      │
         └──────────────────┘ └──────────────────┘ └──────────────────┘
```

---

## The Two Layers

### Layer 1: Generic Type-Safe API (`chasm/engine.go`)

These are convenience functions that application code calls. They:
1. Extract the `Engine` from `context.Context`
2. Provide type-safety via Go generics
3. Convert between `[]byte` and `ComponentRef`

```go
// Generic wrapper - what app code calls
func NewExecution[C Component, I any, O any](
    ctx context.Context,
    key ExecutionKey,
    newFn func(MutableContext, I) (C, O, error),
    input I,
    opts ...TransitionOption,
) (O, NewExecutionResult, error) {
    var output O
    result, err := engineFromContext(ctx).NewExecution(  // ← Extracts engine from context
        ctx,
        NewComponentRef[C](key),                          // ← Creates typed ref
        func(ctx MutableContext) (Component, error) {
            var c C
            c, output, err = newFn(ctx, input)            // ← Captures output
            return c, err
        },
        opts...,
    )
    return output, result, nil
}
```

### Layer 2: Concrete Implementation (`service/history/chasm_engine.go`)

The actual engine that interacts with Temporal's internal systems:

```go
type ChasmEngine struct {
    executionCache  cache.Cache       // In-memory cache of MutableState
    shardController shard.Controller  // Routes to correct shard
    registry        *chasm.Registry   // Type registry
    config          *configs.Config   // Server config
    notifier        *ChasmNotifier    // For long-polling
}
```

---

## Operation Deep Dives

### 1. `NewExecution` - Creating a New CHASM Execution

This is the most complex operation. Here's the flow:

```
NewExecution(ctx, ref, newFn, opts)
         │
         ▼
┌────────────────────────────────────────────────┐
│ 1. constructTransitionOptions()                │
│    - Apply WithBusinessIDPolicy, WithRequestID │
│    - Generate UUID for requestID if not set    │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 2. getShardContext(ref)                        │
│    - Hash BusinessID to get ShardID            │
│    - Get ShardContext from ShardController     │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 3. lockCurrentExecution()                      │
│    - Lock the "current execution" for this     │
│      (namespaceID, businessID, archetypeID)    │
│    - Prevents concurrent creates with same ID  │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 4. createNewExecution()                        │
│    a) Generate new RunID (UUID)                │
│    b) Create new MutableState                  │
│    c) Get CHASM tree from MutableState         │
│    d) Create MutableContext from tree          │
│    e) Call newFn() to create root component    │
│    f) tree.SetRootComponent(component)         │
│    g) mutableState.CloseTransactionAsSnapshot()│
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 5. persistAsBrandNew()                         │
│    - Try to create in DB as brand new          │
│    - If success → return Created=true          │
│    - If conflict → return current run info     │
└────────────────────────────────────────────────┘
         │
    ┌────┴────┐
    │ Conflict │
    └────┬────┘
         ▼
┌────────────────────────────────────────────────┐
│ 6. handleExecutionConflict()                   │
│    - Check if dedupe via requestID             │
│    - Check failover version                    │
│    - Route based on current state:             │
│      * RUNNING → handleConflictPolicy()        │
│      * COMPLETED → handleReusePolicy()         │
└────────────────────────────────────────────────┘
```

#### Conflict and Reuse Policies

```go
// Conflict Policy - what to do when execution with same BusinessID is RUNNING
type BusinessIDConflictPolicy int
const (
    BusinessIDConflictPolicyFail            // Fail the request
    BusinessIDConflictPolicyTerminateExisting // Terminate old, create new (NOT IMPLEMENTED)
    BusinessIDConflictPolicyUseExisting     // Return existing execution's ref
)

// Reuse Policy - what to do when execution with same BusinessID is COMPLETED
type BusinessIDReusePolicy int
const (
    BusinessIDReusePolicyAllowDuplicate          // Always allow creating new
    BusinessIDReusePolicyAllowDuplicateFailedOnly // Allow only if previous failed
    BusinessIDReusePolicyRejectDuplicate         // Never allow reuse
)
```

---

### 2. `UpdateComponent` - Mutating Component State

```
UpdateComponent(ctx, ref, updateFn, opts)
         │
         ▼
┌────────────────────────────────────────────────┐
│ 1. getExecutionLease(ctx, ref)                 │
│    a) getShardContext() - route to shard       │
│    b) GetChasmLeaseWithConsistencyCheck()      │
│       - Load MutableState from cache/DB        │
│       - Check ref not stale (via IsStale)      │
│       - Reload if state is stale               │
│    c) Return lease with lock held              │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 2. Get CHASM tree from MutableState            │
│    chasmTree := mutableState.ChasmTree()       │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 3. Create MutableContext                       │
│    mutableCtx := chasm.NewMutableContext(...)  │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 4. Retrieve component from tree                │
│    component := chasmTree.Component(ctx, ref)  │
│    - Validates path, access rules, etc.        │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 5. Execute user function                       │
│    updateFn(mutableCtx, component)             │
│    - Mutates component state                   │
│    - May add tasks via ctx.AddTask()           │
│    - May create sub-components                 │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 6. Persist changes                             │
│    executionLease.GetContext().               │
│        UpdateWorkflowExecutionAsActive(...)    │
│    - Tree.CloseTransaction() is called         │
│    - Syncs dirty nodes to persistence          │
│    - Generates physical tasks                  │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│ 7. Return new component ref                    │
│    return mutableCtx.Ref(component)            │
└────────────────────────────────────────────────┘
```

---

### 3. `ReadComponent` - Read-Only Access

Similar to `UpdateComponent` but:
- Creates `Context` instead of `MutableContext`
- No persistence update
- Lock is always released with `nil` error (no cache invalidation needed)

```go
func (e *ChasmEngine) ReadComponent(...) error {
    _, executionLease, err := e.getExecutionLease(ctx, ref)
    defer executionLease.GetReleaseFn()(nil)  // ← Always nil!
    
    chasmTree := executionLease.GetMutableState().ChasmTree().(*chasm.Node)
    chasmContext := chasm.NewContext(ctx, chasmTree)  // ← Immutable context
    component := chasmTree.Component(chasmContext, ref)
    
    return readFn(chasmContext, component)  // ← Read-only
}
```

---

### 4. `PollComponent` - Long-Polling for Conditions

This is used for waiting until a predicate becomes true (e.g., waiting for activity completion).

```
PollComponent(ctx, ref, predicate, opts)
         │
         ▼
┌────────────────────────────────────────────────┐
│ checkPredicateOrSubscribe()                    │
│                                                │
│  1. Get execution lease                        │
│  2. Check if predicate is already satisfied    │
│     - If yes → return ref immediately          │
│  3. Subscribe to ChasmNotifier BEFORE          │
│     releasing the lock                         │
│     (prevents missing notifications)           │
│  4. Release lease                              │
└────────────────────────────────────────────────┘
         │
    ┌────┴────┐
    │Not ready│
    └────┬────┘
         ▼
┌────────────────────────────────────────────────┐
│ Wait loop:                                     │
│                                                │
│  for {                                         │
│      select {                                  │
│      case <-ch:    // Notification received    │
│          checkPredicateOrSubscribe()           │
│      case <-ctx.Done():                        │
│          return ctx.Err()                      │
│      }                                         │
│  }                                             │
└────────────────────────────────────────────────┘
```

#### The ChasmNotifier

```go
// Manages subscriptions for execution changes
type ChasmNotifier struct {
    executions map[chasm.ExecutionKey]*subscriptionTracker
    lock       sync.Mutex
}

// Subscribe returns a channel that closes on notification
func (n *ChasmNotifier) Subscribe(key ExecutionKey) (<-chan struct{}, func()) {
    // ...
}

// Called when execution state changes (e.g., after UpdateComponent)
func (n *ChasmNotifier) Notify(key ExecutionKey) {
    n.lock.Lock()
    defer n.lock.Unlock()
    if s, ok := n.executions[key]; ok {
        close(s.ch)  // Wake up all waiters
        delete(n.executions, key)
    }
}
```

**Key insight**: The predicate must be **monotonic** - once true, it stays true. This ensures no missed notifications.

---

## The Execution Lease Pattern

The `getExecutionLease` method is critical for correctness:

```go
func (e *ChasmEngine) getExecutionLease(ctx, ref) (ShardContext, WorkflowLease, error) {
    shardContext := e.getShardContext(ref)
    
    consistencyChecker := api.NewWorkflowConsistencyChecker(shardContext, e.executionCache)
    
    executionLease, err := consistencyChecker.GetChasmLeaseWithConsistencyCheck(
        ctx,
        nil,
        func(mutableState) bool {
            // Check if ref is consistent with current state
            err := mutableState.ChasmTree().IsStale(ref)
            if errors.Is(err, ErrStaleState) {
                return false  // Reload from DB
            }
            staleReferenceErr = err  // Reference itself is stale
            return true
        },
        workflowKey,
        archetypeID,
        lockPriority,
    )
    
    return shardContext, executionLease, err
}
```

**Consistency guarantees**:
1. **ErrStaleState**: In-memory state is behind the ref → reload from DB
2. **ErrStaleReference**: Ref points to old version → fail the request
3. Lock is held while operating on MutableState

---

## The CHASM Tree Integration

The engine interacts with `chasm.Node` (the tree) via MutableState:

```go
// Get tree from MutableState
chasmTree := mutableState.ChasmTree().(*chasm.Node)

// Create context for operations
mutableCtx := chasm.NewMutableContext(ctx, chasmTree)

// Get component by reference
component := chasmTree.Component(mutableCtx, ref)

// After mutations, close transaction (called by MutableState)
mutation, err := chasmTree.CloseTransaction()
```

The tree's `CloseTransaction()` does:
1. Execute immediate pure tasks
2. Sync sub-component structure
3. Resolve deferred pointers
4. Handle root lifecycle changes
5. Update visibility if needed
6. Serialize dirty nodes
7. Generate physical task records

---

## Summary: How a Typical Flow Works

```
1. API Handler receives request
         │
         ▼
2. Handler calls chasm.UpdateComponent(ctx, ref, fn, input)
         │
         ▼
3. Generic wrapper extracts Engine from ctx, calls Engine.UpdateComponent
         │
         ▼
4. ChasmEngine.UpdateComponent:
   a) Routes to correct shard via ref.ShardingKey → ShardID
   b) Gets lease (lock + MutableState) via consistency checker
   c) Gets CHASM tree from MutableState
   d) Creates MutableContext wrapping tree
   e) Retrieves component from tree by ref
   f) Calls user's updateFn(mutableCtx, component)
   g) User code mutates component, adds tasks via ctx.AddTask()
   h) Persists via UpdateWorkflowExecutionAsActive (triggers CloseTransaction)
   i) Returns new serialized ref
         │
         ▼
5. Tasks are executed asynchronously:
   - Pure tasks: within same transaction
   - Side-effect tasks: via task queues
         │
         ▼
6. Waiters on PollComponent are notified via ChasmNotifier
```

The beauty of this design is that **application code only deals with type-safe generic functions** while the engine handles all the complexity of sharding, locking, persistence, and consistency.

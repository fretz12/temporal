# CHASM API Calls That Trigger Transactions

From an **application developer's perspective**, here are the API calls that open and close CHASM transactions:

## Transaction-Creating APIs (Mutations)

These APIs **open AND close** a transaction atomically:

### 1. `chasm.NewExecution()` - Create New Execution

```go
output, result, err := chasm.NewExecution(
    ctx,
    chasm.ExecutionKey{NamespaceID: ns, BusinessID: "my-id"},
    func(ctx chasm.MutableContext, input MyInput) (*MyComponent, MyOutput, error) {
        // ← TRANSACTION IS OPEN HERE
        component := &MyComponent{...}
        ctx.AddTask(component, attrs, task)  // Can add tasks
        return component, output, nil
    },
    input,
    chasm.WithBusinessIDPolicy(...),
)
// ← TRANSACTION CLOSED, persisted to DB
```

**What happens internally:**
1. Creates new MutableState
2. Creates MutableContext (TX opens)
3. Calls your `newFn`
4. `CloseTransactionAsSnapshot()` (TX closes)
5. Persists to database

---

### 2. `chasm.UpdateComponent()` - Mutate Existing Component

```go
output, newRef, err := chasm.UpdateComponent(
    ctx,
    componentRef,  // or []byte serialized ref
    func(c *MyComponent, ctx chasm.MutableContext, input MyInput) (MyOutput, error) {
        // ← TRANSACTION IS OPEN HERE
        c.Status = "updated"
        ctx.AddTask(c, attrs, task)
        return output, nil
    },
    input,
)
// ← TRANSACTION CLOSED, persisted to DB
```

**What happens internally:**
1. Loads MutableState from cache/DB (acquires lock)
2. Creates MutableContext (TX opens)
3. Retrieves component from tree
4. Calls your `updateFn`
5. `UpdateWorkflowExecutionAsActive()` → `CloseTransactionAsMutation()` (TX closes)
6. Persists to database
7. Releases lock

---

### 3. `chasm.UpdateWithNewExecution()` - Create + Update Atomically

```go
out1, out2, key, ref, err := chasm.UpdateWithNewExecution(
    ctx,
    executionKey,
    func(ctx chasm.MutableContext, input I) (*MyComponent, O1, error) {
        // ← TX OPEN: Create component
        return &MyComponent{}, output1, nil
    },
    func(c *MyComponent, ctx chasm.MutableContext, input I) (O2, error) {
        // ← TX STILL OPEN: Update component
        return output2, nil
    },
    input,
)
// ← TRANSACTION CLOSED
```

---

## Read-Only APIs (No Transaction)

These APIs **DO NOT open a transaction** - they only read:

### 4. `chasm.ReadComponent()` - Read Component State

```go
output, err := chasm.ReadComponent(
    ctx,
    componentRef,
    func(c *MyComponent, ctx chasm.Context, input MyInput) (MyOutput, error) {
        // ← NO TRANSACTION - read-only Context (not MutableContext)
        return MyOutput{Status: c.Status}, nil
    },
    input,
)
```

**What happens internally:**
1. Loads MutableState (acquires lock)
2. Creates **immutable** `Context` (NOT MutableContext)
3. Calls your `readFn`
4. Releases lock (no persistence, no CloseTransaction)

---

### 5. `chasm.PollComponent()` - Long-Poll for Condition

```go
output, newRef, err := chasm.PollComponent(
    ctx,
    componentRef,
    func(c *MyComponent, ctx chasm.Context, input MyInput) (MyOutput, bool, error) {
        // ← NO TRANSACTION - read-only
        if c.Status == "completed" {
            return MyOutput{}, true, nil  // Condition satisfied
        }
        return MyOutput{}, false, nil  // Keep waiting
    },
    input,
)
```

**What happens internally:**
1. Loads MutableState, checks predicate
2. If not satisfied, subscribes to notifications and waits
3. On notification, re-checks predicate
4. Never opens a transaction (read-only)

---

## State Machine Transitions (Within a Transaction)

When you're **already inside** a transaction (inside `newFn` or `updateFn`), you can use:

### 6. `Transition.Apply()` - Apply State Machine Transition

```go
chasm.UpdateComponent(ctx, ref, func(cb *Callback, ctx chasm.MutableContext, _ any) (any, error) {
    // Already inside a transaction
    
    // Apply a state transition (stays within same TX)
    err := TransitionScheduled.Apply(cb, ctx, EventScheduled{})
    
    return nil, err
}, nil)
```

This doesn't create a new transaction - it operates within the existing one.

---

## Task Executors (Implicit Transactions)

### 7. Pure Task Executor - Runs Within Existing Transaction

```go
type BackoffTaskExecutor struct{}

func (e *BackoffTaskExecutor) Execute(
    ctx chasm.MutableContext,  // ← Already in a transaction
    cb *Callback,
    attrs chasm.TaskAttributes,
    task *BackoffTask,
) error {
    // This runs WITHIN the CloseTransaction() call
    // as part of executeImmediatePureTasks()
    return TransitionRescheduled.Apply(cb, ctx, EventRescheduled{})
}
```

Pure tasks execute **inside** the existing transaction during `CloseTransaction()`.

---

### 8. Side-Effect Task Executor - Creates New Transaction

```go
type InvocationTaskExecutor struct{}

func (e *InvocationTaskExecutor) Execute(
    ctx context.Context,        // ← Regular context, NOT MutableContext
    ref chasm.ComponentRef,
    attrs chasm.TaskAttributes,
    task *InvocationTask,
) error {
    // 1. Read component (no TX)
    invokable, _ := chasm.ReadComponent(ctx, ref, (*Callback).loadInvocationArgs, nil)

    // 2. Make external HTTP call (side effect)
    result := invokable.Invoke(ctx, ...)

    // 3. Update component with result (NEW TX opens and closes)
    _, _, err := chasm.UpdateComponent(ctx, ref, (*Callback).saveResult, result)
    return err
}
```

Side-effect executors receive a regular `context.Context` and must explicitly call `UpdateComponent()` to persist changes.

---

## Summary Table

| API | Opens TX? | Closes TX? | Use Case |
|-----|-----------|------------|----------|
| `chasm.NewExecution()` | ✅ | ✅ | Create new execution with root component |
| `chasm.UpdateComponent()` | ✅ | ✅ | Mutate existing component |
| `chasm.UpdateWithNewExecution()` | ✅ | ✅ | Create + update atomically |
| `chasm.ReadComponent()` | ❌ | ❌ | Read-only access |
| `chasm.PollComponent()` | ❌ | ❌ | Wait for condition |
| `Transition.Apply()` | ❌ | ❌ | Within existing TX |
| `PureTaskExecutor.Execute()` | ❌ | ❌ | Runs within existing TX |
| `SideEffectTaskExecutor.Execute()` | ❌ | ❌ | Must call UpdateComponent() |

---

## Visual Flow

```
Application Code
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│  chasm.UpdateComponent(ctx, ref, updateFn, input)            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  TRANSACTION BOUNDARY                                  │  │
│  │                                                        │  │
│  │  1. Load MutableState (acquire lock)                   │  │
│  │  2. NewMutableContext() ← TX OPENS                     │  │
│  │  3. Get component from tree                            │  │
│  │  4. updateFn(component, ctx, input) ← YOUR CODE        │  │
│  │     │                                                  │  │
│  │     ├── Modify component fields                        │  │
│  │     ├── ctx.AddTask(...)                               │  │
│  │     ├── Transition.Apply(...)                          │  │
│  │     └── Create/delete sub-components                   │  │
│  │                                                        │  │
│  │  5. CloseTransactionAsMutation() ← TX CLOSES           │  │
│  │     └── chasmTree.CloseTransaction()                   │  │
│  │         ├── executeImmediatePureTasks()                │  │
│  │         ├── syncSubComponents()                        │  │
│  │         ├── serializeNodes()                           │  │
│  │         └── generatePhysicalTasks()                    │  │
│  │                                                        │  │
│  │  6. Persist to database                                │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  7. Release lock                                             │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
    Returns (output, newRef, error)
```

---

## Key Takeaways

1. **Transactions are automatic** - You don't manually open/close them. The CHASM APIs handle it.

2. **Mutation APIs = Transaction** - `NewExecution`, `UpdateComponent`, `UpdateWithNewExecution` all create transactions.

3. **Read APIs = No Transaction** - `ReadComponent`, `PollComponent` are read-only.

4. **One transaction per API call** - Each `UpdateComponent()` call is one atomic transaction.

5. **Pure tasks run inside transactions** - They execute during `CloseTransaction()`.

6. **Side-effect tasks create new transactions** - They must explicitly call `UpdateComponent()` to persist.

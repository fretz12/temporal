# CHASM Tree Structure Deep Dive

The CHASM tree is the core in-memory data structure that manages the hierarchical state of CHASM components. It handles serialization, deserialization, dirty tracking, and transaction management.

## Tree Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CHASM Tree (Node)                              │
│                                                                             │
│  Root Node (Component: e.g., Workflow)                                      │
│  ├── serializedNode: ChasmNode proto (persisted state)                     │
│  ├── value: deserialized Go struct (runtime state)                          │
│  ├── valueState: sync status (NeedDeserialize | Synced | NeedSerialize)    │
│  ├── children: map[string]*Node                                             │
│  │   ├── "Callbacks" (Collection Node for Map)                              │
│  │   │   ├── "cb-1" (Component Node: Callback)                              │
│  │   │   └── "cb-2" (Component Node: Callback)                              │
│  │   ├── "Activity" (Component Node)                                        │
│  │   │   └── "LastHeartbeat" (Data Node)                                    │
│  │   └── "Config" (Data Node)                                               │
│  └── nodeBase: shared state across all nodes                                │
│       ├── registry: *Registry                                               │
│       ├── mutation: NodesMutation (tracks changes)                          │
│       ├── newTasks: map[component][]task                                    │
│       └── isActiveStateDirty: bool                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Node Types

The tree consists of four types of nodes, determined by `serializedNode.Metadata.Attributes`:

### 1. Component Node (`ComponentAttributes`)
- Represents a CHASM Component (implements `Component` interface)
- Has lifecycle state (Running, Completed, Failed)
- Can have child nodes (sub-components, data, maps)
- Stores tasks (pure tasks and side-effect tasks)

```go
type ChasmComponentAttributes struct {
    TypeId          uint32                          // Registered component type ID
    PureTasks       []*ChasmComponentAttributes_Task
    SideEffectTasks []*ChasmComponentAttributes_Task
}
```

### 2. Data Node (`DataAttributes`)
- Stores protobuf data (implements `proto.Message`)
- Leaf node (no children)
- Used for embedded proto state within components

### 3. Collection Node (`CollectionAttributes`)
- Represents a `chasm.Map[K, Field[T]]`
- Children are keyed by the map key (converted to string)
- Intermediate node - doesn't hold data itself

### 4. Pointer Node (`PointerAttributes`)
- Stores a path reference to another node in the tree
- Used for `ComponentPointerTo` and `DataPointerTo`
- Resolved at transaction close

---

## The Node Structure

```go
type Node struct {
    *nodeBase  // Shared state for all nodes in tree

    parent   *Node             // Parent node (nil for root)
    children map[string]*Node  // Child nodes by name
    nodeName string            // Key in parent's children map

    serializedNode *persistencespb.ChasmNode  // Persisted proto
    value          any                         // Deserialized Go value
    valueState     valueState                  // Sync status

    encodedPath *string  // Cached path (e.g., "Callbacks/cb-1")
    terminated  bool     // Force-closed flag (root only)
}
```

### nodeBase (Shared State)

All nodes share a `nodeBase` that tracks tree-wide state:

```go
type nodeBase struct {
    registry    *Registry        // Component/Task type registry
    timeSource  clock.TimeSource
    backend     NodeBackend      // MutableState interface
    pathEncoder NodePathEncoder  // Encodes paths for persistence
    logger      log.Logger

    // Transaction-scoped changes (cleared after CloseTransaction)
    mutation       NodesMutation              // User data changes (replicated)
    systemMutation NodesMutation              // System changes (not replicated)
    newTasks       map[any][]taskWithAttributes
    immediatePureTasks map[any][]taskWithAttributes

    isActiveStateDirty bool  // Any user data mutated?

    // Visibility tracking
    currentSA   map[string]VisibilityValue
    currentMemo proto.Message

    needsPointerResolution bool
}
```

---

## Value State Machine

Each node tracks its sync status between the deserialized `value` and the `serializedNode`:

```
                    ┌─────────────────────┐
                    │  valueStateUndefined │  (initial)
                    └──────────┬──────────┘
                               │ Load from DB
                               ▼
                    ┌─────────────────────┐
              ┌────│ valueStateNeedDeserialize │
              │     └──────────┬──────────┘
              │                │ deserialize()
              │                ▼
              │     ┌─────────────────────┐
              │     │   valueStateSynced   │ ◄─────────────────┐
              │     └──────────┬──────────┘                    │
              │                │ Access via MutableContext     │ serialize()
              │                ▼                               │
              │     ┌─────────────────────┐                    │
              │     │ valueStateNeedSerialize │ ────────────────┘
              │     └──────────┬──────────┘
              │                │ Modify component structure
              │                ▼
              │     ┌──────────────────────────┐
              └────►│ valueStateNeedSyncStructure │
                    └──────────────────────────┘
                               │ syncSubComponents()
                               ▼
                    ┌─────────────────────┐
                    │ valueStateNeedSerialize │
                    └─────────────────────┘
```

**Key insight**: Dirtiness is tracked at increasing levels:
1. `NeedDeserialize` - Not even loaded yet
2. `Synced` - In-memory matches persisted
3. `NeedSerialize` - Data changed, tree structure is synced
4. `NeedSyncStructure` - Tree structure may have changed (components added/removed)

---

## Tree Operations

### 1. Loading from Database (`NewTreeFromDB`)

```go
func NewTreeFromDB(
    serializedNodes map[string]*persistencespb.ChasmNode, // path -> node
    registry, timeSource, backend, pathEncoder, logger,
) (*Node, error) {
    root := newTreeHelper(...)
    
    for encodedPath, serializedNode := range serializedNodes {
        nodePath := pathEncoder.Decode(encodedPath)  // e.g., "Callbacks/cb-1" -> ["Callbacks", "cb-1"]
        root.setSerializedNode(nodePath, encodedPath, serializedNode)
    }
    
    // Initialize search attributes from root component
    newTreeInitSearchAttributesAndMemo(root)
    return root, nil
}
```

### 2. Finding a Node (`findNode`)

```go
func (n *Node) findNode(path []string) (*Node, bool) {
    if len(path) == 0 {
        return n, true  // Found!
    }
    
    childName := path[0]
    childNode, ok := n.children[childName]
    if !ok {
        return nil, false
    }
    return childNode.findNode(path[1:])  // Recurse
}
```

### 3. Traversing the Tree (`andAllChildren`)

Uses Go 1.23 iterators for depth-first traversal:

```go
func (n *Node) andAllChildren() iter.Seq2[[]string, *Node] {
    return func(yield func([]string, *Node) bool) {
        var walk func([]string, *Node) bool
        walk = func(path []string, node *Node) bool {
            if !yield(path, node) {
                return false
            }
            for _, child := range node.children {
                if !walk(append(path, child.nodeName), child) {
                    return false
                }
            }
            return true
        }
        walk(nil, n)
    }
}

// Usage:
for path, node := range root.andAllChildren() {
    fmt.Println(path, node.value)
}
```

### 4. Syncing Component Structure (`syncSubComponents`)

When a component's fields are modified, the tree structure must be synchronized:

```go
func (n *Node) syncSubComponents() error {
    if n.valueState < valueStateNeedSyncStructure {
        return nil  // Already synced
    }
    
    childrenToKeep := make(map[string]struct{})
    
    for field := range n.valueFields() {
        switch field.kind {
        case fieldKindSubField:  // Field[T]
            // Create/update child node for this field
            childNode := n.children[field.name]
            if childNode == nil {
                childNode = newNode(n.nodeBase, n, field.name)
                n.children[field.name] = childNode
            }
            childrenToKeep[field.name] = struct{}{}
            
        case fieldKindSubMap:  // Map[K, Field[T]]
            // Create collection node and sync each item
            collectionNode := n.children[field.name]
            for _, mapKey := range field.val.MapKeys() {
                collectionKey := n.mapKeyToString(mapKey)
                collectionNode.syncSubField(...)
            }
            childrenToKeep[field.name] = struct{}{}
        }
    }
    
    // Delete nodes that are no longer in the component
    n.deleteChildren(childrenToKeep)
    n.setValueState(valueStateNeedSerialize)
    return nil
}
```

---

## Field Reflection

The tree uses reflection to discover CHASM-managed fields in components:

```go
type Callback struct {
    chasm.UnimplementedComponent
    
    *callbackspb.CallbackState           // fieldKindData (proto.Message)
    
    CompletionSource chasm.ParentPtr[...] // fieldKindParentPtr
}

type Activity struct {
    chasm.UnimplementedComponent
    
    *activitypb.ActivityState            // fieldKindData
    
    Visibility    chasm.Field[*Visibility]           // fieldKindSubField
    LastAttempt   chasm.Field[*ActivityAttemptState] // fieldKindSubField
    RequestData   chasm.Field[*ActivityRequestData]  // fieldKindSubField
    Store         chasm.Field[ActivityStore]         // fieldKindSubField (pointer to parent)
}

type Workflow struct {
    chasm.UnimplementedComponent
    
    *emptypb.Empty                       // fieldKindData
    
    chasm.MSPointer                      // fieldKindMutableState
    Callbacks chasm.Map[string, *Callback] // fieldKindSubMap
}
```

The `fieldsOf` iterator discovers these:

```go
for field := range fieldsOf(componentValue) {
    switch field.kind {
    case fieldKindData:       // Proto message (one per component)
    case fieldKindSubField:   // chasm.Field[T]
    case fieldKindSubMap:     // chasm.Map[K, Field[T]]
    case fieldKindMutableState: // chasm.MSPointer
    case fieldKindParentPtr:  // chasm.ParentPtr[T]
    }
}
```

---

## Serialization and Deserialization

### Serializing a Component

```go
func (n *Node) serializeComponentNode() error {
    for field := range n.valueFields() {
        if field.kind == fieldKindData {
            // Serialize the proto data into serializedNode.Data
            blob := serialization.ProtoEncode(field.val.Interface().(proto.Message))
            n.serializedNode.Data = blob
        }
        // Sub-fields are serialized via their own child nodes
    }
    n.updateLastUpdateVersionedTransition()
    n.setValueState(valueStateSynced)
    return nil
}
```

### Deserializing a Component

```go
func (n *Node) deserializeComponentNode(valueT reflect.Type) error {
    valueV := reflect.New(valueT.Elem())  // Create new instance
    
    for field := range fieldsOf(valueV) {
        switch field.kind {
        case fieldKindData:
            // Deserialize proto from serializedNode.Data
            value := unmarshalProto(n.serializedNode.Data, field.typ)
            field.val.Set(value)
            
        case fieldKindSubField:
            // Link to existing child node
            if childNode, found := n.children[field.name]; found {
                chasmFieldV := reflect.New(field.typ).Elem()
                internal := newFieldInternalWithNode(childNode)
                chasmFieldV.FieldByName("Internal").Set(reflect.ValueOf(internal))
                field.val.Set(chasmFieldV)
            }
            
        case fieldKindSubMap:
            // Link each map item to its collection item node
            for itemName, itemNode := range collectionNode.children {
                // ... similar linking
            }
            
        case fieldKindParentPtr:
            // Initialize parent pointer with current node
            parentPtrV := reflect.New(field.typ).Elem()
            parentPtrV.FieldByName("Internal").Set(
                reflect.ValueOf(parentPtrInternal{currentNode: n}))
            field.val.Set(parentPtrV)
        }
    }
    
    n.value = valueV.Interface()
    n.setValueState(valueStateSynced)
    return nil
}
```

---

## Transaction Flow (`CloseTransaction`)

When MutableState closes a transaction, the tree finalizes all changes:

```go
func (n *Node) CloseTransaction() (NodesMutation, error) {
    defer n.cleanupTransaction()
    
    // 1. Execute immediate pure tasks (within transaction)
    n.executeImmediatePureTasks()
    
    // 2. Sync tree structure with component values
    n.syncSubComponents()
    
    // 3. Resolve deferred pointers
    if n.needsPointerResolution {
        n.resolveDeferredPointers()
    }
    
    // 4. Handle root lifecycle changes (Running → Completed)
    rootLifecycleChanged := n.closeTransactionHandleRootLifecycleChange()
    
    // 5. Update visibility if needed
    if n.isActiveStateDirty {
        n.closeTransactionForceUpdateVisibility(rootLifecycleChanged)
    }
    
    // 6. Serialize all dirty nodes
    n.closeTransactionSerializeNodes()
    
    // 7. Generate physical tasks (to task queues)
    n.closeTransactionUpdateComponentTasks(nextVersionedTransition)
    
    // Return mutations for persistence
    return n.mutation, nil
}
```

### Generating Mutations

The mutation tracks what changed:

```go
type NodesMutation struct {
    UpdatedNodes map[string]*persistencespb.ChasmNode  // path -> new node
    DeletedNodes map[string]struct{}                   // paths to delete
}
```

These are applied to the database atomically with the MutableState.

---

## Path Encoding

Paths are encoded for persistence and references:

```go
// Logical path: ["Callbacks", "cb-1"]
// Encoded path: "Callbacks/cb-1" (or similar encoding)

type NodePathEncoder interface {
    Encode(node *Node, path []string) (string, error)
    Decode(encodedPath string) ([]string, error)
}
```

Each node caches its encoded path to avoid recomputation:

```go
func (n *Node) getEncodedPath() (string, error) {
    if n.encodedPath != nil {
        return *n.encodedPath, nil
    }
    path := n.path()  // Walk up to root
    encodedPath, err := n.pathEncoder.Encode(n, path)
    if err == nil {
        n.encodedPath = &encodedPath
    }
    return encodedPath, err
}
```

---

## Example: Tree for a Workflow with Callbacks

```
Root (Component: Workflow)
├── serializedNode.Metadata.ComponentAttributes.TypeId = "workflow.workflow"
├── value = &Workflow{Callbacks: map["cb-1"]: Field[*Callback]{...}}
├── valueState = valueStateSynced
└── children:
    └── "Callbacks" (Collection Node)
        ├── serializedNode.Metadata.CollectionAttributes = {}
        └── children:
            └── "cb-1" (Component: Callback)
                ├── serializedNode.Metadata.ComponentAttributes.TypeId = "callback.callback"
                ├── serializedNode.Data = <serialized CallbackState>
                ├── value = &Callback{CallbackState: &callbackspb.CallbackState{...}}
                ├── valueState = valueStateSynced
                └── children: {} (no sub-fields)
```

### Persistence Representation

In the database, this is stored as a flat map:

```go
map[string]*ChasmNode{
    "": {  // Root
        Metadata: {ComponentAttributes: {TypeId: 12345}},
        Data: <serialized empty.Empty>,
    },
    "Callbacks": {  // Collection
        Metadata: {CollectionAttributes: {}},
    },
    "Callbacks/cb-1": {  // Callback component
        Metadata: {ComponentAttributes: {TypeId: 67890, PureTasks: [...], SideEffectTasks: [...]}},
        Data: <serialized CallbackState>,
    },
}
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Node Types** | Component, Data, Collection, Pointer |
| **Value State** | Tracks sync between Go value and proto (`NeedDeserialize` → `Synced` → `NeedSerialize` → `NeedSyncStructure`) |
| **Field Discovery** | Reflection-based iteration over `Field[T]`, `Map[K,V]`, `ParentPtr[T]`, and proto data fields |
| **Serialization** | Components serialize their proto data; sub-fields are separate nodes |
| **Transaction** | `CloseTransaction` syncs structure, serializes nodes, generates tasks, returns mutations |
| **Persistence** | Flat `map[encodedPath]*ChasmNode` in database |
| **Path Encoding** | `["Callbacks", "cb-1"]` → `"Callbacks/cb-1"` |

The tree provides the illusion of a hierarchical Go object graph while transparently handling persistence, lazy deserialization, dirty tracking, and transactional semantics.

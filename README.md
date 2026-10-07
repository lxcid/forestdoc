# ForestDoc

## TL;DR

**ForestDoc is a collaborative document model where the persistence and synchronization primitive is a Forest rather than a Block.**

A Forest contains any number of Nodes. Nodes form ordered parent/child relationships and may themselves reference another Forest.

```text
Forest
├── Node
├── Node
│   └── Forest
│       ├── Node
│       ├── Node
│       └── Node
└── Node
```

This gives ForestDoc a few useful properties at once:

- a one-node Forest can represent a block;
- a one-root Forest can represent a conventional tree;
- multiple roots naturally form a forest;
- Nodes can nest other Forests;
- one Forest can store many blocks/nodes;
- Forest boundaries can become CRDT, persistence, loading, and sharding boundaries;
- large Forests can be transparently split into sub-Forests;
- Yjs handles concurrent state changes;
- ForestDoc only needs to enforce structural invariants such as cycle freedom.

The central idea is:

> **Do not make Block the storage primitive. Make Forest the storage primitive.**

A block is simply a Forest containing one Node.

---

# 1. Core model

The fundamental objects are:

```ts
type ForestId = string
type NodeId = string
type Position = string

interface Forest {
  id: ForestId
  nodes: Map<NodeId, Node>
}

interface Node {
  id: NodeId

  parent: {
    id: NodeId | null
    pos: Position
  }

  type: string

  // optional typed components
  [component: string]: unknown
}
```

For example:

```json
{
  "id": "node_123",

  "parent": {
    "id": "node_100",
    "pos": "a1"
  },

  "type": "image",

  "image": {
    "src": "...",
    "alt": "..."
  }
}
```

The Forest itself does not need to explicitly maintain child arrays.

Children are derived from parent relationships.

```text
children(A) =
    nodes
    where parent.id == A
    ordered by parent.pos
```

Roots use exactly the same mechanism:

```text
roots =
    children(null)
```

Therefore:

```text
parent.id = null
```

means the Node is a root of the Forest.

Multiple Nodes can have a null parent, so multiple roots are completely normal.

That is why the primitive is called **Forest**, rather than Tree.

---

# 2. Why Forest instead of Block

A block-oriented storage system conceptually tends toward:

```text
Block A
  children → B, C

Block B

Block C
  children → D

Block D
```

The logical tree is reconstructed from individually stored blocks.

ForestDoc changes the persistence boundary:

```text
Forest {
  nodes: {
    A,
    B,
    C,
    D
  }
}
```

One Forest can therefore contain:

```text
1 Node
10 Nodes
1,000 Nodes
10,000 Nodes
```

without changing the logical document model.

This is an important distinction:

```text
Block storage

[A] [B] [C] [D] [E] [F]
```

versus:

```text
Forest storage

[A B C D] [E F G H I] [J K L]
```

ForestDoc decouples:

```text
UI granularity
      from
storage granularity
      from
synchronization granularity
```

A Node can look like an individual block to the user without requiring that Node to be an independently persisted CRDT document.

---

# 3. A Block is just a one-node Forest

A Forest containing:

```text
Forest
└── Node A
```

is effectively a block.

Therefore:

```text
Block ≈ Forest<NodeCount = 1>
```

But:

```text
Forest != Block
```

because a Forest may contain many locally related Nodes.

This means block storage becomes a special case rather than the fundamental architecture.

---

# 4. Nested Forests

A Node may reference another Forest.

Conceptually:

```ts
Node {
  id
  parent
  type

  forest?: {
    ref: ForestId
  }
}
```

For example:

```text
Forest A
├── Node A
├── Node B
│   └── Forest B
│       ├── Node D
│       ├── Node E
│       └── Node F
└── Node C
```

This distinction is important.

`D.parent` does NOT point to `B`.

Instead:

```text
B → Forest B
```

and inside Forest B:

```text
D.parent = null
E.parent = null
F.parent = ...
```

The Forest is a genuine structural boundary.

There are therefore two different relationships:

```text
Node → Node
```

for hierarchy inside a Forest, and:

```text
Node → Forest
```

for composition between Forests.

---

# 5. Conflict-free collaboration

ForestDoc is designed around Yjs.

Each Forest can naturally become a Yjs synchronization unit:

```text
Forest
   ↓
Y.Doc
   ↓
Y.Map<NodeId, Node>
```

Moving a Node does not require synchronously modifying child arrays.

A move is essentially:

```ts
node.parent = {
  id: newParent,
  pos: newPosition
}
```

Because the Node owns its parent relationship, a subtree move requires changing only the moved Node.

Its descendants remain attached to it.

Example:

```text
Before

A
└── B
    └── C


B.parent = D


After

A

D
└── B
    └── C
```

Only `B.parent` changed.

Yjs provides conflict-free convergence of the underlying replicated values.

ForestDoc provides the structural semantics above that CRDT state.

---

# 6. Ordering

`parent` contains both ownership and ordering:

```ts
parent: {
  id: NodeId | null
  pos: Position
}
```

For example:

```text
A.parent = { id: null, pos: "a0" }

B.parent = { id: A, pos: "a0" }
C.parent = { id: A, pos: "a1" }
D.parent = { id: A, pos: "a0V" }
```

A fractional indexing / LexoRank-like scheme can allow concurrent insertion without globally renumbering siblings.

Importantly, roots use the same mechanism:

```text
A.parent = { id: null, pos: "a0" }
B.parent = { id: null, pos: "a1" }
```

There is no separate root array.

`children(null)` simply returns the ordered roots.

---

# 7. Cycle repair

CRDT convergence alone does not guarantee a valid Forest.

Two clients might concurrently produce:

```text
A.parent = B
B.parent = A
```

Yjs can converge perfectly while the resulting parent graph contains a cycle.

ForestDoc therefore has one critical structural normalization operation:

> **Detect parent cycles and deterministically break them.**

For:

```text
A → B → C → A
```

every replica must deterministically select the same edge to remove or reparent.

Once cycles are removed, a graph where each Node has at most one parent naturally becomes a Forest.

This gives the core collaboration model:

```text
Yjs
 ↓
converged Node state
 ↓
deterministic normalization
 ↓
cycle-free parent graph
 ↓
Forest
```

The normalization algorithm must be deterministic so every replica derives the same valid structure from the same converged state.

This should receive extensive property/fuzz testing with concurrent moves across replicas.

---

# 8. Node types and components

ForestDoc separates structure from semantics.

Every Node has a primary `type`:

```json
{
  "type": "image",

  "image": {
    "src": "...",
    "alt": "..."
  }
}
```

`type` answers:

> **What is this Node primarily?**

Type-specific components carry its data.

For example:

```json
{
  "type": "task",

  "task": {
    "title": "Ship ForestDoc",
    "completed": false
  }
}
```

ForestDoc should maintain an external schema registry:

```ts
schemas = {
  image: ImageSchema,
  task: TaskSchema,
  position: PositionSchema,
  size: SizeSchema,
  forest: ForestSchema,
}
```

A type can require its corresponding component:

```text
type=image
    → image required
    → validate against ImageSchema
```

---

# 9. Components rather than subtype explosion

Nodes may contain additional components.

For example:

```json
{
  "id": "node_123",

  "parent": {
    "id": null,
    "pos": "a0"
  },

  "type": "image",

  "image": {
    "src": "cat.jpg"
  },

  "position": {
    "x": 100,
    "y": 200
  },

  "size": {
    "width": 400,
    "height": 300
  }
}
```

The same Image Node in a normal document may simply be:

```json
{
  "type": "image",

  "image": {
    "src": "cat.jpg"
  }
}
```

Therefore ForestDoc does not need types such as:

```text
DocumentImage
CanvasImage
ResizableCanvasImage
CanvasTask
DocumentTask
...
```

Instead:

```text
Node
 ├── type=image
 ├── image
 ├── position
 └── size
```

This resembles an entity-component model.

The important separation is:

```text
Forest        → collaboration/storage boundary

parent        → structural relationship

type          → primary semantic identity

components    → capabilities/data
```

---

# 10. No explicit `schemas` manifest initially

An earlier possibility was:

```json
{
  "schemas": {
    "image": true,
    "position": true
  },

  "image": {...},
  "position": {...}
}
```

This is probably unnecessary.

The presence of the component already expresses the capability:

```text
node.image exists
    → implements ImageSchema

node.position exists
    → implements PositionSchema
```

An explicit schema manifest would introduce duplicated state:

```text
schemas.image = true
image missing
```

or:

```text
schemas.image = false
image exists
```

ForestDoc should initially avoid this additional invariant.

Schema metadata/versioning can be introduced later if necessary.

---

# 11. Canvas becomes a view over components

A Canvas does not require a fundamentally different document model.

Given:

```json
{
  "type": "image",

  "image": {...},

  "position": {
    "x": 100,
    "y": 200
  }
}
```

a document renderer can ignore `position`.

A Canvas renderer can interpret it.

Likewise:

```json
{
  "type": "task",

  "task": {...},

  "position": {...}
}
```

can render as a spatial Task.

Therefore the same Forest can support multiple views without changing its structural primitive.

---

# 12. ProseMirror

ProseMirror should be treated as an editor for appropriate content, not as ForestDoc's fundamental document schema.

Conceptually:

```text
ForestDoc
    ↓
Node
    ↓
type / components
    ↓
appropriate editor
```

A rich-text component may use ProseMirror:

```text
Node
└── richtext
    └── ProseMirror/Yjs
```

An Image Node may not require ProseMirror at all.

A Task may use its own structured schema.

This keeps ForestDoc independent from any specific editor implementation.

Hocuspocus can provide Yjs collaboration infrastructure while ProseMirror handles rich-text editing where appropriate.

---

# 13. Forest as the synchronization boundary

One particularly useful property is that:

```text
Forest ≈ Y.Doc
```

can become a natural implementation.

This avoids both extremes.

### Entire document as one CRDT

```text
[                    huge document                    ]
```

Simple, but eventually expensive to load, synchronize and render.

### Every Block as its own CRDT

```text
[A] [B] [C] [D] [E] [F] [G] ...
```

Very granular, but potentially creates huge numbers of tiny independently managed objects.

### ForestDoc

```text
[A B C D E] [F G H] [I J K L M N] ...
```

The system chooses useful synchronization boundaries.

---

# 14. Forest sharding

This becomes especially powerful for extremely large documents.

Suppose one Forest eventually contains:

```text
1,000,000 Nodes
```

It may become undesirable to load or synchronize the entire structure as one CRDT.

ForestDoc does not need a new representation.

The Forest can simply be split into nested Forests.

Conceptually:

```text
Forest
│
├── Node → SubForest 0001
│           Nodes 1–1,000
│
├── Node → SubForest 0002
│           Nodes 1,001–2,000
│
├── Node → SubForest 0003
│           Nodes 2,001–3,000
│
│   ...
│
└── Node → SubForest 1000
            Nodes 999,001–1,000,000
```

Now the root Forest is small.

It primarily describes how to reach its child Forests.

Only relevant sub-Forests need to be loaded.

```text
               Root Forest
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Forest 1      Forest 2     Forest 3
   1–1000       1001–2000    2001–3000
       │
       ▼
    loaded

others remain unloaded
```

This makes **Forest nesting a natural sharding primitive**.

---

# 15. Semantic nesting and physical sharding use the same abstraction

This may be one of ForestDoc's strongest properties.

A nested Forest can exist because the user explicitly created a semantic boundary:

```text
Project
└── Design Document
    └── Forest
```

But a nested Forest can also exist because the system decided a structure became too large:

```text
Huge Forest
   ↓
automatic split
   ↓
Forest A
Forest B
Forest C
```

The underlying primitive remains identical.

Therefore ForestDoc doesn't need a completely separate "large document architecture."

The same recursive abstraction scales downward and upward.

```text
one Node
    ↓
Forest

many Nodes
    ↓
Forest

huge number of Nodes
    ↓
Forest of Forests

extremely huge structure
    ↓
Forest of Forests of Forests
```

This is effectively a recursive storage hierarchy.

---

# 16. Important distinction: logical vs physical Forests

It may eventually be useful to distinguish:

```text
Semantic Forest
```

created because the structure actually represents an independently meaningful document/container,

from:

```text
Shard Forest
```

created automatically as an implementation detail.

They can use the exact same underlying representation.

The distinction only determines whether the boundary should be visible to consumers.

For example:

```text
User sees:

Document
├── 1
├── 2
├── ...
└── 1,000,000
```

while storage sees:

```text
Root Forest
├── shard 1
├── shard 2
├── shard 3
...
└── shard 1000
```

The renderer/API can flatten physical shard boundaries when traversing the logical document.

This means the system can rebalance storage without changing the user's document model.

That is a significant architectural advantage.

---

# 17. Automatic sharding

Eventually ForestDoc could have policies such as:

```text
if Forest.nodeCount > threshold
    split Forest
```

But node count doesn't have to be the only signal.

A Forest could split based on:

```text
serialized bytes
CRDT state size
update frequency
working-set locality
render cost
memory pressure
access patterns
```

For example:

```text
Forest
  50,000 nodes
  80 MB Yjs state
  only 500 nodes normally visible
```

could automatically become:

```text
Root
├── Forest A
├── Forest B
├── Forest C
└── Forest D
```

Each child becomes independently loadable and synchronizable.

This creates the possibility of **adaptive document sharding** without changing the public document abstraction.

---

# 18. Lazy loading

Nested Forests naturally provide lazy loading.

Opening:

```text
Forest A
```

does not necessarily require loading:

```text
Forest A
└── Node
    └── Forest B
        └── Node
            └── Forest C
```

Initially:

```text
load Forest A
```

When the user expands or navigates into the nested structure:

```text
load Forest B
```

and only later:

```text
load Forest C
```

This makes very large documents practical while retaining a simple recursive programming model.

---

# 19. Addressing

Because Nodes live inside Forests, the canonical address should likely include both:

```text
ForestId + NodeId
```

For example:

```text
forest_123/node_456
```

or structurally:

```ts
interface NodeRef {
  forest: ForestId
  node: NodeId
}
```

This avoids requiring Node IDs to encode storage location and keeps Node identity separate from Forest identity.

It also makes cross-Forest references explicit.

---

# 20. Ownership versus references

A useful future distinction is:

```text
Node → nested Forest
```

meaning structural ownership,

versus:

```text
Node → reference to Forest
```

meaning ordinary linking.

Ordinary references may freely form graphs and cycles.

Ownership should remain acyclic.

For example:

```text
Forest A
  Node → owns Forest B

Forest B
  Node → references Forest A
```

is fine.

But:

```text
Forest A owns Forest B
Forest B owns Forest A
```

should not be allowed.

This keeps the actual document hierarchy acyclic without preventing rich graph relationships between content.

---

# 21. Core invariants

ForestDoc should attempt to keep its mandatory invariants extremely small.

Within a Forest:

1

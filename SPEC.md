# The Substrates Specification

**Version 3.0.5**
**Copyright © 2025–2026 William David Louth / Humainary**

---

## 1. Purpose

Substrates is a specification for deterministic signal circulation infrastructure. It defines a set
of structural primitives, behavioral contracts, and lifecycle semantics for constructing
computational networks in which typed values flow through governed topologies with precise ordering
guarantees.

The specification is independent of any programming language, runtime, or transport mechanism. A
conformant implementation may be realized as an in-process library, a networked service, or any
combination thereof. The structural and behavioral contracts remain identical across all
projections.

### 1.1 Conformance Language

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "MAY", and "OPTIONAL" in this specification are to be interpreted as described in RFC
2119 and RFC 8174 when, and only when, they appear in all capitals as shown here.

### 1.2 Definitions

This specification uses the following abstract terms to remain independent of any specific runtime
model:

- **Execution context**: An abstract unit of sequential execution. Depending on the runtime, an
  execution context may be a thread, a goroutine, a coroutine, a task, an actor, or a turn of an
  event loop.
- **Circuit context**: The single sequential execution context owned by a circuit. All circuit
  processing occurs within this context.
- **Caller context**: Any execution context that invokes operations on substrate components from
  outside the circuit context.
- **Happens-before**: A partial ordering on operations. If operation A happens-before operation B,
  then the effects of A are observable by B and A is ordered before B.
- **Suspend**: To yield control of the current execution context until a condition is met. The
  mechanism is runtime-dependent — blocking a thread, yielding a coroutine, awaiting a future, or
  any equivalent.
- **Canonical identity**: Two references designate the same abstract entity. Within a single address
  space, this is typically reference equality (pointer comparison). Across address spaces (networked
  projections, serialization boundaries), an implementation MUST define an equivalence protocol that
  preserves the identity guarantees of this specification. The underlying mechanism — pointer
  equality, interned handle comparison, content-addressed lookup, or distributed consensus — is not
  specified. The protocol MUST satisfy:
    - **Observable**: Identity is testable through the projection's standard equality mechanism.
      Consuming code MUST NOT require special APIs to check canonical identity.
    - **Stable**: Two canonically identical references remain so for the lifetime of the entity's
      owning container (e.g., a name remains canonically identical for the lifetime of its cortex).
    - **Documented**: The projection declares its equivalence protocol as part of its conformance
      statement (see Appendix A).
- **Absent value**: A sentinel indicating "no value produced." Used in mapping and transformation
  operations to signal that an emission should be dropped. Each language projection maps this to its
  idiomatic representation. The abstract signatures in this specification use the `?` suffix to
  denote operations that may return or produce an absent value. Non-normative examples of idiomatic
  mappings: `null` in Java/C#, `None` in Python, `Option::None` in Rust, a pointer/interface nil or
  a sum type in Go, `undefined` in JavaScript, `Nothing` in Haskell.

## 2. Design Principles

Three governing principles shape the specification.

**Determinism over throughput.** Every emission is processed in a defined order. Earlier emissions
complete before later emissions begin. All observers see emissions in the same sequence. This
property enables replay, testing, digital twin synchronization, and reasoning about system behavior.

**Composition over inheritance.** The specification defines small, focused primitives that compose
into complex behaviors. Circuits contain conduits. Conduits pool named pipes. Subscribers wire
downstream pipes to named pipes. Fiber operators chain into per-emission recipes; Flow composes
type-changing stages, stateful folds, and output-side recipes. Each primitive has a single, clear
responsibility.

**Explicitness over magic.** Data flow is visible and traceable. There is no reflection-based
wiring, no implicit dependency injection, no hidden framework machinery. Every connection between
components is established through explicit operations defined in this specification.

## 3. Structural Overview

The topology of a Substrates system is a containment hierarchy.

A **Cortex** is the root. It is the entry point and factory for all runtime components. A cortex
creates circuits, names, scopes, and other foundational objects.

A **Circuit** is an execution engine owned by a cortex. Each circuit maintains a single sequential
execution context and an ordered admission path for pending work. All emission processing within a
circuit occurs within that context, sequentially.

A **Conduit** is a pipe factory owned by a circuit. A conduit creates and pools pipes by name — the
same name always resolves to the same pipe. Derived views over the pipe pool are created via
`pool(fn)`, caching the transformation result per name. A conduit is also a source — it supports
subscriptions that enable dynamic discovery of it's named pipes.

A **Pipe** is an emission carrier. It accepts a typed value and delivers it to its target. The
delivery path depends on how the pipe was created, but observable dispatch is governed by the owning
circuit's ordering rules. Pipes are the primary mechanism for introducing values into the system.

A **Receptor** is a callback that receives emissions. Receptors are the consumption-side counterpart
to pipes.

A **Lookup** is the abstract base for name-indexed retrieval. Two concrete forms extend Lookup:
**Pool**, which adds composable derived views and is implemented by conduits; and **Bank**, which
adds ownership semantics — closing a bank closes all resources it has materialized.

A **Bank** is a closeable name-indexed holder for same-kind named resources. Banks are created from
a circuit and provide stable shared references to named conduits with uniform emission type and
routing. Unlike pool views derived from a conduit, a bank owns its conduits: closing the bank closes
the conduits it has materialized.

A **Cell** is a circuit-owned, initialized, single-slot state holder. A cell is created with a
non-null seed value, exposes one update pipe and one read accessor, and is never empty. Emissions
sent through the pipe are processed on the owning circuit; each accepted emission becomes the cell's
published value. Cells are the specified end-of-pipeline mechanism for publishing the latest value
computed by a Fiber or Flow recipe to readers that need a stable handle rather than an event stream.

The containment hierarchy is:

```text
Cortex
  └── Circuit
        └── Conduit
```

Each level governs the level beneath it. Closing a circuit closes its conduits.

## 4. Identity System

Every component in a Substrates system carries identity through a subject. The identity system
comprises three elements: names, identifiers, and subjects.

### 4.1 Name

A **Name** is a hierarchical, dot-separated sequence of string segments. Names are interned —
structurally equivalent names MUST resolve to the same canonical entity. Name comparison is
canonical identity (see §1.2), providing constant-time lookup.

Names follow these rules:

- A name consists of one or more non-empty string segments.
- Segments are separated by the dot character (`.`).
- A name MUST NOT begin or end with a dot.
- Consecutive dots (empty segments) are prohibited.
- Names are immutable once created.
- A name MAY be extended by appending additional segments, producing a new interned name.

The minimum supported depth is 16 segments. A conformant implementation MUST accept the creation of
any name up to that depth, whether it is constructed in a single operation or reached by successive
extension. An implementation MAY support deeper names, and one that imposes a maximum MUST document
it; exceeding that maximum MUST signal an illegal-argument error (§15.1) from the creation or
extension operation rather than truncating the name or returning a shallower one. Portable code MUST
NOT assume a depth beyond the guaranteed minimum.

The interning guarantee is: given two name creation operations with identical segment sequences,
both operations MUST yield canonically identical names (§1.2). This identity property is
foundational — it enables pooling in conduits and channels, where the same name always routes to the
same component.

Names form an extent (see §4.4). The name `com.example.service` has `service` as its terminal part,
`com.example` as its enclosure, and `com` as its extremity (root).

### 4.2 Identifier

An **Identifier** (Id) is an opaque, runtime-unique token assigned to each subject. Two subjects
with the same name MAY have different identifiers if they are distinct instances. Identifier
comparison is canonical identity.

Identifiers MUST be unique within a single runtime instance. They are not guaranteed unique across
process restarts or distributed deployments.

### 4.3 Subject

A **Subject** is the identity record of a substrate component. Every substrate component has a
subject — this includes every cortex, current, pipe, scope, and resource (and therefore every
subscriber, subscription, circuit, and source — including the source subtype conduit). See the
capability matrix in §16.2 for the authoritative inventory. A subject carries:

- **Identifier**: A unique Id distinguishing this instance.
- **Name**: A hierarchical Name locating this component in the namespace.
- **State**: An immutable collection of typed slots (see §8).
- **Type**: The kind of substrate this subject represents (circuit, conduit, pipe, etc.).

Subjects form an extent (see §4.4). A pipe's enclosure is the subject of the source that owns it —
typically a conduit, but any Source subtype. The source's enclosure is in turn the circuit's
subject, which has the cortex's subject as its enclosure. This produces a fully-qualified path for
every component. The enclosure relation is the mechanism by which a subscriber identifies the origin
of an emission: a `Capture` carries the emitting pipe's subject, and walking one step up the extent
chain yields the source subject without requiring the source type to be exposed on the emission
path. A `Capture` additionally carries the subject of the emitting context (§11.1) — a reference to
that `Current` rather than the live token, keeping the capture an out-of-execution-scope record.

Subject comparison is canonical identity.

### 4.4 Extent

An **Extent** is a hierarchically nested structure of enclosed whole-parts. It provides depth
calculation and path rendering over any containment hierarchy.

An extent has:

- **Part**: A string representation of this level of the hierarchy.
- **Enclosure**: An optional reference to the enclosing (parent) extent. The root extent has no
  enclosure.
- **Extremity**: The outermost (root) extent, found by traversing enclosures to the end.
- **Depth**: The number of levels from this extent to the root, inclusive.

This specification does not define a maximum extent depth. An implementation MAY limit how deeply a
given kind of extent nests and MUST document any limit it imposes; where a minimum is guaranteed, it
is stated with the kind — see §4.1 for names. Portable code MUST NOT assume that operations over an
extent hierarchy remain well defined beyond an implementation's documented depth. This applies to
every operation reached through the hierarchy, not only to traversal — comparison and cascading
lifecycle operations such as scope closure are included.

Traversal proceeds from the current extent toward the root (right to left) via the enclosure chain.
Path rendering concatenates parts with a separator character. The default separator for names is the
dot (`.`). The default separator for other extents is the slash (`/`).

Projections MAY provide additional traversal, folding, iteration, and comparison operations over
extents as non-normative conveniences. The required operations (§16.3) define the minimum portable
surface: `part`, `enclosure`, `extremity`, `depth`, and `path`.

### 4.5 Type

A **Type** is an abstract classifier that identifies the kind of entity a component or value
represents. Types appear in two roles:

- **Substrate type**: Returned by `Subject.type()`, classifying the substrate component. A
  conformant implementation MUST define a distinct type value for each required type listed in
  §16.2. Not all required types carry subjects — types such as Receptor, Fiber, Flow, Capture, and
  Registrar (in its callback role) are abstract protocols or structural types that do not expose
  `subject()`. Resource *is* subject-bearing: being a resource is precisely what confers identity
  alongside lifecycle, so every concrete type marked Resource ✓ in §16.2 exposes `subject()`.
  Resource is an abstract role and is not itself instantiated; concrete resource-bearing types
  expose
  `subject()` per §16.2. The type classifier is defined for all required types regardless;
  `Subject.type()` returns it only for types that have subjects. See §16.2 for the authoritative
  capability matrix.
- **Slot value type**: Returned by `Slot.type()`, classifying the value held by a slot. A conformant
  implementation MUST define a distinct type value for each slot value type listed in §8.3.

Type equality is value equality — two type values representing the same classifier MUST compare as
equal regardless of how they were obtained. The equality mechanism is projection-dependent: a
language binding MAY use interned tokens, enumeration members, nominal type metadata, string tags,
or any other representation, provided the equality guarantee holds.

Types are not required to be serializable, stable across process boundaries, or suitable for
persistent storage. Implementations that require cross-boundary type stability SHOULD define a
projection-specific serialization protocol.

Types are opaque to the specification — the only required operation is equality comparison and the
`Subject.type()` / `Slot.type()` accessors. A language binding MAY expose additional capabilities
(display names, reflection metadata, hierarchical relationships) as non-normative extensions.

## 5. Execution Model

The execution model is the foundation of the specification. Every behavioral guarantee derives from
it.

### 5.1 Circuit Context

Every circuit owns exactly one sequential execution context — the **circuit context**. All
emissions, fiber/flow operations, subscriber callbacks, and state transitions within a circuit
execute exclusively within the circuit context. This is the **circuit-context confinement**
guarantee.

Consequences of this guarantee:

- State accessed only from the circuit context requires no synchronization.
- Only one operation executes at a time per circuit.
- Race conditions within a circuit are structurally impossible.
- Deterministic replay is achievable by replaying the circuit's admitted operation sequence (see
  §5.6). That sequence includes all operations submitted to the circuit — emissions, subscription
  registrations, unsubscriptions, and close operations — not only emissions. Replaying emissions
  alone is insufficient because subscription operations affect pipeline topology (§7.6), which
  determines how subsequent emissions are routed.

The circuit context is an abstract concept. In a threaded runtime, it may be a dedicated thread. In
an async runtime, it may be a pinned task or coroutine. In an event-loop runtime, it may be a
sequence of non-preemptive turns. The implementation mechanism is not specified — only the
sequential, exclusive execution guarantee is REQUIRED.

### 5.2 Caller Context and Circuit Context

Operations in a Substrates system split between two execution contexts.

The **caller context** is any execution context that invokes operations on substrate components —
emitting values, creating conduits, subscribing to sources. Caller contexts submit work for circuit
processing and return immediately. They do not execute circuit logic.

The **circuit context** is the single sequential execution context owned by a circuit. It processes
accepted work sequentially. All fiber/flow pipelines, subscriber callbacks, and emission dispatches
execute here.

The performance principle follows from this split: the circuit context is the bottleneck. Work
SHOULD be shifted to caller contexts where possible. Expensive computation, serialization, and
preparation SHOULD occur before emission. The circuit context SHOULD handle only routing, filtering,
and lightweight state updates.

### 5.3 Logical Ingress and Transit Ordering

The specification defines two logical classes of queued work. These are ordering roles, not a
required implementation data structure.

**Ingress work** is work accepted from caller contexts. This includes emissions and other queued
operations such as subscription registration, subscription close, and resource cleanup work.

**Transit work** is work generated while the circuit context is already processing. When processing
an ingress item triggers further emissions within the circuit context, those cascading emissions are
transit work.

Processing priority: transit work MUST drain completely before the circuit context returns to the
next ingress item. This produces several properties:

- **Causality preservation**: All effects of an emission complete before the next external emission
  begins.
- **Stack safety**: Cascading emissions are staged rather than recursively invoked. Deeply cascading
  chains do not overflow the call stack.
- **Atomicity**: A cascading chain appears atomic to external observers.
- **Cyclic safety**: Feedback loops and recurrent topologies operate correctly. The admission
  boundary breaks synchronous call chains that would otherwise produce infinite recursion.

When transit work itself emits, those emissions are sequenced behind transit work already accepted
for the current cascading chain. The model is always iterative, never recursive.

Implementations MAY realize this model with separate queues, a single mailbox with priority markers,
local slots, actor turns, coroutines, or any other mechanism. The mechanism is not specified; the
observable ingress/transit ordering is.

### 5.4 Memory Model

This section defines the ordering and visibility guarantees of the specification in abstract terms,
independent of any language-specific memory model.

#### 5.4.1 Happens-Before Relations

The specification defines the following happens-before relations. These relations are the formal
basis for all ordering and visibility guarantees in the system.

1. **Submit → Process**: A queued operation accepted from a caller context happens-before the
   processing of that operation within the circuit context.
2. **Process → Process**: Completion of one circuit-context operation happens-before the start of
   the next operation processed by that circuit context.
3. **Transit → Ingress**: Completion of all transit work accepted for the current cascading chain
   happens-before the next ingress item begins processing.
4. **Process → Await**: Completion of all work accepted before an await barrier happens-before the
   return of that await call to the caller context.

These relations are transitive: if A happens-before B and B happens-before C, then A happens-before
C.

#### 5.4.2 Visibility Guarantees

State modifications made within the circuit context during emission processing MUST be visible to:

- Subsequent emissions processed within the same circuit context (by relation 2).
- Caller contexts that complete an await operation accepted after the modification (by relation 4).

State modifications made by a caller context before a queued operation is accepted MUST be visible
to the circuit context when processing that operation (by relation 1).

No other visibility guarantees are made. In particular, state modifications made within the circuit
context are NOT guaranteed visible to caller contexts that have not performed an await.

Implementations MUST provide these visibility guarantees through whatever synchronization mechanism
is appropriate for the target runtime — memory barriers, message passing, channel-based
communication, or runtime-specific ordering guarantees. The mechanism is not specified; the
guarantees are.

#### 5.4.3 Deterministic Ordering

Emissions MUST be observed in strict admission order. Earlier emissions MUST complete before later
ones begin. All observers within the circuit context MUST see emissions in the same sequence.

The admission order is determined by the implementation's concurrency-safe ingress admission
mechanism. When multiple caller contexts emit concurrently, the relative ordering of their emissions
is determined by that mechanism and is not specified by this specification. Determinism is
guaranteed within a single caller context and for the total observed sequence — every run of the
circuit context sees a single, consistent order — but that order may differ across runs when
multiple callers emit concurrently.

**Scope of determinism**: This contract covers admission and dispatch ordering — the sequence in
which emissions are observed by receptors, fiber/flow stages, and subscribers. It does not extend to
value-level outcomes produced by nondeterministic user-supplied callbacks or operators. In
particular, probabilistic operators such as `Fiber.chance(probability)` (§6.2.3) are explicitly
allowed to produce different filter decisions across runs; the specification places no
reproducibility requirement on such operators. Implementations MAY offer seeded pseudo-random
variants as non-normative extensions.

### 5.5 Await

The **await** operation is a barrier, not an idleness snapshot. It suspends the calling execution
context until every circuit operation accepted before the await call has completed. It does not
guarantee that the circuit has no work when it returns: operations accepted concurrently with or
after the await call are outside that await's barrier.

Await establishes a happens-before relationship with all work accepted before the barrier (see
§5.4.1, relation 4).

Await is the primary coordination mechanism between caller contexts and the circuit context. After
await returns, all operations accepted before the await call — emissions, subscription
registrations, unsubscriptions, and any other queued work — have completed, and their effects are
visible to the caller.

**Caller-side return is not effect-at-caller**: Queued operations (`emit`, `subscribe`,
`Subscription.close`, and any other operation that submits work per §5.1) return to the caller as
soon as the operation has been accepted for circuit processing. Their effect is not guaranteed to
have taken place at the moment of return. A caller that needs to observe the effect of a queued
operation — for example, to verify that a subscription has been installed, or that an emission has
been dispatched — MUST call `await` to synchronize. Callers MUST NOT assume that the effect of a
queued operation is visible before `await` returns.

Await MUST NOT be called from within the circuit context. An implementation that detects this
violation MUST signal an illegal context use error (§15.1). If detection is not feasible, the
behavior is undefined (deadlock is the likely outcome).

Multiple caller contexts MAY call await concurrently; each suspends independently. After circuit
closure, await MUST return immediately.

The suspension mechanism is runtime-dependent. In a threaded runtime, await may block the calling
thread. In an async runtime, it may yield the current task. In a distributed runtime, it may involve
a round-trip. The mechanism is not specified — only the synchronization and visibility guarantees.

### 5.6 Admission Capacity and Control

The emit operation MUST NOT perform downstream circuit work on the caller context. The caller-side
submission path MUST remain bounded and responsive. The emit operation MUST NOT discard the emission
while the circuit is open. Together these properties imply that the ingress admission path cannot
rely on a fixed capacity limit that rejects or drops open-circuit emissions as normal operation.

This is a deliberate consequence of the governing principle "determinism over throughput" (§2). An
admission path that drops accepted work discards emissions, violating the total delivery guarantee
that underlies deterministic replay (§5.1). Implementations MAY use bounded internal buffers,
batching, or backpressure in non-caller-visible layers, provided accepted open-circuit emissions are
not lost.

Admission control is addressed at the pipeline level. `Fiber.limit()` (§6.2.3) caps the number of
emissions processed by a given pipeline stage. `Fiber.every()` and `Fiber.chance()` (§6.2.3)
provide interval-based and probabilistic rate filtering respectively. Both execute within the
circuit context after admission and therefore do not alter ingress ordering. These are the specified
mechanisms for filtering emission volume during circuit-context processing. Caller-side
synchronization, when required, is expressed through `Circuit.await()` (§5.5).

### 5.7 Circuit Context Identity and Pulse

A circuit exposes the `Current` identity token of its circuit context. `Circuit.current()` MUST
return a stable Current for the lifetime of the circuit. Comparing `Circuit.current()` with
`Cortex.current()` lets callers detect whether they are currently executing on that circuit context
without invoking an operation that is illegal from the circuit context, such as `await` or `pulse`.

A circuit also exposes a **pulse** diagnostic operation, distinct from the `Fiber.pulse` operator
(§6.2.3). A pulse submits a no-op probe through the circuit's caller-side admission path and returns
a timing snapshot if the probe traverses the circuit. The probe MUST preserve normal ordering: it
waits behind earlier ingress work and behind any transit work caused by earlier jobs (§5.3). The
probe MUST NOT dispatch user emissions, mutate topology, or alter circuit state beyond any
implementation-internal diagnostic bookkeeping.

The pulse snapshot records four monotonic timestamp values:

- **Start**: captured on the caller context immediately before the probe is submitted.
- **Enqueued**: captured on the caller context immediately after submission returns.
- **Dequeued**: captured on the circuit context when the probe begins executing.
- **Stop**: captured on the caller context immediately before the pulse operation returns.

The timestamp unit is projection-dependent and MUST be documented. Only differences between
timestamp values are meaningful.

Pulse lifecycle behavior:

- **Active / Closing**: The probe enters the caller-side admission path and returns a present pulse
  if it is processed. During Closing, a present pulse does not imply the circuit remains healthy
  beyond the probe; it only proves the probe traversed the circuit.
- **Closed**: Once the circuit context has terminated and the probe cannot traverse the circuit, the
  operation returns absence rather than a degenerate pulse.

Pulse MUST NOT be called from within the circuit context. An implementation that detects this
violation MUST signal an illegal context use error (§15.1).

### 5.8 Circuit Processing Time

Some operators are time-aware — most visibly the time-bounded `window(Duration, capacity)` operator
(§6.2.3), and any provider-published capture measure (§11.1). The processing time they observe is
defined as **stimulus time** with the following properties:

- **Monotonic and process-relative**: It derives from a monotonic clock with an implementation-fixed
  origin. It is not a wall-clock instant and is not comparable across processes or machines; only
  differences are meaningful. The unit is projection-dependent and MUST be documented.
- **Stimulus-scoped, not per-step**: A single ingress item (§5.3) and the entire transit cascade it
  triggers share one processing-time reading. Time advances on each ingress item — which includes
  external emissions, ticker fires, and cross-circuit arrivals (all admitted as ingress) — but does
  **not** advance across the internal transit hops of one cascade. Internal cause-effect within a
  cascade is therefore treated as a single causal moment, co-temporal.
- **Lazily established**: An implementation is not required to read the clock for ingress items that
  no time-aware operator observes. It MUST, however, ensure that all time-aware observations within
  one ingress chain see the same reading.

This model favours causal consistency and aggregation (turning many co-temporal signals into a
status) over sub-stimulus timing resolution, which at circuit processing speeds is dominated by
noise. Implementations that need intra-cascade latency MUST use the pulse diagnostic (§5.7), not the
processing-time reading.

## 6. Emission Path

The emission path describes how a value enters the system, is processed, and reaches consumers.

### 6.1 Pipe

A **Pipe** is the emission mechanism. It accepts a typed value through an `emit` operation and
delivers it to its target. The delivery path depends on how the pipe was created — conduit pipes
(created via `conduit.get(name)`) submit work to the owning circuit, registered downstream pipes
execute within the circuit context during dispatch, and circuit-created pipes submit to their owning
circuit before dispatching to a target pipe or receptor.

Pipes are created by the framework, not by user code. They are obtained from channels, circuits, or
registration callbacks.

Pipes have no inherent concurrency contract — their execution context depends on how they were
created:

- Pipes obtained from channels submit emissions to the owning circuit.
- Pipes created by circuits with a target pipe or receptor dispatch through the owning circuit.
- Registered downstream pipes execute within the circuit context when the named pipe dispatches.

Implementations MAY use different internal queues, local slots, batching, or fast paths for
caller-context submissions and circuit-context cascading submissions. These optimizations MUST
preserve the public ordering, non-recursion, failure-isolation, and circuit-context confinement
contracts defined in this specification.

The emit operation MUST NOT be called with an absent value (see §1.2). Implementations MUST signal
an absence violation error (see §15.2) if an absent value is detected.

#### 6.1.1 Emission Safety Contract

A value emitted through a `Pipe` crosses a context boundary: it is admitted on the caller's thread
and consumed on the owning circuit's worker thread, possibly after queuing. To preserve the
single-threaded execution discipline the circuit guarantees, emitted values MUST satisfy at least
one of these conditions:

- The value is immutable.
- The value is effectively immutable for the lifetime of all possible observers.
- The value is a managed substrate handle whose mutation is governed by Substrates (such as `Pin`
  — see §11.6 — which carries an owner-context guard that prevents dereferencing outside the circuit
  that owns it).

Mutable application objects MUST NOT be emitted directly when they can be mutated after admission or
observed concurrently by another circuit. If mutable state must flow through a topology, it SHOULD
flow as a managed handle (e.g. `Pin`) or be copied/projected into an immutable snapshot before
emission. This rule tightens — but does not replace — the existing safe-publication language for
state holders (§11.2 Cell, §11.5 Port) that warns callers about mutating already published
references.

Implementations are not required to prove deep immutability of emitted values. Language projections
SHOULD document unsafe mutable categories and MAY provide diagnostic checks for obvious cases such
as arrays, mutable collections, mutable buffers, and atomic wrappers.

### 6.2 Fiber and Flow

Two complementary types carry the emission-processing vocabulary:

- A **Fiber\<E\>** is a same-type per-emission recipe — a chain of stateless and stateful operators
  over a single type `E`. Fibers hold every type-preserving per-emission operator (guard, diff,
  peek, reduce, every, chance, and the rest).
- A **Flow\<I, O\>** is the left-to-right composition surface. Flows carry the type-changing
  operator `map`, the type-changing stateful fold `scan`, the bounded temporal rolling window
  operator `window`, the run-length operators `run` and `change`, output-side Fiber attachment
  (`fiber` or `fiber(factory)`), and Flow composition (`flow` or `flow(factory)`). A Flow is
  optionally type-changing: `Flow<E, E>` is a valid same-type Flow.

Both Fibers and Flows are standalone immutable values obtained from cortex factories (§16.3). They
may be retained, shared across contexts, and materialized multiple times. Per-materialization state
for stateful operators owned by Fiber or Flow values is allocated at materialization, not at
configuration.

Flows may attach a **subject-aware fiber factory** at the output side. The factory is invoked once
per `Flow.pipe(pipe)` attachment with the materialized pipe's subject and returns the same-type
Fiber recipe to use for that attachment. The returned Fiber is then materialized like any other
attached Fiber, with independent per-attachment state. A factory MUST return a non-absent Fiber and
MUST NOT rely on mutable state shared across invocations. Factory invocation is configuration-time
work: if the factory signals an error or returns an absent value, the `pipe` attachment fails
synchronously; emission-time errors from the materialized Fiber remain subject to the external
callback isolation rule in §15.4.

Flows may also compose with a **subject-aware flow factory** — the type-changing analogue of the
fiber factory above. The factory is invoked once per `Flow.pipe(pipe)` attachment with the
materialized pipe's subject and returns the Flow segment to compose at the output side. The returned
Flow is then materialized like any other composed Flow, with independent framework-owned
per-attachment state for its operators. A factory MUST return a non-absent Flow and MUST NOT return
a recipe whose internal mutable state is shared across factory invocations: the framework
materializes whatever Flow the factory returns without copying or otherwise enforcing freshness, so
a factory that returns a recipe containing mutable state aliased across attachments will alias
execution state across all materializations. Factory invocation is configuration-time work: if the
factory signals an error or returns an absent value, the `pipe` attachment fails synchronously. The
flow factory is the canonical mechanism for per-subject configuration of any downstream Flow stage —
including stateful operators like Scan and Window — and projections SHOULD NOT add per-operator
subject-aware overloads when this composition primitive suffices.

All Fiber and Flow stages execute within the circuit context that materialized them. They process
emissions sequentially and deterministically. Exceptions raised by emission-time operator functions
(predicates, comparators, binary operators, mappers, scan step / emit functions, peek receptors,
fire predicates, etc.) are subject to the external callback isolation rule in §15.4 — the circuit's
dispatch loop MUST NOT be affected, and a throwing operator is treated as having dropped the
emission for that receptor chain.

Client-supplied emission-time operator functions — predicates, mappers, comparators, binary
operators, key functions, fire predicates, scan `step` / `emit` functions, peek receptors, and any
other callable invoked while processing an emission — MUST NOT carry shared mutable
emission-processing state. A Fiber or Flow is a value that MAY be materialized across multiple
conduits and multiple circuits, and these function instances are captured by the value and shared
across every materialization. Mutable state captured inside an emission-time function is therefore
shared across materializations — including across circuits running on different threads — and
subject to data races. The framework owns any state that a stateful operator logically requires:
Reduce holds the accumulator per materialization, Relate holds the previous input per
materialization, Diff holds the last-emitted value per materialization, Scan holds its state slot
per materialization, and so on.

Attachment-time callables — subject-aware Fiber factories, subject-aware Flow factories, and Scan
initial-state Suppliers — MAY allocate fresh mutable state or read immutable / thread-safe
configuration. They MUST NOT use captured mutable state as shared per-emission state across
materializations. Implementations MUST NOT require client functions to carry operator state of their
own; clients MUST put per-emission state in framework-owned state slots such as Reduce's accumulator
or Scan's supplied state.

#### 6.2.1 Fiber and Flow Typing

A Fiber is parameterized by a single emission type `E`. Every Fiber operator preserves that type: a
`Fiber<E>` operator returns a `Fiber<E>`. An **identity fiber** is a fiber with no operators — it
passes every value through unchanged. Identity fibers are obtained from the cortex factory (§16.3);
operators are chained onto an identity fiber to build a per-emission recipe.

A Flow is parameterized by two types: an **input type** `I` (the type of values entering the
pipeline from upstream) and an **output type** `O` (the type delivered downstream). The `map`
operator is **type-changing**: it appends a transformation downstream and returns a `Flow<I, P>`
where `P` is the new downstream-facing type. The `scan` operator is **type-changing and stateful**:
it appends a running fold whose state type `S` is independent of input `O`; projected scan returns
`Flow<I, P>`, while state-as-output scan returns `Flow<I, S>` (see §6.2.3 for semantics). The
`window` operator is **type-changing and stateful**: it appends a bounded rolling buffer over
surviving `O` values and returns `Flow<I, Window<O>>`. The `fiber`
operator appends a same-type Fiber at the output side, returning a `Flow<I, O>`. The `flow`
operator composes with another Flow end-to-end, producing a `Flow<I, P>` when the composed flow's
output type is `P`. An **identity flow** has `I == O`; it is obtained from the cortex factory and is
the starting point for building a Flow pipeline.

#### 6.2.2 Stateless Fiber Operators

**Guard (predicate)**: Passes emissions only when a predicate returns true. Each emission is
evaluated independently.

**Replace**: Transforms each emission using a same-type mapping function. If the function returns an
absent value, the emission is dropped (filtered). Operates within the existing type without
introducing a type boundary. Each emission is transformed independently.

**Clamp**: Replaces values that fall outside an inclusive range `[lower, upper]` with the nearest
bound. Values below `lower` are replaced with `lower`; values above `upper` are replaced with
`upper`; values within the range pass through unchanged. Uses a comparator (§6.2.4) to define
ordering. Each emission is clamped independently.

**Peek**: Observes emissions without modifying them. A receptor is invoked for each emission that
reaches this point in the pipeline. The emission continues downstream unchanged.

**Tee**: Forwards each emission to a side pipe without interrupting the downstream flow. Fan-out
semantics: the emission is delivered to the side pipe and continues to the next stage unchanged.
Stateless — each emission is dispatched independently. Distinguished from Peek: Tee targets a peer
pipe (typed emission target); Peek targets a receptor (untyped observation callback).

**Route**: Dispatches matching emissions to a side pipe and drops them from the main pipeline.
Non-matching values pass through unchanged. Stateless — each emission is evaluated independently.
Combine multiple Route stages to implement demultiplexing or partitioning across several pipes. The
predicate determines which emissions are diverted.

**When**: Conditionally routes emissions through an inline sub-fiber when a predicate matches. On no
match the value passes through unchanged. On match the value traverses the sub-fiber; whatever
emerges — including suppression — rejoins the main pipeline at this position. When itself is
stateless — it carries no state across emissions. The sub-fiber is a recipe value; its
per-materialization state, if any, is allocated alongside the enclosing Fiber's state at attachment
time, exactly as for any composed Fiber. Multiple When stages compose independently: a value may
match several predicates in sequence, each applying its own sub-fiber transformation. Predicate
exclusivity is the caller's responsibility.

#### 6.2.3 Stateful Operators

Stateful operators maintain internal state across emissions. Because they execute exclusively within
the circuit context, their state requires no synchronization. State is allocated per
materialization; a single Fiber or Flow value materialized multiple times produces independent
per-materialization state. The Fiber operators in this section are type-preserving (state, input,
and output share the type `E`). The Flow stateful operators are type-changing: Scan's state type
`S` is independent of input `O` and output `P`, while Window changes the output from `O` to
`Window<O>` by exposing the framework-owned rolling buffer for the current callback.

**Diff**: Emits only when the current value differs from the previous emission. The first emission
always passes. An optional initial value MAY be provided for comparison with the first emission.
Equality is determined by the projection's standard value equality mechanism — `equals()` in Java,
`__eq__` in Python, `==` for value types in Go, `Eq` trait in Rust, or equivalent. Canonical
identity (§1.2) is not sufficient; value equality is REQUIRED.

**Heartbeat**: Consecutive-duplicate suppression with a processing-time keep-alive. A positive
duration `maxSilence` is provided. A value that differs from the previously emitted value (by the
same value-equality mechanism as Diff) always passes and clears the heartbeat timer without
consulting the clock. While a run of duplicates continues, the first duplicate anchors the timer and
is dropped; a later duplicate whose processing-time arrival is at least `maxSilence` after the
anchor is emitted as a heartbeat — re-emitting the held value unchanged — and re-anchors the timer.
The first emission always passes. Silence is therefore measured from the start of the current
duplicate run, not from the last emitted value; the processing-time clock is consulted only on
duplicate emissions, so a stream of changing values costs no more than Diff. Elapsed time is
measured when the emission reaches this operator on the circuit context (§5.8), not when the value
was submitted by the caller. Stateful per materialization. Distinguished from Diff: Diff drops every
consecutive duplicate unconditionally; Heartbeat additionally re-emits a duplicate once its run has
persisted for
`maxSilence`, providing a keep-alive for downstream consumers during long stable runs.

**Change**: The projection-aware variant of Diff. A key function is applied to each emission and the
derived key is compared against the key of the previous emission. If the keys differ (or this is the
first emission), the emission passes and the stored key is updated; otherwise the emission is
dropped. Key comparison uses the projection's value equality mechanism; null keys are compared as
equal to other null keys. Distinguished from Diff: Diff compares whole values; Change compares a
projected attribute, which is the common pattern for event streams where only a derived category
matters.

**Guard (bi-predicate)**: Maintains a reference to the last emitted value. Each emission is compared
with the previous value using a bi-predicate. Only emissions where the predicate returns true are
forwarded and become the new previous value. An initial value MUST be provided for comparison with
the first emission; the initial value MUST NOT be absent. Projections that wish to support a "first
emission always passes" variant SHOULD expose it as a separate operator overload without an initial
value, rather than by accepting an absent initial.

**Limit**: Passes at most N emissions, then drops all subsequent values. Once the limit is reached,
the pipeline is permanently blocked at this stage.

**Skip**: Drops the first N emissions, then passes all subsequent values. The inverse of limit.

**TakeWhile**: Passes emissions while a predicate returns true. Once the predicate returns false for
the first time, all subsequent emissions are dropped regardless of the predicate. The transition to
the dropping state is permanent.

**DropWhile**: Drops emissions while a predicate returns true. Once the predicate returns false for
the first time, all subsequent emissions are passed regardless of the predicate. The transition to
the passing state is permanent. Symmetric to TakeWhile.

**Reduce**: Maintains a running accumulation. An initial value and a binary operator are provided;
the initial value MAY be absent. Each emission is combined with the current accumulator using the
operator. The result becomes the new accumulator state. If the result is present, it is emitted; if
the result is absent, the current emission is dropped and subsequent operator invocations receive
absence as the accumulator until the operator returns a present value. When the initial value is
absent, the accumulator starts absent and the operator is invoked with an absent first argument on
the first emission — callers who supply an absent initial value MUST ensure the operator handles an
absent accumulator without signaling an error.

**Relate**: Emits a derived value computed from the previous input and the current input. An initial
value and a binary operator are provided; the initial value MAY be absent. On each emission, the
operator is invoked as `operator(previous_input, current_input)`; the result is emitted and
`previous_input` is updated to `current_input`. When the initial value is absent, `previous_input`
starts absent and the operator is invoked with an absent first argument on the first emission —
callers who supply an absent initial value MUST ensure the operator handles an absent previous input
without signaling an error. If the operator returns an absent value, the emission is dropped but the
previous-value state is still updated. Distinguished from Reduce: Reduce tracks an accumulator (the
state IS the output), whereas Relate tracks the previous input (the state tracks input history
independent of output). This distinction is load-bearing for derivative-style computations (deltas,
differences, two-point smoothing).

**Edge**: The transition-detector primitive. A bi-predicate and an initial previous-input value are
provided; the initial value MUST NOT be absent. On each emission, the predicate is invoked as
`transition(previous_input, current_input)`; if it returns true, the current value is emitted,
otherwise the emission is dropped. The previous-input state advances to the current input on every
emission regardless of pass/drop. Distinguished from Guard (bi-predicate): Guard advances its
reference only when the predicate passes; Edge advances on every input. Distinguished from Diff:
Diff is the special case `edge((p, c) -> !equals(p, c))` with the first emission as the seed; Edge
generalizes to arbitrary transitions — zero-crossings, rising edges, threshold crossings.

**Pulse**: The rising-edge detector primitive. A boolean predicate is provided. An internal `active`
state starts false. On each emission, if the predicate returns true and the previous state was
false, the current value is emitted and the state becomes true; otherwise the emission is dropped
and the state tracks the current predicate result. The gate re-arms whenever the predicate returns
false — the next true reading emits again. Distinguished from Edge: Edge operates on
`(previous_input, current_input)` transitions on raw values; Pulse specializes to rising-edge
detection on a boolean predicate, retaining only a single boolean of state.

**Steady (count)**: Temporal-confirmation primitive. A count `N` is provided. A change is reported
only after it has been confirmed by `N` consecutive identical emissions. Internal state tracks a
candidate value, a run counter, and whether the current run has already been emitted. On each
emission: if the value equals the candidate, the counter increments and the value is emitted exactly
once when the counter reaches `N`; otherwise the candidate resets to the new value with counter 1
(and emits immediately if `N == 1`). For `count = 1`, Steady degenerates to Diff.

**Steady (bi-predicate)**: Generalized temporal-confirmation. A count and a bi-predicate are
provided. The bi-predicate compares each incoming value against the *candidate* — the first value
that started the current run — not against the immediately preceding value. This anchored comparison
detects drift: a signal that moves gradually away from the candidate resets the counter even if each
individual step is small.

**Streak**: Consecutive-match thresholding primitive. A positive integer `required` and a unary
predicate `matches` are provided. An internal counter starts at zero. On each emission: if `matches`
returns true, the counter increments; if the incremented counter reaches `required`, the value is
emitted and the counter resets to zero. If `matches` returns true but the counter has not yet
reached `required`, the emission is dropped and the counter is retained. If `matches` returns false,
the counter resets to zero and the emission is dropped. For `required = 1`, Streak degenerates to
Guard (predicate) — every match is emitted with no carried state. Distinguished from Steady: Steady
counts confirmations of a stable value (consecutive equal values); Streak counts confirmations of a
predicate (consecutive matching values, regardless of value identity). Distinguished from Every:
Every counts unconditionally and emits every Nth value; Streak filters by
`matches` first and resets the count on any non-match. The cumulative analogue — N matches across a
stream regardless of intervening misses — is the composition of Guard followed by Every.

**Hysteresis**: Gated pass-through with two-threshold latch. Enter and exit predicates are provided.
An internal active/inactive state starts inactive. While inactive, each emission is tested against
the enter predicate; if it returns true, the state transitions to active and the triggering value is
emitted. While active, each emission is emitted unchanged unless the exit predicate returns true, in
which case the state transitions to inactive and the triggering value is dropped. The asymmetric
entry/exit semantics are deliberate — the enter event is itself a valid in-regime emission, while
the exit event is the first out-of-regime emission and therefore suppressed.

**Inhibit**: Refractory-period primitive. A non-negative integer `refractory` is provided. When an
emission passes through, the next `refractory` emissions reaching this operator are suppressed. Once
the refractory count is exhausted, the next emission passes and the cycle repeats. The first
emission always passes immediately. Distinguished from Every: Every drops the first N-1 and passes
the Nth (periodic thinning); Inhibit passes the first and drops the next N (cooldown after
detection). The phase difference matters when composed after event-detecting operators.

**Integrate**: The **integrate-and-fire** pattern — accumulate silently until a fire condition is
met, then emit and reset. An initial value, a binary accumulator operator
`(state, input) → new state`, and a fire predicate are provided; the initial value MAY be absent. On
each emission: the accumulator is updated; if the fire predicate tests true of the new state, the
accumulator value is emitted downstream and the state resets to the initial value; otherwise, the
emission is filtered (no downstream emit) and the state is retained. When the initial value is
absent, the accumulator starts absent, resets to absent after every fire, and the accumulator
operator and fire predicate are invoked with an absent state argument on the first post-reset
emission — callers who supply an absent initial value MUST ensure both handle an absent state
without signaling an error. The accumulator operator MAY return absence; the fire predicate receives
that absence as the new state. If the fire predicate returns true for an absent state, the stage
resets to the initial value but the absent output still drops the current emission. Distinguished
from Reduce: Reduce emits on every step; Integrate accumulates silently and emits only at fire
points. Intended for batching, chunking, alternative-bar construction, and spiking-neuron patterns.

**Delay**: Temporal-displacement primitive. A positive integer `depth` and an initial value are
provided. A circular buffer of size `depth` is maintained, initialized with the initial value. On
each emission, the oldest value in the buffer is emitted downstream and replaced with the current
input. The first `depth` emissions each produce exactly the initial value; thereafter each emission
produces the value received `depth` steps earlier. Never filters — every input produces exactly one
output. Composes with Relate for shifted-delta computations.

**Rolling**: Sliding-window aggregation. A positive integer `size`, a binary combiner, and an
identity value are provided. A circular buffer of `size` slots is maintained. On each emission, the
oldest slot is overwritten by the current input. Once the buffer is full, each subsequent emission
folds the entire buffer — from `identity`, over all `size` values, in insertion order — through
`combiner` and emits the result when present. The combiner MAY return absence; the fold continues
with absence as the accumulator, and an absent final aggregate drops the current emission. The first
`size - 1` inputs are warm-up and produce no output. The fold is recomputed in O (size) per
emission, enabling any `BinaryOperator` including non-invertible ones (max, min, concatenation).

**Tumble**: Count-triggered aggregation. A positive integer `size`, a binary combiner, and an
identity value are provided. A running accumulator starts at `identity`. Each emission is folded
into the accumulator via `combiner`. After exactly `size` emissions the accumulator is emitted
downstream when present and reset to `identity`; the intermediate `size - 1` emissions produce no
output. The combiner MAY return absence; the accumulator state becomes absent, and if the batch
fires with an absent accumulator the current emission is dropped before the state resets to
`identity`. If the stream ends with fewer than `size` inputs since the last emission, the in-flight
partial window is never emitted. Distinguished from Rolling: Rolling produces overlapping windows
(one emission per input after warm-up); Tumble produces non-overlapping windows (one emission per
`size` inputs). Distinguished from Integrate: Integrate fires when the accumulator state satisfies a
predicate; Tumble fires after a fixed number of inputs regardless of content.

**Every (interval)**: Emits every Nth value, dropping others. For example, every (3) emits the 3rd,
6th, 9th values.

**Every (duration)**: Processing-time interval sampler. A positive duration is provided. The first
observed value anchors the interval and is dropped. The next value whose processing-time arrival is
at least `duration` after the anchor is emitted and becomes the new anchor. If more than one full
interval has elapsed between successive arrivals, the internal clock advances by whole interval
slots rather than to the arrival time, so the operator does not drift after overruns. Elapsed time
is measured when the emission reaches this operator on the circuit context, not when the value was
submitted by the caller. Stateful per materialization. Distinguished from Every (interval): the
count-based form samples by emission ordinality regardless of timing; the duration-based form
samples by processing-time elapsed regardless of intervening emission count.

**Distinct**: Suppresses all previously seen emissions. A growing set of observed values is
maintained; any value that has appeared before is dropped. Memory grows proportionally to stream
cardinality. Stateful. Distinguished from Diff: Diff suppresses consecutive duplicates; Distinct
suppresses all historical duplicates regardless of position.

**Distinct (capacity)**: Suppresses emissions seen within a FIFO window of `capacity` accepted
distinct values. When the window is full, the oldest accepted value is evicted, allowing that value
to pass again in future emissions. Suppressed duplicates do not refresh their position in the
window. Stateful. Distinguished from unbounded Distinct: the bounded variant allows values to
re-emerge after the window evicts them, making it suitable for streams with cycling value sets.

**Chance (probability)**: Probabilistically passes each emission with the specified probability. A
probability of 0.0 drops all emissions. A probability of 1.0 passes all emissions. Per-emission
decisions are made using an implementation-defined uniform random source. Reproducibility across
runs, across materializations of the same Fiber value, or across separate channels is NOT required —
implementations MAY use a non-deterministic source (e.g., a freshly seeded PRNG per
materialization). Applications requiring deterministic sampling SHOULD use Every (interval)
instead.

**Scan (Flow)**: The type-changing stateful fold; the Flow analogue of Reduce. Folds each emission
of `O` through a binary `step(state, input) -> state` function and emits either the resulting state
itself or a projection of that state. The state type `S` is independent of input `O`, enabling
running summary statistics whose state shape (e.g., `(sum, count)`, `(count, mean, M2)`) differs
from the input and, in projected forms, from output `P`. Three emission shapes are defined:
state-as-output `scan(initial, step)` emits the resulting `S`; state-only projection
`scan(initial, step, emit(state))` emits `P`; input-aware projection
`scan(initial, step, emit(state, input))` emits `P` for cases that need to blend the running state
with the current sample (z-scores, residuals, anomaly scoring). The state-as-output form directly
publishes the accumulator, so callers MUST ensure each emitted `S` is safe to publish downstream:
immutable, effectively immutable, or otherwise not mutated after publication. Mutable working state
MUST use a projected form that emits a safe public value, typically an immutable snapshot. A `step`
function MAY return the same `S` reference when that reference is safe to publish; implementations
MUST NOT reject a state-as-output scan solely because the returned reference is identical to the
previous state. The initial-state Supplier is invoked once per materialization at attachment time,
not lazily on first emission; the returned value becomes the initial state slot. The Supplier is the
per-attachment hook for state allocation; for mutable `S`, callers MUST return a fresh instance from
each invocation. The framework stores whatever the Supplier returns without copying or otherwise
enforcing freshness. Per-subject seed configuration is not expressed via a per-operator overload;
callers compose Scan inside `Flow.flow(factory)` (§6.2) where the materialized subject is in scope.
**Per-emission semantics**: on each emission, `step` is invoked with the current state and the
input; if it returns normally the stored state reference is replaced with the result, after which
the scan emits that state directly or invokes `emit`. If the direct state-as-output value or
projected `emit` result is absent, the emission is dropped (filtered downstream). **Exception
semantics** (asymmetric and normative): if `step` raises, the stored state reference is NOT
replaced — side effects or in-place mutations performed by `step` before raising are not undone, and
the next emission proceeds from the prior state; if `emit` raises, the stored state has already been
advanced by `step` and the next emission proceeds from the advanced state. Both raises propagate to
the external callback isolation rule in §15.4. Allowed null-return contracts: the Supplier MAY
return absent (state begins absent and is passed to `step` as `prev`); `step` MAY return absent
(state becomes absent and is passed to `emit` or emitted directly as absence); `emit` MAY return
absent (filters the emission downstream).

**Window (Flow)**: Bounded rolling temporal window. A positive integer `count` is provided. For each
surviving upstream `O` value, the operator appends the value to a per-materialization rolling buffer
and emits a `Window<O>` containing at most the most recent `count` surviving values. During warm-up,
before `count` values have survived, the emitted window contains the available prefix and is never
empty. The encounter order of the window emitted by `Flow.window(count)` is oldest to newest. Window
restriction operations (`prefix`, `suffix`, `skip`, `trim`, `slice`, and
`reverse`) operate in the current window's encounter order and produce another callback-scoped
Window over the same stable emission set without requiring value copies. The Window also exposes
eager terminal operations (`forEach`, `isEmpty`, `all`, `any`, `none`, `count`, `fold`, `reduce`)
that consume the view without allocating lazy traversal handles. `count` MUST be greater than zero;
there is intentionally no unbounded `window()` operator. Unbounded history retention is a different
policy and can exhaust memory under long-running or high-rate feeds.

**Window (Flow, duration + capacity)**: Time-bounded rolling temporal window. A positive `duration`
and a positive integer `capacity` are provided. For each surviving upstream `O` value, the operator
captures the circuit-context processing timestamp — the stimulus time of the current ingress chain
(§5.8), so all values in one cascade are co-temporal — evicts retained values whose timestamps are
older than the configured duration relative to the current emission, and then appends the new value
before emitting a `Window<O>`. Each emitted window contains values observed no earlier than
`duration` before the current emission and at most `capacity` values; the capacity bound MAY evict
older values that are still within the duration bound under high input rates. The emitted window is
never empty — the current surviving value is always included — and its encounter order is oldest to
newest. As with the count-based form, the Window is callback-scoped (§6.4) and MUST NOT be retained,
emitted, or forwarded; callers that need values beyond the callback MUST copy them. There is
intentionally no duration-only overload: time-based retention without an explicit capacity bound is
unbounded in memory under high-rate feeds and is therefore prohibited.

**Run (Flow)**: Per-admission run-length annotation. For each surviving upstream `O`, the operator
emits a `Run<O>` envelope carrying the emission and the length of its consecutive run — the number
of consecutive value-equal admissions ending at this one. The length is 1 on the first admission and
after every change, and increments while the value repeats. Consecutive repetition is decided by
value equality (§1.2); callers wanting repetition over a derived key map to the key first. Stateful
per materialization (a predecessor and a running count, mutated only within the circuit context).
Cardinality is preserved: every admission emits exactly one `Run`. The `Run` is an immutable
envelope; unlike `Window` it is not callback-scoped and MAY be retained, compared, and forwarded.

**Change (Flow)**: Run-boundary annotation — the boundary projection of `Run`'s resets. The operator
emits only at a boundary, an admission value-unequal to its predecessor, a `Change<O>` envelope
carrying the value the closed run held (`from`), the value that opened the next run (`to`), and the
terminal `length` the closed run reached. The first admission opens the first run and MUST produce
no output; a value-equal admission continues the open run and MUST produce no output; a
value-unequal admission closes the run and emits the change. The `length` a change carries MUST
equal the length
`Run` would have reported for `from` on the admission immediately before the boundary; consequently
the open or final run is never reported by `Change`, and its length is observable only through
`Run`. Consecutive repetition is decided by value equality (§1.2). Stateful per materialization. The
`Change` is an immutable envelope; not callback-scoped, it MAY be retained, compared, and forwarded.

#### 6.2.4 Comparison Operators

Comparison operators are Fiber operators that provide value filtering and clamping using a
comparator that defines value ordering. The comparator returns an **Ordering** — an abstract
three-valued result representing *less than*, *equal to*, or *greater than*. Projections map this to
their idiomatic ordering type: `Comparator<T>` (returning negative/zero/positive integer) in Java,
`Ordering` enum in Rust, `cmp.Ordering` in Go 1.21+, a three-case enum or integer convention as
appropriate. The mechanism is projection-dependent; the three-valued semantics are normative.
Comparison operators include:

- **Above**: Passes values strictly greater than a lower bound (exclusive).
- **Below**: Passes values strictly less than an upper bound (exclusive).
- **Min**: Passes values greater than or equal to a minimum (inclusive).
- **Max**: Passes values less than or equal to a maximum (inclusive).
- **Range**: Passes values within an inclusive range (lower ≤ value ≤ upper).
- **Deadband**: The complement of Range. Passes values strictly outside an inclusive band (value <
  lower OR value > upper); drops values at or between the bounds. Stateless. Intended for noise
  suppression around a setpoint.
- **Clamp**: Replaces out-of-range values with the nearest bound (see §6.2.2). Stateless; unlike the
  filtering operators above, clamp modifies rather than drops.
- **High**: Passes only values that represent a new maximum (running high-water mark). Stateful.
- **Low**: Passes only values that represent a new minimum (running low-water mark). Stateful.

Each comparison operator is a standalone Fiber stage. Multiple comparison operators MAY be chained;
each operates independently.

#### 6.2.5 Operator Ordering

Fiber operators execute in strict left-to-right order — the reading order is the execution order. No
pivots, no inversion. Cheap filters SHOULD be placed early to reduce downstream work.

Flow operators follow a **left-to-right builder** convention for output-side stages. `map` appends a
transformation downstream of this flow; its signature — `(O) → P` — maps the current output type `O`
into a new downstream type `P`. `scan` appends a stateful type-changing fold at the same output-side
position; later Flow stages see the scan output type `P`. `window(count)` appends a stateful bounded
rolling window at the same output-side position; later Flow stages see `Window<O>` as the output
type. `flow(next)` appends the supplied Flow as a suffix, and `flow(factory)` resolves a suffix Flow
once per materialization before emissions flow. The textual chain order matches execution order.

Reading a flow chain from emit side to downstream: successive `map`, `scan`, `window`, and
`flow(...)` stages produce successive downstream type-widenings; a `fiber(Fiber)` at the output side
runs its Fiber operators last on each emission before dispatch.

The subject-aware `fiber(factory)` and `flow(factory)` forms follow the same output-side placement
as their non-factory siblings. Each factory runs once per pipe attachment, before emissions flow,
and receives the subject of the materialized pipe being attached. This subject is the canonical key
for per-pipe configuration: its name identifies the destination pipe and its enclosing extent
identifies the downstream pipe receiving the flow output.

#### 6.2.6 Pipe Composition

Both Fibers and Flows are applied to a pipe via composition methods on the composition values
themselves:

- `fiber.pipe(pipe)` returns a new pipe of the same type `E` whose emissions are processed by the
  fiber's operator chain before reaching the original pipe. This is the direct attachment for
  type-preserving recipes; no widening through Flow is required. The pipe argument MUST be a
  provider-compatible Pipe (§15.1 provider mismatch).
- `flow.pipe(pipe)` returns a new pipe whose input type is the flow's upstream type `I` — which may
  differ from the original pipe's type when the flow carries a `map`. The flow's stages (including
  any attached Fiber) execute before dispatch to the original pipe. The pipe argument MUST be a
  provider-compatible Pipe (§15.1 provider mismatch).
- `fiber.pipe(cell)` attaches the fiber recipe ahead of the cell's update pipe. Emissions that
  survive the fiber's operator chain are forwarded to the cell's update pipe and become its
  published value; filtered emissions do not update the cell. The cell argument MUST be a
  provider-compatible Cell (§15.1 provider mismatch). The returned pipe carries the same type `E` as
  the fiber.
- `flow.pipe(cell)` attaches the flow pipeline ahead of the cell's update pipe. Output emissions
  from the flow are forwarded to the cell's update pipe and become its published value; absent
  outputs do not update the cell. The returned pipe carries the flow's upstream type `I`. The cell
  argument MUST be a provider-compatible Cell (§15.1 provider mismatch).

In all four forms the target occupies a pure consumer position: a pipe or cell whose element type
is a supertype of the delivered type MUST be accepted (in typed projections, `Pipe<? super E>` /
`Cell<? super E>` for fibers and `Pipe<? super O>` / `Cell<? super O>` for flows), matching the
contravariance of `Registrar.register` and `Circuit.pipe(target)`.

Each composed pipe has its own per-materialization state slots, ensuring stateful operators (diff,
reduce, limit, integrate, relate, scan, etc.) maintain separate framework-owned state per composed
pipe. For Scan, mutable user state is independent only when the Supplier returns a fresh object for
each materialization. All processing executes within the circuit context.

Fibers and Flows are constructed from identity values obtained from the cortex factory (§16.3), have
operators chained onto them, and are then handed a pipe via `fiber.pipe(...)` / `flow.pipe(...)`.
The value itself is independent of any pipe until composed. After composition, the returned pipe is
the only reference to the underlying materialized state.

### 6.3 Pipe Dispatch

After an emission passes through all fiber/flow stages, the named pipe dispatches it to all
registered downstream pipes and receptors. Pipe registrations are invoked sequentially in
registration order within the circuit context. Receptor registrations have the same channel
visibility, temporal validity, circuit-context confinement, and failure-isolation guarantees as pipe
registrations, but implementations MAY optimize their internal representation (for example, by
storing a receptor directly rather than wrapping it in a pipe). An exception thrown by any one
downstream pipe or receptor MUST NOT prevent dispatch to siblings registered on the same channel;
see §15.4.

**Receptor instances and cross-circuit sharing**: Circuit-context confinement is a per-circuit
guarantee. A receptor instance registered with exactly one circuit is invoked from that circuit's
single execution context and may safely hold mutable state without synchronization. The same
receptor instance registered with two or more circuits MAY have its receive operation invoked
concurrently from different circuit contexts; the per-circuit confinement does not extend across
circuits. Such receptors MUST protect their own state with appropriate synchronization or avoid
mutable state entirely. This parallels the operator-function purity contract for Fibers and Flows
materialized across multiple circuits (§6.2, §16.1#16).

### 6.4 Temporal Contracts

Several types in the specification carry a **temporal contract** — they are valid only during a
bounded lifetime and MUST NOT be used beyond that lifetime. The specification defines two temporal
lifetime scopes:

**Callback-scoped**: Valid only during a specific callback invocation. The object MUST NOT be
retained after the callback returns.

- **Registrar**: Valid only during the subscriber callback.
- **Window**: Valid only during the receptor or Fiber callback that observes it.

**Single-use-scoped**: Valid for exactly one invocation of its primary operation. After that
invocation, further use is illegal. Validity is also bounded by the owning scope's lifetime — if the
scope closes before the operation is invoked, the operation becomes a no-op.

- **Closure**: Valid for exactly one call to `consume`. Subsequent calls are illegal. If the owning
  scope is already closed, `consume` MUST be a no-op.

`Current` is not temporal. A Current is an interned identity token for an execution context and MAY
be retained or shared for identity comparison and immutable subject inspection. `Cortex.current()`
answers "which execution context is calling now"; a previously retained Current continues to
identify the context that produced it and does not update when observed from another context.

#### 6.4.1 Enforcement Requirements

Using a temporal object beyond its lifetime is an **illegal operation**. Implementations MUST handle
violations in one of the following ways:

- **Detect and signal**: Signal an illegal temporal use error (§15.1). This is the RECOMMENDED
  behavior where detection does not impose overhead on the valid (in-scope) execution path.
- **Undefined behavior**: The operation produces arbitrary results, including silent corruption.
  This is acceptable only when detection would require per-invocation checks that degrade
  performance on the valid path.

Implementations SHOULD document which temporal contract violations are detected and which produce
undefined behavior.

**Registrar exception**: For `Registrar` specifically, the performance escape clause does not apply.
Registration is not a hot-path operation — it occurs only during subscriber callbacks, which are
themselves rare relative to emission. Detection via sentinel state imposes negligible cost on
valid-path execution. Implementations MUST detect and signal `Registrar` temporal contract
violations; undefined behavior is not an acceptable choice for this type.

**Window exception**: For `Window` specifically, the performance escape clause does not apply
either. Window operator invocations land on the user's callback path, not the framework's emission
dispatch path: the framework hot path is the per-emission `emit → receive` chain that materializes
the Window, while operator entries (`first`, `fold`, `forEach`, view derivations, etc.) are
user-driven calls inside the receiving callback. Detection via a per-operator lease check imposes
one branch per operator entry — negligible compared to the operator's own work, and entirely off the
framework emission hot path. The silent-corruption modes for escaped Windows are also unusually
insidious: provider reuse of the Window reference across callbacks means same-type comparison on
`Window<O>` (such as `Fiber.diff()` by reference equality)
would silently return false-negatives instead of detecting changes, and a leaked Window forwarded
into another pipeline would observe values from a future emission cycle without error.
Implementations MUST detect and signal `Window` temporal contract violations; undefined behavior is
not an acceptable choice for this type.

The general SHOULD-detect framework above remains applicable to any other temporal types whose
validity-checks would lie on a hot path.

Implementations MAY reuse callback-scoped temporal objects across callbacks. This reuse is safe
precisely because the temporal contract prohibits retention. A Window observed during a callback
MUST remain stable for the duration of that callback, including across same-circuit emissions queued
from the callback itself; the queued emissions MUST NOT advance that observed Window re-entrantly.
Callers that need values beyond the callback MUST copy those values during the callback.

## 7. Subscription Model

The subscription model enables dynamic discovery of channels and adaptive topology construction.

### 7.1 Source

A **Source** is any component that supports subscription. Sources are Resources (§9.1) — they
inherit identity (via subject) and lifecycle (via close) from the Resource role, and add the
subscription affordance on top. The specification requires one source type: conduits (which emit
domain values through channels). Projections MAY provide additional source types that satisfy the
same Source contract.

### 7.2 Subscriber

A **Subscriber** is a callback mechanism created from a circuit. When subscribed to a source, the
subscriber receives callbacks as named pipes are discovered.

Subscribers are circuit-scoped — they execute within the circuit context that created them. A
subscriber MUST NOT be subscribed to a source belonging to a different circuit.

**Enforcement**: Cross-circuit subscription is a caller-misuse condition, not a timing race. The
binding between a subscriber and its owning circuit is established at construction time and is
knowable synchronously at the `subscribe` call site — the caller has explicitly passed a subscriber
from another circuit. A conformant implementation MUST detect this condition and MUST signal an
illegal-argument error per the enforcement framework in §15.1, surfacing the error *synchronously on
the caller context before any subscription registration takes place*. The `subscribe` operation MUST
NOT proceed, MUST NOT admit registration work, and MUST NOT install the subscriber on the source.
This is distinct from the queued-operation drop semantics of §9.1 (post-close): those exist because
close/emit races are inherent and unknowable at call time, whereas cross-circuit misuse is a
deterministic programming error that silent behavior would render undiagnosable — a caller doing
`source.subscribe(foreignSub); circuit.await();` and observing zero callbacks would have no way to
distinguish a dropped subscription from one that simply hasn't delivered yet (§5.5).

The check MUST happen before the subscription is registered with the source and before any pipe is
created on behalf of the subscriber, so that a rejected subscription leaves no observable side
effects on either circuit.

Subscribe is also an open-required operation (§9.1). If the source has accepted close, or if the
subscriber argument has accepted close, `subscribe` MUST signal a closed-resource error
synchronously before admitting registration work. A closed-source rejection is attributed to the
source's subject. A closed-subscriber rejection is attributed to the source's subject and identifies
the subscriber's subject as the offending argument. In either case, no subscription exists and any
supplied `onClose` callback MUST NOT be invoked.

A circuit MAY create a **pool-backed subscriber** from a name and a pool of pipes. This is
semantically equivalent to a callback subscriber that, for each discovered pipe subject, obtains a
pipe from the pool using that subject and registers it with the registrar. This form is a
convenience for per-channel wiring and self-feedback topologies; it does not change subscription
timing, circuit affinity, lazy rebuild, or closed-resource behavior.

### 7.3 Subscription Lifecycle

The subscription lifecycle proceeds as follows:

1. A subscriber is created from a circuit, receiving a name and a callback function.
2. The subscriber is subscribed to a source. This returns a subscription handle. The subscribe
   operation is asynchronous — it submits a registration job to the circuit context and returns
   immediately. The registration does not take effect at the moment of return; its effect becomes
   observable only after the circuit context processes the registration job (see §5.5 and §7.6).
3. When a named pipe within the source processes an emission whose ingress-admission position falls
   within the subscription's visibility window (§7.6), and the named pipe has not yet rebuilt for
   this subscription, a rebuild is triggered within the circuit context.
4. During rebuild, the subscriber's callback is invoked with the pipe's subject and a registrar.
5. The callback uses the registrar to attach downstream pipes or receptors.
6. Registered pipes receive the emission that triggered the rebuild and all subsequent emissions
   from that named pipe that fall within the subscription's visibility window.

Key properties of this model:

- **Lazy invocation**: Callbacks are invoked on first emission, not on pipe creation. Creating a
  pipe via `get(name)` does not trigger callbacks. This avoids overhead for pipes that never emit.
- **Exactly-once callback per subscription/channel pair**: For each channel that receives an
  emission while a subscription is active, the subscriber callback is invoked exactly once for that
  subscription/channel pair. If the callback signals an error, the failure is handled under the
  external callback isolation rule (§15.4), registration calls completed before the error remain
  registered, and the callback is not retried for that subscription/channel pair.
- **Dynamic discovery**: Subscribers do not need prior knowledge of pipe names. They discover pipes
  as those pipes become active.
- **Circuit-context execution**: All callbacks execute within the circuit context. No
  synchronization is needed within callback logic.

### 7.4 Registrar

A **Registrar** is a temporal handle provided during a subscriber callback. It allows downstream
pipes or receptors to be attached to the named pipe identified by the callback's subject.

Registration semantics:

- Multiple downstream pipes or receptors MAY be registered for the same named pipe. All receive
  every emission (fan-out).
- Pipe registrations are invoked in registration order. Receptor registrations have the same
  delivery and failure-isolation contract, but an implementation MAY represent them directly rather
  than as pipes; projections that expose ordering details across mixed pipe and receptor
  registrations MUST document that ordering.
- The same pipe or receptor instance MAY be registered multiple times, creating independent
  callbacks.
- The registrar carries a temporal contract (see §6.4). Using it after the callback returns is an
  illegal operation.

### 7.5 Subscription Handle

A **Subscription** is a cancellable handle returned by the subscribe operation. Closing a
subscription:

- Stops all future subscriber callbacks from this subscription.
- Removes all pipes registered by this subscription from active channels.
- Is idempotent — repeated close calls MUST be safe.

Subscription close is an asynchronous operation: it submits a close job to the circuit context and
returns immediately (per §5.5). The close does not take effect at the moment of return. Its effect
becomes observable only after the circuit context processes the close job. Callers that need to
observe the effect of close — for example, to verify that subsequent emissions will not reach the
subscription — MUST call `await` to synchronize.

Unsubscription uses lazy rebuild. Named pipes detect the unsubscription on their next emission and
rebuild their downstream pipe lists to exclude the removed pipes. This avoids global coordination
and suspension.

After closing a subscription, the same subscriber MAY be resubscribed to the same or different
sources. Each subscribe operation creates an independent subscription instance.

### 7.6 Eventual Consistency

#### 7.6.1 Visibility Window

A subscription's **visibility window** is the half-open interval of ingress-admission positions
`[subscribe_admitted, close_admitted)`, where `subscribe_admitted` is the admission position of the
subscription's registration job and `close_admitted` is the admission position of the subscription's
close job (or unbounded if the subscription has not been closed).

A subscription receives exactly those emissions whose ingress-admission position falls within this
window. Emissions accepted before the subscribe job, or at-or-after the close job, are not visible
to the subscription.

This is a direct consequence of the FIFO ordering of the accepted operation sequence (§5.1, §5.4.3).
Subscribe, emit, and close are all admitted operations, and the circuit context processes them in
admission order. A single caller context that executes `subscribe(); emit(X); await()`
observes the emission `X` on the new subscriber because, by per-caller FIFO, the subscribe job is
admitted before the emit job; the circuit processes them in that order; the registration is
installed before `X` is dispatched. Conversely, a caller that executes `emit(X); subscribe();
await()` does not observe `X` on the new subscriber, because `X` is admitted before the registration
job and is dispatched before the subscription is installed. Symmetrically for close:
`subscribe(); emit(X); close(); emit(Y); await()` delivers `X` but not `Y` to the subscription.

Across multiple concurrent caller contexts, the relative admission order of their operations is
determined by the implementation's ingress admission mechanism and is not specified (§5.4.3). The
visibility window rule still applies — each subscription's window is defined by its own admission
positions — but the set of emissions that fall within a given window may differ across runs when
multiple callers are racing.

#### 7.6.2 Version-Tracked Lazy Rebuild

Subscription changes (add/remove) use lazy rebuild with version tracking. A version counter is
incremented when subscriptions change. Named pipes detect version mismatches on their next emission
and rebuild their downstream pipe lists.

This produces eventual consistency: subscription changes are not immediately visible to all named
pipes. Each named pipe discovers the change on its next emission. The benefit is lock-free operation
with minimal coordination overhead. The visibility window rule in §7.6.1 is preserved because the
version increment is itself processed in ingress-admission order within the circuit context.

**Replay note**: Because subscription registrations and unsubscriptions are queued operations that
modify pipeline topology, a deterministic replay log MUST capture these operations alongside
emissions. The version counter increment that triggers lazy rebuild is a circuit-context side effect
of the registration operation. If a replay log contains only emissions, the rebuild timing and
resulting pipe lists may differ from the original execution, producing non-deterministic routing.
See §5.1 for the complete replay requirement.

## 8. State

### 8.1 State Object

A **State** is an immutable collection of typed slots that uses structural sharing — each state
operation (adding a slot) returns a state value containing the new slot as the most recently written
entry, while prior state values MUST remain unchanged. Writing a slot whose (name, type) matches an
existing entry MUST upsert that logical entry: the new slot becomes the most recently written entry
and the prior matching slot is removed. An implementation MAY return the same state instance when
the operation would produce a semantically equivalent result (when an equal slot with the same value
already exists). ("Persistent" here refers to the persistent data structure property of structural
sharing, not to durable storage.)

Because of upsert semantics, a state contains at most one slot per (name, type) pair, and its size
is bounded by the number of unique (name, type) pairs ever written to the chain of states leading to
it. States iterate from the most recently written slot to the oldest.

### 8.2 Slot

A **Slot** is a named, typed value within a state. A slot carries:

- **Name**: The slot's identity within the state.
- **Type**: The value's type (e.g., integer, string, boolean, float, double, long, name, state).
- **Value**: The slot's immutable value.

Slot matching uses both name canonical identity (§1.2) and type value equality (§4.5). Multiple
slots MAY share the same name if they have different types. Slot instances are immutable — the value
MUST NOT change across invocations.

### 8.3 Supported Slot Types

The specification REQUIRES support for the following slot value types:

| Abstract Type | Definition                              | Example Projections                           |
|---------------|-----------------------------------------|-----------------------------------------------|
| Boolean       | Two-valued logical type                 | `boolean` (Java), `bool` (Rust/Go/Python)     |
| Integer       | Signed 32-bit integer                   | `int` (Java/Go), `i32` (Rust)                 |
| Long          | Signed 64-bit integer                   | `long` (Java), `int64` (Go), `i64` (Rust)     |
| Float         | IEEE 754 32-bit floating point          | `float` (Java), `float32` (Go), `f32` (Rust)  |
| Double        | IEEE 754 64-bit floating point          | `double` (Java), `float64` (Go), `f64` (Rust) |
| String        | Immutable character sequence            | `String` (Java/Rust), `string` (Go)           |
| Name          | Hierarchical interned identifier (§4.1) | —                                             |
| State         | Nested state (self-referential)         | —                                             |

The projection examples are non-normative. A conformant implementation MUST provide the specified
precision and range for numeric types. The abstract type names (Boolean, Integer, etc.) are
specification vocabulary — bindings are not required to use these names.

This set defines the minimum required slot types. Projections MAY support additional slot types
(e.g., unsigned integers, byte arrays, timestamps) as non-normative extensions, provided the
required types are supported.

## 9. Lifecycle Management

### 9.1 Resource

A **Resource** is any substrate component with explicit cleanup requirements. The close operation
releases resources and terminates associated operations. Every Resource is a substrate component in
the sense of §4 — it carries a `subject()` — so Resource is the junction point where identity and
lifecycle meet for every closeable component in the model.

Close semantics:

- **Idempotent**: The first call performs cleanup. Subsequent calls MUST be no-ops.
- **Concurrency-safe**: Close MAY be called from any execution context.
- **Queued execution**: For circuit-managed resources, close submits a cleanup job to the circuit
  context and returns immediately. The actual cleanup occurs asynchronously. No caller-thread
  cleanup fallback is performed: if the owning circuit has already accepted close and can no longer
  accept the cleanup job, the resource MAY remain with cleanup requested but not drained, and any
  `onClose`-style callbacks attached to that cleanup MAY never fire. Callers that require cleanup to
  complete deterministically MUST close circuit-backed resources before the owning circuit reaches
  its terminal state, or rely on `Scope`-driven (§9.2) cleanup that runs while the owning circuit is
  still drainable.
- **Synchronous close (closeAwait)**: Resources MUST provide a blocking close variant.
  `closeAwait()` requests close — identical in effect to `close()` — and returns only after the
  resource's cleanup has fully taken effect. For resources with queued close semantics this is
  equivalent to `close()` followed by a barrier await on the close's execution; a resource whose
  `close()` completes synchronously satisfies the contract with `closeAwait()` behaving as
  `close()`. `closeAwait()` is idempotent: multiple calls have no additional effect beyond the
  first. For circuit-backed resources, `closeAwait()` MUST NOT be called from within the owning
  circuit's context — the same restriction applies as for `Circuit.await()` (§5.5). This operation
  is useful when the caller must not proceed until cleanup has completed.
- **Closed-state guard**: Once a resource has accepted a close request, it is no longer open. An
  operation that is required to run only on an open receiver MUST reject the call synchronously when
  it observes the receiver as non-open, even if the resource's asynchronous cleanup has not yet
  completed.
- **Queued post-close semantics**: Pure queued operations that do not need to return a newly-created
  resource or snapshot to the caller — notably `emit`, `close`, and cleanup jobs — MUST NOT throw an
  error synchronously merely because the receiver closes concurrently. Queued operations whose
  ingress-admission position falls strictly after the component's own close job MUST be silently
  dropped by the component when processed on the circuit context: no delivery, no topology change,
  and no observable effect beyond any onClose-style callback symmetry defined by the specific
  operation type. The component MAY additionally report the drop via a side channel (e.g., a fault
  emission or a circuit-state event on the circuit context). Operations admitted strictly *before*
  the close job — pre-close operations still being drained — MAY be processed to completion or
  silently dropped at the implementation's discretion; implementations MUST document this choice.
  Callers that need to confirm no further operations will take effect MUST use `await` (§5.5) for
  synchronization.
- **Open-required synchronous semantics**: Operations that create resources, create topology, return
  a snapshot of resource-owned data, or otherwise require an open receiver MUST signal a
  closed-resource error synchronously when invoked after the receiver has accepted close. They MUST
  NOT return an inert resource or stale result. The required open-guarded operations in this
  specification are `Circuit.conduit`, `Circuit.bank`, `Circuit.basin`, `Circuit.cell`,
  `Circuit.pipe` (all forms), `Circuit.port`, `Circuit.pin`, `Circuit.sink`, `Circuit.ticker`,
  `Circuit.subscriber`, `Source.subscribe`, and `Bank.get`; projections MAY designate additional
  operations as open-guarded. The error MUST identify the receiver's subject. When rejection is caused by a closed
  substrate argument, such as subscribing with a closed subscriber, the error MUST also identify that
  argument's subject.

Types satisfying the Resource contract defined by this specification: Circuit, Source (and therefore
its subtype Conduit), Subscriber, Subscription, Bank, and Ticker. Because Conduit is a Source, it is
closeable — closing a conduit releases its pipe pool and cancels any outstanding subscriptions to
it, subject to the post-close operation semantics above. Closing a Bank closes all conduits it has
materialized; conduits created independently are unaffected.

Scope has its own closeable lifecycle contract (§9.2) but is not part of the universal Resource
contract unless a projection explicitly chooses to model it that way. Portable code MAY rely on
`Scope.close()` and the ordering semantics in §9.2; it MUST NOT assume that Scope exposes
`Resource.closeAwait()`. The spec does not mandate any specific projection mechanism for lifecycle
roles: a projection MAY realize Resource via a dedicated interface, and MAY realize Scope via the
host language's standard closeable type (e.g., Java's `AutoCloseable`) or any equivalent affordance.
Projections MAY provide additional resource types that satisfy the Resource contract. See §16.2 for
the authoritative capability matrix.

### 9.2 Scope

A **Scope** provides hierarchical, automatic resource management. When a scope closes, all resources
registered with it close automatically in reverse registration order (last registered, first
closed).

Scopes have two states: open and closed. Transition to closed is terminal.

Scope operations:

- **Register**: Adds a resource to the scope's management. The resource will close when the scope
  closes. Registering the same resource instance more than once MUST be a safe no-op. Returns the
  resource for fluent usage.
- **Closure**: Creates a single-use, block-scoped resource handle. The closure's `consume` operation
  executes a block with the resource, then closes the resource when the block exits. Single-use —
  each closure manages exactly one execution.
- **Child scope**: Creates a nested scope within the current scope. Closing the parent closes all
  children.

Close semantics for scopes:

- All registered resources MUST close in reverse registration order.
- All child scopes MUST close.
- The scope transitions to closed state.
- If a resource signals an error during close, the error MUST be suppressed and remaining resources
  MUST still close.
- Close is idempotent.

### 9.3 Circuit Lifecycle

Circuit closure proceeds as follows:

1. The circuit is marked as closed (idempotent flag). New emissions from caller contexts MUST NOT be
   processed. Per the post-close operation semantics (§9.1), post-close emissions MUST NOT throw
   synchronously in the caller context. An implementation MAY silently drop them, report them via a
   side channel (e.g., a circuit-state event or fault emission on the circuit context), or use any
   other mechanism that prevents processing without surfacing an error to the caller context. An
   implementation MUST document its post-close emission behavior.
2. The circuit context is signaled to terminate.
3. Emissions already admitted at the time of marking MAY or MAY NOT complete — this is explicitly
   implementation-defined. An implementation MUST document its choice.
4. After the circuit context terminates, conduits and their channels are released.

Close is non-blocking — it marks the circuit for closure and returns immediately.

**Portable quiescence**: The portable way to guarantee that all pending work completes before
shutdown is to stop submitting new work, then call `await()` before `close()`. After `await()`
returns, all work accepted before the await barrier has completed (§5.5). A subsequent `close()`
then has no prior accepted work left to drain, making the drain question moot. Implementations
SHOULD treat this as the recommended shutdown sequence.

After the circuit reaches its terminal closed state, `await()` MUST return immediately without
suspending. If close has been accepted but the circuit context is still drainable, a projection MAY
allow `await()` to wait for already-accepted lifecycle and barrier work to complete.

## 10. Conduit Pattern

### 10.1 Conduit as Pool

A conduit provides name-based pipe lookup with caching. Requesting a pipe by name either returns the
existing pipe or creates a new one. The identity guarantee holds: pipes with the same name MUST be
canonically identical (§1.2).

This identity guarantee ensures stable routing — all emissions for a given pipe name flow through
the same emission point to the same subscribers.

A **Pool** is a composable name-based view over a namespace of values. The base operation
`get(name)` returns the instance for a given name, creating one on first access and caching
subsequent lookups. A conduit is a pool whose values are pipes.

The `pool(fn)` operation creates a derived pool that applies a transformation function to each
result, caching the transformed result per name. Derived pools enable domain-specific wrappers
(e.g., `conduit.pool(Sensor::new)`) and decorated pipes (e.g., `conduit.pool(fiber::pipe)`), while
preserving the identity guarantee: the same name in a derived pool MUST always yield the canonically
identical transformed result.

**Function contract**: The transformation function MUST return a non-absent value. If the function
returns absence, the derived pool's `get(name)` MUST signal an absence violation error (§15.2) for
that name. The function is invoked **at most once per name**: the cached outcome — successful
transformed result, returned-absence rejection, or thrown error — is replayed for every subsequent
`get(name)` for the same name without re-invoking the function. From the caller's perspective,
derived-pool failures are deterministic per name; retry requires obtaining a fresh derived pool.

**Root pools**: A pool can also be constructed directly from the cortex factory operation (`pool`)
with a factory function from names to values, without an underlying conduit. The base operation and
function contract above apply unchanged: the factory is invoked at most once per name, the cached
outcome — value, returned-absence rejection, or thrown error — is replayed for subsequent lookups,
and retrieval MUST be thread-safe. A root pool provides lookup over the values it materializes but
does not own them: it is not a resource, has no close operation, and cascades no lifecycle to
materialized values — materialized resources that require ownership belong in a Bank (§10.4). Root
pools compose with the same derived-view mechanism (`pool(fn)`) and pool-accepting operations (such
as pool-backed subscribers) as conduit-backed pools.

### 10.2 Conduit as Source

A conduit is also a source (see §7). Subscribing to a conduit enables dynamic discovery of its named
pipes. The subscriber callback is invoked lazily when named pipes receive their first emission,
providing the pipe's subject and a registrar for downstream pipe attachment.

### 10.3 Conduit Creation and Routing

A conduit is created from a circuit with an emission type classifier (see §4.5). Projections MAY
provide an additional **routing** parameter controlling how emissions propagate within a conduit's
name hierarchy:

- **Per-pipe routing** (default): Emissions are dispatched only to subscribers of the target named
  pipe.
- **Hierarchical routing** (optional extension): Emissions propagate from the target named pipe
  upward through all ancestor names in the hierarchy, leaf-first, enabling hierarchical observation
  patterns without explicit subscriber wiring at each level.

Hierarchical routing is an OPTIONAL capability. Implementations that do not provide it MUST behave
as if per-pipe routing were always in effect.

### 10.4 Bank

A **Bank** is a closeable name-indexed holder for resources of a uniform kind. A bank is obtained
from a circuit factory operation (`bank`) and provides a stable shared mapping from names to
conduits that share the same emission type and routing behavior.

**Materialization semantics**: `Bank.get(name)` creates the corresponding resource on first access
and returns the cached instance on all subsequent lookups of the same name. The identity guarantee
holds: for any name within a bank, repeated `get(name)` calls MUST return canonically identical
resources (§1.2). This is the same guarantee as conduit pipe pooling (§10.1), applied at the conduit
level rather than the pipe level.

**Ownership semantics**: Every resource materialized by a bank is owned by that bank. Closing the
bank closes those resources; the close order is implementation-defined unless a projection documents
a stronger guarantee. Resources created independently through other factories — e.g., via
`Circuit.conduit` directly — are not affected by the bank's close.

**Open-required**: `Bank.get(name)` is an open-required operation (§9.1). Calling it after the bank
has accepted close MUST signal a closed-resource error synchronously.

**Thread safety**: Bank implementations MUST be thread-safe for concurrent retrieval.

**Relationship to Pool**: Pool (§10.1) is the composable derived-view mechanism over a conduit's
existing pipe namespace. Bank is a separate factory-level concept: it creates and owns conduits on
demand, scoped to a named application component. Both are concrete implementations of the abstract
Lookup base (§16.3).

### 10.5 Sink

A **Sink** is the output-closed dual of a conduit: a pool of named pipes whose destination is a
single endpoint pipe fixed at creation, instead of a source that subscribers discover. A conduit
leaves its output open — emissions are dispatched to whatever subscribers attach at runtime, each
receiving the raw value. A sink inverts this: the destination is supplied once, at creation, as a
`Pipe<Capture<E>>`, and there is no source to subscribe to.

A sink is created from a circuit with the endpoint pipe and an optional name. The emission type `E`
is determined by the endpoint's capture type, so no separate type classifier is required; and
because a single endpoint receives every emission, a sink takes no routing parameter.

**Channel behavior**: A sink is a pool (§10.1) — `get(name)` returns a cached `Pipe<E>` per name and
`pool(fn)` derives cached views, including pre-capture transformation and filtering via
`pool(flow::pipe)` / `pool(fiber::pipe)`. Each value emitted into a channel pipe is wrapped as a
capture (§11.1) and forwarded to the endpoint. The capture's **subject** is the channel pipe
(`Subject<Pipe<E>>`, taken after any derived-pool processing), its **current** is the emitting
execution context (§5.2) — the caller for an ingress emission, the circuit context for a transit
cascade (§5.3) — and its **state** defaults to the empty state. A value suppressed by a derived flow
or fiber never reaches the channel and so produces no capture. Because a sink is a pool of pipes, it
MAY be passed directly to the pool-backed subscriber factory (§7.2):
`source.subscribe(circuit.subscriber(name, sink))` wires each discovered source channel to the
same-named sink channel.

**Producing captures**: A sink is a capture-producing pool of channels — each capture is minted at
the sink channel the value passes through, which owns and stamps it (as a conduit pipe owns its
emissions). Because only the runtime can observe the per-emission emitting context, the minted
capture carries the per-emission **current** context: a subscriber can recover a channel's subject
(bound once at registration) but not the emitting context of each individual emission, which is
execution state observable only at the emission point.

**Endpoint**: The endpoint MAY be any `Pipe<Capture<E>>` and need not belong to the sink's circuit.
Because a capture is an out-of-execution-scope record, forwarding it to another circuit is an
ordinary cross-circuit emission; when the endpoint belongs to the sink's own circuit, same-circuit
dispatch applies (§6.3). The endpoint is not owned by the sink.

**Lifecycle**: A sink is a substrate, not a resource — it owns nothing that requires release and has
no close operation. Its channels are ordinary circuit pipes: valid while the owning circuit is open
(§9.3) and reclaimed by the garbage collector once unreachable. The endpoint's lifecycle is
independent of the sink. The `sink` factory itself is open-required (§9.1): creating a sink on a
circuit that has accepted close MUST signal a closed-resource error synchronously (§16.3).

**Flow control**: A sink exposes no flow-control or disable operation, and requires none — gating
composes around it with the same Fiber/Pipe operators used elsewhere, with the sink unaware. A
filtering Fiber on the **input** (via `pool(fiber::pipe)`, e.g. `Fiber.guard`) suppresses values
before they reach a channel, so no capture is minted; a Fiber wrapping the **endpoint** before
construction filters or rate-limits captures before the real endpoint. Stopping flow entirely is a
property of the producers and the subscription: cease emitting, close the bridging subscription when
the sink is fed from a source, or close the owning circuit.

## 11. Derived Structures

This section defines the secondary substrate types built on the core emission and execution model:
state holders (Cell, Basin, Port, Pin), context tokens (Current), and scheduled emitters (Ticker).

### 11.0 State Holder Capability Model

Three state holders — `Cell` (§11.2), `Port` (§11.5), and `Pin` (§11.6) — together cover the
spectrum of circuit-owned state authority. They exist as distinct types, not as one type with
multiple methods, because each grants a different **capability bundle** to its holder. The split
supports principle-of-least-authority composition: callers receive the narrowest authority needed
for their role, with no implicit access to operations they didn't ask for.

| Type   | Read authority                       | Write authority                      | Access mode           |
|--------|--------------------------------------|--------------------------------------|-----------------------|
| `Cell` | direct, any-thread (`get()`)         | queued replace via `pipe()`          | mixed                 |
| `Port` | **none**                             | queued replace / transform / emit    | queued, any-context   |
| `Pin`  | direct, owner-context only (`get()`) | direct, owner-context only (`set()`) | immediate, owner-only |

Specific operations narrow further:

- `Cell.pipe()` returns a stable `Pipe<E>` that grants **replace-without-read** authority and is
  irrevocable for the cell's lifetime (the same instance is returned on every call). Handing the
  pipe to a holder grants update authority without read; revocation is achieved by closing the
  owning circuit, not by reissuing the pipe.
- `Port.update(fn)` and `Port.update(arg, fn)` grant **transform-without-read** authority: the
  function runs inside the owning circuit context, reads the current value, and writes the result,
  but the caller never observes the value. This is the operation that distinguishes Port from
  `Cell.pipe()` (which only supports whole-value replacement).
- `Pin.get()` / `Pin.set()` are **owner-context immediate**: invocation from any other context
  raises an illegal-context-use error (§15.1). Pin's safety comes from confinement, not
  synchronization — the guard prevents cross-thread access entirely rather than coordinating it.

Consolidating these surfaces onto a single type would force callers needing one authority bundle to
over-receive others, making it harder to express least-authority composition and making mutable
circuit-confined state easier to leak outside the circuit boundary. The three-type separation is the
design's enforcement mechanism for §6.1.1 in the state-holder surface.

### 11.1 Basin

A **Basin** is a circuit-owned, bounded in-memory buffer of values fed through a pipe. It is the
multi-value sibling of a Cell (§11.2): where a cell holds the single latest value, a basin retains
the most recent `capacity` values, in emission order. It is created from a circuit with a fixed
positive capacity; an invalid capacity (< 1) MUST be signalled synchronously as an invalid-argument
error, and creation on a closed circuit MUST signal a closed-resource error. A basin holds whatever
type is emitted through its pipe; it does not itself mint captures.

A **Capture** is an immutable provenance envelope for a single emission. It bundles:

- **emission** — the emitted value of type `E`.
- **subject** — the subject of the emitting channel pipe, typed by the pipe role
  (`Subject<Pipe<E>>`, its value type and the captured value type `E` coincide), distinguishing it
  from the subjects of other components such as conduits or sources.
- **current** — the subject of the emitting execution context (`Subject<Current>`): the caller
  context for a value admitted from outside the circuit (an external thread, or another circuit's
  worker for a cross-circuit emission), or the circuit context itself for a value emitted by a
  transit cascade (§5.3) — so for a cascade it equals `Circuit.current()`. It is the *immediate*
  emitter, not the ultimate root of a cascade chain. Circuit-internal mechanisms are attributed to
  the owning circuit — a ticker emission (§11.4) reports the owner circuit's context, not the
  ticker's scheduling context.
- **state** — a `State` (§8) of implementation- and configuration-dependent per-emission measures
  (for example a processing-time reading). What, if anything, is published is provider-defined; the
  default is the empty state. Consumers MUST treat any particular slot as optional and MUST NOT
  assume the state is non-empty.

The context is exposed as a subject, not a live `Current`, so a Capture is a pure
out-of-execution-scope reference record — safe to retain, compare, and stream without a handle to
any live counterpart, mirroring the principle that the pipe is identified by its subject rather than
the live pipe.

Captures are produced by a `Sink` (§10.5): a capture-producing pool of channels — the output-closed
dual of a conduit — that mints each capture at the channel it passes through and forwards it to a
fixed endpoint. A basin buffers values, and buffers captures when its pipe is a sink's endpoint (a
`Basin<Capture<E>>` fed by `sink`). When a sink is fed from a source, the sink mints during the
source's transit cascade (§5.3), so a source-fed capture's `current` is the circuit and its
`subject` is the sink channel (mirroring the source channel's name with its own identity).

A basin is fed by emitting into its pipe; each accepted value is appended to the buffer on the
circuit worker thread, evicting the oldest retained value when at capacity. The buffer is read by
drain, which has the following behavioral requirements:

- **Drain to a pipe**: drain forwards the retained values to a target pipe, in emission order, on
  the circuit worker thread, and then clears the buffer. The call enqueues the drain and returns
  immediately; no buffered state crosses the thread boundary. When the target shares the basin's
  circuit the forward is worker-local transit work (§5.3); otherwise each value is delivered as a
  cross-circuit emission. To consume with arbitrary logic, drain into a circuit pipe built from a
  receptor.
- **Ordering guarantee**: values MUST be forwarded in emission order — the order in which the
  circuit processed them.
- **Bounded retention**: a basin MUST NOT retain more than `capacity` values. When a value is
  accepted while the buffer is full, the implementation MUST evict the oldest retained value before
  recording the new one.
- **Clearing**: drained values are evicted; a subsequent drain forwards only values retained since
  the previous drain.
- **Post-close**: a drain is itself a queued emission, not a lifecycle operation — a drain enqueued
  after the owning circuit has accepted close is silently dropped (§9.1), not guaranteed to run.

A basin is a Substrate, not a Resource — it owns nothing requiring release. When a basin is fed from
a source through a sink, the feeding lifecycle is governed by the bridging subscription; closing
that subscription stops the feed while the retained values remain drainable.

### 11.2 Cell

A **Cell** is a circuit-owned, initialized, single-slot state holder. A cell is created from a
circuit with a non-null seed value and exposes two operations: an `emit`-only update pipe and a read
accessor that returns the current published value. Cells provide the specified mechanism for
converting an emission stream into a latest-value handle that readers can poll without subscribing.

A cell has the following properties:

- **Circuit-bound updates**: The update pipe submits emissions to the owning circuit. Each accepted
  emission is processed in circuit-context confinement (§5.1) and becomes the cell's published
  value. The cell is not itself a Receptor or Pipe — its update capability is exposed only through
  the pipe returned by the cell's `pipe` accessor.
- **Latest-wins semantics**: A cell stores at most one value. Each accepted emission replaces the
  previously published value. The cell defines no transformation, accumulation, or windowing
  semantics — callers compose those upstream via Fiber or Flow and terminate the recipe with the
  cell's update pipe.
- **Initialized state**: A cell is always created with a non-null seed value. Reads before the first
  accepted emission has been processed MUST return the seed. The cell has no absent state: `get()`
  is total and never returns an absent value (§1.2). Applications that need "no value yet" semantics
  MUST model that as a domain value (such as an `Optional<E>`-like wrapper or a domain sentinel) and
  seed the cell with the absent value of that domain.
- **Safe publication**: After the circuit context has processed an update, subsequent reads that are
  ordered after that processing (by the happens-before relations in §5.4.1, in particular the await
  barrier of §5.5) observe the published value. The publication guarantee applies to the value
  reference and to state reachable from it at publication time. Implementations are not required to
  defensively copy mutable value graphs; callers MUST treat published values as immutable snapshots
  or supply their own synchronization when `E` is mutable.
- **Read concurrency**: The read accessor MAY be invoked from any execution context. It is
  observational — it does not interact with the circuit's admission path and does not order itself
  against pending or in-flight updates. Callers that need to observe the effect of an emission MUST
  use `Circuit.await()` (§5.5) before reading.

A cell is created with a non-null seed value. A cell MAY be created with an explicit name; when no
name is supplied, the implementation uses the owning circuit's name. Named factory calls do not
imply pooling — two named factory calls with the same name MUST return distinct handles with
distinct subject identities.

Cell is a substrate component — it carries a `subject()` — but it is not a Source and not a
Resource. A cell's lifetime is bounded by its owning circuit: closing the circuit releases the cell
along with any other circuit-owned resources.

### 11.3 Current

A **Current** represents the execution context from which substrate operations originate. It is a
context-local reference — each execution context has at most one current, and obtaining it is
observational (it does not create or modify state).

Each execution context has exactly one Current, interned for that context's lifetime. Current
references MAY be retained and shared across execution contexts for identity comparison and for
reading their immutable subject. Calling `Cortex.current()` answers "which execution context is
calling now"; a previously captured Current answers "which execution context did this token
identify" and does not update when inspected from another context.

`Circuit.current()` returns the Current of the circuit context owned by that circuit. The returned
reference MUST remain stable for the circuit's lifetime. Comparing it with `Cortex.current()` is the
specified way to detect re-entrancy into the circuit context before invoking operations that are
illegal from that context, such as `await` or `pulse`.

Current is identity-bearing (it has a subject) but is not required to be serializable or stable
beyond the lifetime of its execution context. It is purely observational — it identifies substrate
execution contexts, not external systems.

The projection mechanism is runtime-dependent — `Thread.currentThread()` in Java, goroutine-local
state in Go, coroutine context in Kotlin, isolate reference in Dart, or equivalent. The mechanism is
not specified; only the identity and temporal guarantees are REQUIRED.

### 11.4 Ticker

A **Ticker** is a circuit-owned periodic emitter. A ticker is created from a circuit with a strictly
positive interval and a target pipe, and drives a regular sequence of emissions into that pipe
without any caller action after creation.

A ticker has the following properties:

- **Monotonic sequence emissions**: Each tick emits a sequence number into the target pipe. The
  first tick emits zero, and each subsequent tick emits the previous value plus one. The sequence is
  gap-free and strictly increasing. Sequence numbers carry no wall-clock meaning — they are an
  ordinal count of ticks. A receptor detects missed or delayed ticks from stalls in the rate at
  which the sequence advances, not from the values themselves.
- **Circuit-admitted delivery**: Each tick is submitted through the owning circuit's admission path
  (§5.6) and is ordered with all other admitted work under the circuit's deterministic ordering
  guarantee (§5.1). The target receptor executes in circuit-context confinement (§5.1).
- **Fixed-rate scheduling**: Ticks target points on a grid, each one interval after the last. A tick
  that is scheduled late MUST NOT shift the grid — the delay of one tick MUST NOT be carried into
  the scheduled time of subsequent ticks. Over a run free of stalls, the mean interval between ticks
  MUST converge to the requested interval; scheduling error MUST NOT accumulate.
- **Bounded catch-up**: If scheduling falls more than one interval behind the grid, the ticker MUST
  re-anchor the grid to the current time rather than emitting the skipped grid points in immediate
  succession. A stall therefore produces a one-time phase shift in the grid, never a burst of ticks;
  at most one tick MAY be emitted early to rejoin the grid. Re-anchoring affects scheduling only —
  the sequence numbering remains gap-free across a re-anchor.
- **Submission-timing scope**: The scheduling guarantee governs the time at which a tick is
  submitted to the circuit. Any latency between submission and delivery introduced by the circuit's
  admission path and dispatch ordering (§5.1, §5.6) is not part of the ticker's timing guarantee.
- **No tick shedding**: A ticker does not drop ticks. If the target receptor processes ticks more
  slowly than the interval, admitted-but-unprocessed work accumulates under the circuit's admission
  capacity rules (§5.6). Callers that need to coalesce or rate-limit under backlog compose the
  target with Fiber rate-control operators (§6.2.3) rather than relying on the ticker to shed load.
- **Failure isolation**: An error raised by the target receptor is isolated by the owning circuit as
  an external callback failure (§15.4) and MUST NOT stop the ticker; subsequent ticks continue to be
  emitted.

A ticker is created with a non-absent target pipe and a strictly positive interval. A zero or
negative interval MUST signal an illegal-argument error (§15.1). The target pipe MUST be a
provider-compatible pipe (§15.1 provider mismatch). A ticker MAY be created with an explicit name;
when no name is supplied, the implementation MUST assign a valid non-empty default name (§16.3).

Tickers are resources (§9.1) — closing a ticker stops further emissions, though ticks already
admitted to the circuit before the close MAY still be delivered. Closing the owning circuit also
stops the ticker: once the circuit has accepted close, further tick submissions are rejected (§9.3).

### 11.5 Port

A **Port** is a circuit-owned, initialized state holder that grants queued mutation authority to its
holder without exposing direct read access. A port is created from a circuit with a non-null seed
value and exposes four queued operations: replace, transform, transform-with-argument, and emission
of the current value to a downstream pipe. The current value is never observable to the port's
holder; transformations execute inside the owning circuit context and the result remains
circuit-confined.

A port has the following properties:

- **Initialized state**: A port is always created with a non-null seed value. The seed is the port's
  initial stored value; transformations passed to `update` receive this value on first invocation.
  Ports have no absent state — the stored value is total. Applications that need "no value yet"
  semantics MUST model that as a domain value (such as an `Optional<E>`-like wrapper or a domain
  sentinel) and seed the port with the absent value of that domain.
- **Queued operations**: Every port operation is queued. A call from any caller context is admitted
  through the owning circuit's admission path (§5.6) as ingress work. A call from the owning circuit
  context is transit work (§5.3) and MUST NOT execute its state effect inline. A call from a
  different circuit context is ingress work for the owning circuit. No public port operation
  executes its state effect inline; this rule prevents port operations from jumping ahead of
  previously accepted transit work.
- **Local-call non-immediacy**: A port operation invoked from the owning circuit context returns
  before its state effect has been applied. Code executing immediately after the call in the same
  callback observes the *previous* stored value, not the new one. The effect is processed when the
  transit queue drains. Code that needs the mutation to be visible to subsequent statements in the
  same callback requires an immediate-mode state surface, which Port does not provide.
- **Deterministic ordering**: Port operations participate in the same deterministic ordering model
  as emissions (§5.1, §5.3, §5.4). If operation A is accepted before operation B on the same owning
  circuit, A's effect is processed before B's effect.
- **Replacement**: `replace(value)` queues a whole-value replacement. The replacement value MUST be
  present (§15.2).
- **Transformation**: `update(fn)` queues a transformation of the stored value. When processed, the
  function is invoked on the owning circuit context with the current present stored value. If the
  function returns a present value, the port stores it. If the function returns absence (§1.2) or
  throws, the port retains its previous value; the failure is isolated under the circuit's
  external-callback failure path (§15.4) — sibling operations, subsequent queued work, and the
  circuit's dispatch loop are unaffected, and the submitting caller does not observe the failure
  synchronously. When the failure is a null/absent return (a framework-detected contract violation
  rather than a user-thrown exception), implementations SHOULD surface the failure as a structured
  fault carrying the port's subject and `operation = "update"`, so observers can distinguish
  framework-detected contract violations from arbitrary user-thrown exceptions (§15.4).
- **Argument transformation**: `update(arg, fn)` queues a transformation that carries an explicit
  argument supplied to the function on the worker thread. The argument MUST be present, crosses a
  context boundary, and SHOULD satisfy the emission safety contract (immutable, effectively
  immutable, or a managed substrate handle). Same failure-isolation semantics as `update(fn)`.
- **Emission**: `emit(pipe)` queues an emission of the port's current stored value to the supplied
  target pipe. Because a port is always initialized, emit always forwards one present value when the
  queued operation is processed. The emitted value MUST satisfy the emission safety contract. The
  target pipe MUST be a provider-compatible pipe (§15.1 provider mismatch).
- **Replacement-oriented, not transactional**: Port operations are replacement-oriented, not
  transactional rollback boundaries. Side effects or in-place mutations performed by a transform
  function before it throws are not undone. For this reason port values SHOULD be immutable or
  effectively immutable; an immediate-mode in-place mutation surface for circuit-confined mutable
  state is outside Port's contract.

A port is created with a non-null seed value. A port MAY be created with an explicit name; when no
name is supplied, the implementation uses the owning circuit's name. Named factory calls do not
imply pooling — two named factory calls with the same name MUST return distinct handles with
distinct subject identities.

Port factory calls are open-required operations (§9.1). After the owning circuit has accepted close,
port factories MUST signal a closed-resource error (§15.1) on the caller thread. Port operations on
a port whose owning circuit has accepted close follow the same post-close behavior as pipe emissions
(§9.3): they MUST NOT throw synchronously merely because the circuit has closed, and they MUST NOT
take effect after the circuit drops post-close queued work.

Port is a substrate component — it carries a `subject()` — but it is not a Source and not a
Resource. A port's lifetime is bounded by its owning circuit: closing the circuit releases the port
along with any other circuit-owned resources.

### 11.6 Pin

A **Pin** is a circuit-owned, initialized state handle that grants **immediate** inspect and mutate
access confined to the owning circuit's context. Where `Cell` (§11.2) grants any-thread read with
queued replace and `Port` (§11.5) grants queued mutation without read, a Pin grants synchronous
read/write directly into a circuit-owned slot — invoked from inside the owning circuit's worker
thread, returning before the next statement in the same callback executes.

A pin has the following properties:

- **Initialized state**: A pin is always created with a non-null seed value. `get()` is total and
  MUST NOT return an absent value (§1.2). Applications that need "no value yet" semantics MUST model
  that as a domain value (such as an `Optional<E>`-like wrapper or a domain sentinel) and seed the
  pin with the absent value of that domain.
- **Immediate access**: `get()` and `set(value)` execute immediately on the owning circuit's worker
  thread; no queueing, no admission overhead. A `set` followed by `get` in the same callback
  observes the new value — this is the defining contract that distinguishes Pin from Port.
- **Owner-context guard**: `get()` and `set(value)` MUST be invoked from the owning circuit's worker
  thread. Invocation from any other execution context — a caller context, another circuit's context,
  or the cortex root context — MUST raise an illegal-context-use error (§15.1). The principled
  detection idiom mirrors the re-entrancy check used for `Circuit.await()`
  and `Circuit.pulse()`: compare `Cortex.current()` against the pin's owning circuit's `current()`.
- **Immediate replacement**: `set(value)` replaces the stored value immediately on the worker
  thread. The replacement value MUST be present.
- **Mutable contents**: Within the owning circuit context, code MAY access and mutate a mutable
  object stored inside a pin in-place. That mutation is safe only because the access is
  circuit-confined. A value obtained from `get()` MUST NOT be retained, published, or mutated
  outside the owning circuit context unless that value independently satisfies the emission safety
  contract (§6.1.1).
- **Emission as managed handle**: A Pin is itself a managed substrate handle and MAY be emitted as a
  value through any pipe. Receivers outside the owning circuit may store, forward, or route the
  handle, but invoking `get()` or `set()` from a non-owner context raises the owner-context guard
  violation above. Dereferencing requires the handle to reach a pipe owned by the original circuit.
- **Not thread-safe through synchronization**: Pins are not made thread-safe through volatile or
  atomic access. Safety comes from confinement, not coordination. Cross-thread access from outside
  the owning circuit is prohibited by the guard, not synchronized.

A pin is created with a non-null seed value. A pin MAY be created with an explicit name; when no
name is supplied, the implementation uses the owning circuit's name. Named factory calls do not
imply pooling — two named factory calls with the same name MUST return distinct handles with
distinct subject identities.

Pin factory calls are open-required operations (§9.1). After the owning circuit has accepted close,
pin factories MUST signal a closed-resource error (§15.1) on the caller thread.

Pin is a substrate component — it carries a `subject()` — but it is not a Source and not a Resource.
A pin's lifetime is bounded by its owning circuit: closing the circuit releases the pin along with
any other circuit-owned resources.

## 12. Interning and Pooling

The specification uses two forms of instance management.

**Interning** applies to names and identifiers. Structurally equivalent names MUST resolve to the
same canonical entity (§1.2). Name comparison is canonical identity. This enables constant-time
lookup and minimal memory usage for repeated name references.

**Pooling** applies to pipes within a conduit. Pipes are pooled by name — the same name MUST always
resolve to a canonically identical pipe. This guarantees stable routing: all emissions for a given
name flow through the same pipe, and all subscribers bound to that name receive them.

## 13. Tenure

Every type in the specification carries a tenure annotation describing its retention behavior:

- **Interned**: Instances are pooled by key. The same key yields a canonically identical result
  (§1.2). The owning container maintains a reference.
- **Ephemeral**: Instances are not cached by their creator. Each creation is independent. Only
  external references keep them alive.
- **Anchored**: Tenure depends on attachment. Standalone instances are ephemeral; attached instances
  are retained by the substrate they are bound to.

Tenure classifications for the core types:

| Type         | Tenure    |
|--------------|-----------|
| Cortex       | Interned  |
| Circuit      | Ephemeral |
| Conduit      | Ephemeral |
| Sink         | Ephemeral |
| Bank         | Ephemeral |
| Cell         | Ephemeral |
| Port         | Ephemeral |
| Pin          | Ephemeral |
| Ticker       | Anchored  |
| Pipe         | Anchored  |
| Subscriber   | Anchored  |
| Subscription | Anchored  |
| Basin        | Ephemeral |
| Scope        | Anchored  |
| Name         | Interned  |
| Id           | Interned  |
| Subject      | Anchored  |
| State        | Anchored  |
| Slot         | Anchored  |
| Capture      | Ephemeral |
| Closure      | Ephemeral |
| Current      | Interned  |
| Pulse        | Ephemeral |
| Window       | Ephemeral |
| Run          | Ephemeral |
| Change       | Ephemeral |

## 14. Circuit-Dispatched Pipe Dispatch

Circuits can create pipes that dispatch emissions to a target pipe or receptor through the owning
circuit rather than by invoking the target directly on the caller context. This breaks synchronous
call chains and enables:

- Deep hierarchical propagation without stack overflow.
- Cyclic topologies (feedback loops, recurrent networks).
- Offloading work from caller contexts to the circuit context.

When creating a circuit-dispatched pipe where the target already belongs to the same circuit, the
target SHOULD be returned as-is — no additional indirection is added.

Implementations MAY use different internal queues or fast paths depending on whether the emission
originates from a caller context or the circuit context itself, provided the public ordering and
circuit-context confinement guarantees are preserved.

Per-emission processing is applied as a separate step via pipe composition (§6.2.6):
`fiber.pipe(circuit.pipe(target))` or `flow.pipe(circuit.pipe(target))` produces a pipe whose
emissions are filtered, transformed, and rate-limited before reaching the circuit-dispatched target.
All operator stages execute within the circuit context after emissions are accepted for processing.
Composition is orthogonal to circuit-dispatched delivery: any pipe (obtained from conduit `get`,
circuit `pipe`, or subscriber `register`) can serve as the target of `Fiber.pipe(pipe)` /
`Flow.pipe(pipe)` to add a processing stage upstream of it. A Cell can serve as the terminal target
through `Fiber.pipe(cell)` / `Flow.pipe(cell)`, which target the cell's update pipe.

## 15. Error Model

The specification defines an abstract error model with portable categories. Each category represents
a class of contract violation. Some categories require unconditional detection; others permit
undefined behavior where detection would degrade valid-path performance. The enforcement level for
each category is specified in §15.1. The signaling mechanism is projection-dependent — exceptions in
Java/C#/Python, error returns in Go, `Result` types in Rust, rejected futures in async runtimes, or
service-level error codes in networked projections. The categories are normative; the mechanism is
not.

### 15.1 Error Categories

| Category                 | Enforcement                               | When Signaled                                                                                                                                                                                                                                                                                                                                                                                       |
|--------------------------|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Provider mismatch**    | MUST detect                               | Objects from incompatible provider implementations are mixed — e.g., a name created by one cortex is passed to a circuit created by a different cortex; a target pipe from a different provider is passed to `Circuit.ticker` or `Port.emit`; a Cell, Pipe, Fiber, or Flow from a different provider is passed to a composition operation (`Fiber.pipe`, `Flow.pipe`, `Flow.fiber`, `Fiber.fiber`). |
| **Absence violation**    | MUST detect                               | A required parameter receives an absent value (§1.2) where one is not permitted.                                                                                                                                                                                                                                                                                                                    |
| **Illegal temporal use** | SHOULD detect (MUST detect for Registrar) | A temporal object (§6.4) is used after its callback scope has ended. Detection is RECOMMENDED in general; undefined behavior is acceptable where detection would degrade valid-path performance. **Registrar violations MUST be detected and signaled** — see §6.4.1.                                                                                                                               |
| **Illegal context use**  | SHOULD detect                             | An operation is invoked from a prohibited execution context — e.g., `await()` or `pulse()` called from within the circuit context (§5.5, §5.7). Detection is RECOMMENDED where feasible; if not feasible, the behavior is undefined.                                                                                                                                                                |
| **Configuration error**  | MUST detect                               | An error occurs while a fiber, flow, or derived pool recipe is configured or materialized. Eager configuration errors MUST propagate from the call that builds the recipe. Lazy configuration errors MUST propagate from the call that materializes it for a named pipe (e.g., `Pool.get(name)`), before any emission is dispatched through that materialization.                                   |
| **Closed resource**      | MUST detect                               | An open-required operation (§9.1) is invoked after the receiver has accepted close, or with a substrate argument that has accepted close. The error MUST be signaled synchronously before admitting work.                                                                                                                                                                                           |

### 15.2 Absence Constraints

Parameters that do not accept absent values are designated as **required**. Implementations MUST
signal an absence violation error when a required parameter receives an absent value. The detection
mechanism is projection-dependent — `NullPointerException` in Java, panic in Go/Rust, `TypeError` in
Python, or equivalent.

### 15.3 Signaling Requirements

Implementations MUST document how each error category maps to the projection's error mechanism. A
conformant implementation MAY use a single error type with category metadata, distinct error types
per category, or any representation that preserves the category distinction. The minimum requirement
is that consuming code can distinguish between the categories defined in §15.1.

For a closed-resource error, the signaled error MUST identify the receiver/context subject. If the
failed operation was rejected because a substrate argument was closed, the error MUST also identify
that argument's subject. Projections MAY expose this as structured call metadata, an exception
payload, an error value, or an equivalent mechanism.

### 15.4 External Callback Isolation

The specification distinguishes between **engine code** — the substrate implementation itself — and
**external code** — client-supplied values passed into the substrate as callbacks. External code
includes:

- A `Receptor` registered with a channel via a `Registrar`
- A `Subscriber` callback invoked during channel discovery
- An `onClose` callback attached to a subscription
- Any function value passed to a `Fiber` or `Flow` operator (predicates, comparators, binary
  operators, mappers, peek receptors, fire predicates, etc.)

External code executes within the circuit context but is not part of the engine. It may throw
exceptions for reasons the engine cannot anticipate — programming defects, out-of-memory,
third-party library failures. The specification pins only the safety invariants that any supervisory
strategy (breaker, guard, latch, quarantine) must be able to rely on. It does not mandate a recovery
policy.

An implementation MUST satisfy the following invariants at every point where external code is
invoked:

1. **Isolation**. An uncaught exception raised by external code MUST NOT propagate to the circuit's
   dispatch loop. The dispatch loop MUST remain running and MUST continue to process queued
   operations.

2. **Liveness**. An exception raised while dispatching an emission to one receptor MUST NOT prevent
   dispatch of that emission to other receptors registered on the same channel, nor prevent
   processing of subsequent queued operations (emissions, subscriptions, closes). An exception
   raised by a subscriber callback, registrar interaction, or onClose callback MUST NOT leave the
   circuit in a state that blocks forward progress for unrelated subscriptions.

3. **No silent success**. External code that terminates exceptionally has **failed**, not succeeded.
   A failing Fiber or Flow operator MUST NOT cause the stage chain to behave as if the operator had
   returned a valid value; the emission is considered dropped for that receptor chain. A failing
   subscriber callback is still considered consumed for that subscription/channel pair: registration
   calls completed before the failure remain registered, and the callback MUST NOT be retried for
   that subscription/channel pair. The failure itself MUST remain observable according to the
   implementation's documented failure reporting mechanism.

4. **Observability**. Implementations MAY accept a structured fault receptor when a circuit is created.
   When supplied, that receptor receives every isolated external-callback failure, attributed to the
   closest meaningful substrate subject known at the isolation boundary and otherwise to the circuit.
   The structured fault retains the original failure as its cause. A failure raised by the fault
   receptor is isolated and is not reported recursively. Without a supplied receptor, implementations
   MAY use a documented default reporting mechanism; applications SHOULD select explicitly when observation matters.

**Framework-detected contract violations.** When the engine itself detects that external code has
violated a documented non-null or non-absent return contract — for example, an operator function
passed to `Port.update(fn)` returning `null` where a present value is required (§11.5) — the engine
MAY raise a structured fault on the worker thread carrying the receiver's subject and the operation
name. This is the asynchronous counterpart to the synchronous `Fault` raised on the caller thread
for provider mismatch, closed-resource, and similar caller-thread violations (§15.1):
the receiver and operation are identified the same way, but the failure travels through the
external-callback isolation channel above rather than back to the synchronous call site.
Asynchronous faults of this kind MUST satisfy invariants 1–4 above. Implementations that surface
faults to a structured observer SHOULD use a recognizable fault type so observers can distinguish
framework-detected contract violations from arbitrary exceptions thrown by user code.

Implementations MAY apply these guards only at the **trust boundary** where external code enters the
engine. Wrapping engine-internal implementations of callback interfaces (e.g., internal `Pipe` or
`Receptor` instances used as dispatcher plumbing) is neither required nor encouraged — the overhead
would be spent on code that cannot reach external failures. Which specific instances are at the
trust boundary is implementation-defined.

Recovery strategies — quarantining a receptor after repeated failures, restarting a Fiber/Flow
stage, routing failed emissions to a dead-letter source, tripping a circuit breaker — are
deliberately out of scope for this specification. These concerns belong to extension layers built on
top of a conformant implementation. The contract above exists to guarantee that such layers can be
built without having to defend against engine corruption.

## 16. Conformance

A conformant implementation MUST satisfy the behavioral requirements, implement the required types
and operations, and enforce the temporal contracts defined in this specification.

### 16.1 Behavioral Requirements

A conformant implementation MUST satisfy all of the following behavioral contracts:

1. **Circuit-context confinement**: All emissions, fiber/flow operations, and subscriber callbacks
   within a circuit MUST execute within a single sequential execution context owned by that circuit.
2. **Deterministic ordering**: Emissions MUST be observed in strict admission order. Earlier
   emissions MUST complete before later ones begin.
3. **Ingress/transit priority**: Transit work MUST drain completely before the next ingress item is
   processed, regardless of the implementation's internal queue representation.
4. **Name interning**: Structurally equivalent names MUST resolve to canonically identical names
   (§1.2).
5. **Name depth floor**: An implementation MUST accept names of at least 16 segments, whether
   constructed in a single operation or reached by successive extension, and MUST document any
   maximum it imposes. Exceeding a documented maximum MUST signal an illegal-argument error rather
   than truncating the name (§4.1).
6. **Pipe pooling**: Named pipes with the same name within a conduit MUST be canonically identical
   (§1.2).
7. **Lazy rebuild**: Subscription changes MUST propagate to named pipes on their next emission via
   version-tracked rebuild.
8. **Temporal contracts**: Registrar instances MUST be valid only during the subscriber callback in
   which they are provided. Window instances MUST be valid only during the receptor or Fiber
   callback in which they are observed. Implementations MUST detect and signal Registrar temporal
   contract violations (see §6.4.1). For other temporal types, enforcement follows the general
   SHOULD-detect framework.
9. **Idempotent close**: All resource close operations MUST be idempotent and concurrency-safe.
10. **Closed-resource faults**: Open-required operations invoked on a resource that has accepted
    close MUST signal a closed-resource error synchronously and identify the receiver subject (§9.1,
    §15.3).
11. **Scope ordering**: Resources registered with a scope MUST close in reverse registration order.
12. **State immutability**: State operations MUST preserve immutability — prior instances MUST
    remain unchanged. Implementations MAY return the same instance for semantically equivalent
    writes (e.g., upserting a slot whose name, type, and value match an existing entry).
13. **Happens-before guarantees**: Implementations MUST provide the happens-before relations defined
    in §5.4.1 through appropriate synchronization mechanisms.
14. **Illegal circuit-context calls**: Calling `await`, `pulse`, or `closeAwait` from within the
    circuit context MUST signal an illegal context use error per the enforcement level in §15.1
    (SHOULD detect; MUST signal when detection is feasible).
15. **External callback isolation**: Uncaught exceptions from client-supplied callbacks (receptors,
    subscriber callbacks, onClose handlers, Fiber/Flow operator functions) MUST NOT propagate to the
    circuit's dispatch loop. Dispatch MUST continue to sibling receptors on the same channel and to
    subsequent queued operations. See §15.4.
16. **Subscriber callback consumption**: A subscriber callback is invoked exactly once for each
    active subscription/channel pair. If it fails, completed registrations remain registered and the
    callback is not retried for that pair.
17. **Operator state ownership**: Client-supplied emission-time Fiber and Flow operator functions
    MUST NOT carry shared mutable emission-processing state. A Fiber or Flow value MAY be
    materialized across multiple conduits and multiple circuits; operator function instances are
    shared across materializations. Attachment-time factories and Suppliers MAY allocate fresh state
    or read immutable / thread-safe configuration, but MUST NOT use captured mutable state as shared
    per-emission state. Implementations MUST rely on framework-owned per-materialization state for
    stateful operators (Reduce's accumulator, Relate's previous input, Diff's last-emitted value,
    Scan's state slot, Window's rolling buffer, etc.) and MUST NOT require operator functions to
    carry state of their own. See §6.2.
18. **Bank resource identity**: For any name within a bank, repeated `Bank.get(name)` calls MUST
    return canonically identical resources (§1.2, §10.4). Closing the bank MUST close exactly those
    resources it has materialized and MUST NOT affect resources created through other factories.
19. **Ticker scheduling**: A ticker MUST schedule ticks at a fixed rate — a late tick MUST NOT shift
    the scheduling grid, and scheduling error MUST NOT accumulate across ticks. When scheduling
    falls more than one interval behind the grid, the ticker MUST re-anchor rather than emitting the
    skipped grid points as a burst. Tick sequence numbering MUST remain gap-free regardless of
    scheduling re-anchoring. See §11.4.

### 16.2 Required Types

A conformant implementation MUST provide concrete realizations of the following abstract types:

| Type         | Category     | Description                                |
|--------------|--------------|--------------------------------------------|
| Cortex       | Factory      | Root entry point and factory               |
| Circuit      | Execution    | Sequential execution engine                |
| Conduit      | Routing      | Pipe factory, pool, and source             |
| Sink         | Routing      | Capture-producing pool with fixed endpoint |
| Lookup       | Routing      | Abstract base for name-indexed retrieval   |
| Pool         | Routing      | Composable name-based view                 |
| Bank         | Routing      | Closeable name-indexed resource holder     |
| Routing      | Routing      | Conduit dispatch mode selector             |
| Pipe         | Emission     | Emission carrier                           |
| Cell         | State        | Circuit-owned single-slot state holder     |
| Port         | State        | Circuit-owned queued-mutation state slot   |
| Pin          | State        | Circuit-confined immediate state handle    |
| Ticker       | Execution    | Circuit-owned periodic emitter             |
| Receptor     | Consumption  | Emission callback                          |
| Fiber        | Pipeline     | Same-type per-emission operator chain      |
| Flow         | Pipeline     | Type-changing composition surface          |
| Window       | Pipeline     | Callback-scoped rolling emission window    |
| Run          | Pipeline     | Per-admission run-length envelope          |
| Change       | Pipeline     | Run-boundary change envelope               |
| Subscriber   | Subscription | Pipe discovery callback                    |
| Subscription | Subscription | Cancellable subscription handle            |
| Registrar    | Subscription | Pipe attachment handle (temporal)          |
| Source       | Subscription | Subscribable event origin                  |
| Basin        | Derived      | Bounded value buffer                       |
| Capture      | Derived      | Emission provenance envelope               |
| Current      | Context      | Execution context identity token           |
| Pulse        | Diagnostics  | Circuit diagnostic timing snapshot         |
| Name         | Identity     | Hierarchical interned identifier           |
| Id           | Identity     | Opaque unique token                        |
| Subject      | Identity     | Component identity record                  |
| Extent       | Identity     | Hierarchical containment structure         |
| State        | Data         | Immutable slot collection                  |
| Slot         | Data         | Named typed value                          |
| Scope        | Lifecycle    | Hierarchical resource manager              |
| Closure      | Lifecycle    | Block-scoped resource handle (temporal)    |
| Resource     | Lifecycle    | Closeable component                        |

Types marked **(temporal)** carry temporal contracts as defined in §6.4. The specific lifetime scope
(callback-scoped or single-use-scoped) is specified per type in §6.4.

#### Type Capability Matrix

The following matrix is the authoritative inventory of which required types are substrate components
(expose `subject()`), subscribable origins (satisfy the Source contract), closeable resources
(satisfy the Resource contract), and temporal. All other sections describing these capabilities are
derived from this table.

| Type         | Substrate (has subject) | Source | Resource (closeable) | Temporal |
|--------------|:-----------------------:|:------:|:--------------------:|:--------:|
| Cortex       |           ✓            |        |                      |          |
| Circuit      |           ✓            |        |          ✓          |          |
| Conduit      |           ✓            |   ✓   |          ✓          |          |
| Source       |           ✓            |   ✓   |          ✓          |          |
| Bank         |           ✓            |        |          ✓          |          |
| Sink         |           ✓            |        |                      |          |
| Routing      |                         |        |                      |          |
| Cell         |           ✓            |        |                      |          |
| Port         |           ✓            |        |                      |          |
| Pin          |           ✓            |        |                      |          |
| Ticker       |           ✓            |        |          ✓          |          |
| Pipe         |           ✓            |        |                      |          |
| Current      |           ✓            |        |                      |          |
| Scope        |           ✓            |        |                      |          |
| Basin        |           ✓            |        |                      |          |
| Subscriber   |           ✓            |        |          ✓          |          |
| Subscription |           ✓            |        |          ✓          |          |
| Registrar    |                         |        |                      |    ✓    |
| Receptor     |                         |        |                      |          |
| Fiber        |                         |        |                      |          |
| Flow         |                         |        |                      |          |
| Window       |                         |        |                      |    ✓    |
| Run          |                         |        |                      |          |
| Change       |                         |        |                      |          |
| Lookup       |                         |        |                      |          |
| Pool         |                         |        |                      |          |
| Capture      |                         |        |                      |          |
| Pulse        |                         |        |                      |          |
| Name         |                         |        |                      |          |
| Id           |                         |        |                      |          |
| Subject      |                         |        |                      |          |
| Extent       |                         |        |                      |          |
| State        |                         |        |                      |          |
| Slot         |                         |        |                      |          |
| Closure      |                         |        |                      |    ✓    |
| Resource     |                         |        |                      |          |

Notes:

- **Resource** is the junction point where identity and lifecycle meet: being a Resource confers
  both `subject()` (identity) and `close()` (lifecycle) on every substrate component that satisfies
  the Resource contract — specifically the types marked Resource ✓ in the matrix above. Resource
  itself is an abstract role — projections expose it via whichever mechanism is idiomatic (interface
  inheritance, trait, protocol conformance), and MAY expose additional resource types that satisfy
  the same contract.
- **Source** is the subscribable specialization of Resource: it adds the subscription affordance
  (`subscribe`) on top of the identity-and-lifecycle base, and confers it on Conduit. Like Resource,
  Source is an abstract role.
- **Bank** satisfies the Resource role — it has identity (via `subject()`) and lifecycle (via
  `close()`) — but is not a Source. It is not subscribable; it is a factory-with-ownership that
  materializes and closes conduits on demand.
- **Sink** carries `subject()` like a conduit but is **not** a Source and **not** a Resource: it has
  no subscription side and no `close()` operation (§10.5). It is the output-closed dual of a
  conduit — a pool of circuit-owned channel pipes whose captures forward to a fixed endpoint;
  closing the owning circuit releases its channels.
- **Resource (closeable)** in the table column refers to the closeable-lifecycle role — satisfying
  the Resource contract (§9.1) and exposing `close()` and `closeAwait()`. The spec does not mandate
  any specific projection mechanism: a projection MAY realize this role via a dedicated
  `Resource` interface, via the host language's standard closeable type (e.g., Java's
  `AutoCloseable`), or via any equivalent affordance. Rows marked Resource ✓ are the types that MUST
  satisfy the role.
- **Scope** is closeable but is *not* a Source and is not part of the universal Resource contract —
  it provides hierarchical resource management (§9.2), not subscribable emissions or circuit-backed
  cleanup barriers. Scope exposes `close()` in its own operation block (§16.3). Projections MAY bind
  Scope to the host language's standard closeable mechanism. (In the Java projection, Scope binds
  directly to `AutoCloseable` rather than implementing the `Resource`
  interface — see Appendix A.2.)
- **Cell** is a substrate component (it carries `subject()`) but is not a Source and not a Resource.
  Its lifetime is bound to its owning circuit; closing the circuit releases the cell along with
  other circuit-owned resources.
- **Ticker** is a circuit-owned periodic emitter (§11.4). It satisfies the Resource role — it has
  identity (`subject()`) and lifecycle (`close()`) — but is not a Source: it drives emissions into a
  target pipe rather than exposing subscribable channels.
- **Fiber**, **Flow**, **Receptor**, **Lookup**, and **Pool** are abstract protocols (standalone
  values or structural types), not substrate components; they have no `subject()` of their own.
- **Window**, **Capture**, **Pulse**, **Routing**, **Name**, **Id**, **Subject**, **Extent**,
  **State**, and **Slot** are value, selector, or identity records, not substrate components; they
  have no `subject()` of their own.
- Types with `(temporal)` in §6.4 — **Registrar**, **Window**, and **Closure** — are marked in the
  Temporal column.

A conformant implementation MUST expose `subject()` on every type marked Substrate ✓, MUST satisfy
the Source contract (§16.3) on every type marked Source ✓, MUST satisfy the Resource contract (§9.1)
on every type marked Resource ✓, and MUST enforce the temporal contract (§6.4) on every type marked
Temporal ✓. This matrix is the single point of truth for these capabilities; other sections
reference it rather than restating the inventory.

### 16.3 Required Operations

A conformant implementation MUST provide the following operations. The signatures below use abstract
notation: `Type.operation(param: ParamType): ReturnType`. Notation conventions:

- Optional parameters are suffixed with `?`. A return type suffixed with `?` denotes a result that
  may be an absent value (§1.2).
- Type parameters are written as `<T>`. Operations with no return value omit the return type.
- `(also X, Y)` after a type heading means the type satisfies the contracts of X and Y. Projections
  may express this as interface inheritance, trait implementation, protocol conformance, structural
  typing, or duck typing.
- `Sequence<T>` denotes an ordered, iterable collection. Projections map this to their idiomatic
  traversable type (`Iterable`, `Iterator`, `Stream`, `Vec`, slice, etc.).
- Container parameters (`List<T>`, `Pool<T>`) constrain only the capability of their elements;
  the notation does not repeat projection-specific container variance. Typed projections SHOULD
  accept idiomatic covariant containers of conforming elements — in Java,
  `List<Pipe<? super E>>` in the notation admits `List<? extends Pipe<? super E>>`, and
  `Pool<Pipe<? super E>>` admits `Pool<? extends Pipe<? super E>>`.
- `Ordering` denotes a three-valued comparison result (less than, equal to, greater than). See
  §6.2.4.
- `Routing` denotes a conduit dispatch selector. The required modes are per-pipe routing and
  hierarchical routing (§10.3). Implementations that do not support hierarchical dispatch MUST
  accept the hierarchical mode and behave as if per-pipe routing were selected.
- Scalar type names (`Boolean`, `Integer`, `Long`, `Double`, etc.) are specification vocabulary
  matching §8.3. They are not Java wrapper class names — projections use native types as specified
  in the §8.3 table.
- `Duration` denotes a non-negative span of elapsed time. Projections map this to their idiomatic
  duration type (`java.time.Duration` in Java, `time.Duration` in Go, `Duration` in Rust, etc.). The
  specification requires only positive, well-defined values; representable precision and overflow
  semantics are projection-dependent and MUST be documented.

#### Universal Operations

The operations below are inherited by every type that carries the corresponding capability in the
§16.2 matrix, and are therefore NOT repeated in each type's operation block. A conformant
implementation MUST expose them on every matching type.

```text
<any Substrate ✓ type>.subject(): Subject<S>
<any Resource ✓ type>.close()
<any Resource ✓ type>.closeAwait()
```

- **`subject()`** — returns the identity record of this substrate component (§4.3). The type
  parameter `S` is the type of the receiving substrate, so the returned subject is typed to its
  bearer (e.g., `Subject<Circuit>`, `Subject<Pipe<E>>`). Every type marked Substrate ✓ in §16.2 —
  Cortex, Circuit, Conduit, Source, Bank, Sink, Cell, Basin, Port, Pin, Ticker, Pipe, Current,
  Scope, Subscriber, Subscription — MUST expose `subject()`.
- **`close()`** — releases resources and terminates associated operations with the semantics of §9.1
  (idempotent, concurrency-safe, post-close drop). Every type marked Resource ✓ in §16.2 — Circuit,
  Conduit, Source, Bank, Ticker, Subscriber, Subscription — MUST expose `close()`.
- **`closeAwait()`** — blocking close; returns only after the close has fully taken effect (§9.1).
  For circuit-backed resources it MUST NOT be called from within the owning circuit's context.
  Every type marked Resource ✓ MUST expose `closeAwait()` with the semantics defined in §9.1.

Per-type operation blocks below list only operations that are *specific* to that type. Where a block
shows `close()` or `subject()` explicitly (for example in the Source and Circuit blocks), it is
included for clarity of the surrounding prose, not to exempt other types from the universal
contract. Types whose only operations are the universal ones have an operation block that notes this
explicitly rather than being omitted.

#### Factory Operations

**Cortex** — root factory:

```text
Cortex.circuit(name?: Name): Circuit
Cortex.name(value: String): Name
Cortex.scope(name?: Name): Scope
Cortex.current(): Current
Cortex.state(): State
Cortex.slot(name: Name, value: T): Slot<T>
Cortex.pool(fn: (Name) → T): Pool<T>
Cortex.fiber(): Fiber<E>
Cortex.flow(): Flow<E, E>
Cortex.flow(fiber: Fiber<E>): Flow<E, E>
```

The `slot` operation MUST support all types defined in §8.3. The `pool` operation constructs a root
pool (§10.1) from a factory function: the function is invoked at most once per name, the cached
outcome — value, returned-absence rejection, or thrown error — is replayed for subsequent lookups,
and the returned pool owns nothing (it is not a resource and cascades no lifecycle to materialized
values). The `fiber` operation returns an empty
identity fiber over a single emission type `E`; per-emission operators are chained onto it to build
a recipe (§6.2), which is then attached via `Fiber.pipe(pipe)` or attached to a Flow via
`Flow.fiber(fiber)` / `Flow.fiber(factory)`. The `flow()` operation returns an empty identity flow
whose input and output types coincide; operators are chained onto it to build a composition chain
(§6.2), which is then applied to a pipe via pipe composition (§6.2.6). The `flow(fiber)` operation
returns an identity flow with `fiber` already attached — a convenience for transitioning from a
Fiber recipe to a Flow that can be further composed with type-changing operators. Note:
`Cortex.state()`
creates an empty State, while `State.state(name, value)` adds a slot and returns a new State. The
reuse of the name `state` across these types is deliberate — the operation semantics are unambiguous
because the receiver types differ.

When the optional `name` parameter is omitted from `circuit` or `scope`, the implementation MUST
assign a valid non-empty default name. That default name need not be unique; uniqueness is provided
by the subject identifier (§4.2), not by anonymous-name generation.

Projections MAY provide additional convenience operations — e.g., extra `name()` overloads for
classes, enums, or iterables, `state(Slot)` for direct slot insertion, or `fiber(type)` /
`flow(type)` overloads that use a type classifier to drive inference — as non-normative extensions.
The operations listed above define the minimum portable surface.

#### Execution Operations

**Circuit** — sequential execution engine (also Resource):

```text
Circuit.conduit(name?: Name, type?: Type): Conduit<E>
Circuit.bank(type: Type): Bank<Conduit<E>>
Circuit.bank(type: Type, routing: Routing): Bank<Conduit<E>>
Circuit.current(): Current
Circuit.pipe(): Pipe<E>
Circuit.pipe(target: Pipe<? super E>): Pipe<E>
Circuit.pipe(targets: List<Pipe<? super E>>): Pipe<E>
Circuit.pipe(receptor: Receptor<? super E>): Pipe<E>
Circuit.pipe(name: Name, receptor: Receptor<? super E>): Pipe<E>
Circuit.pipe(name: Name, targets: List<Pipe<? super E>>): Pipe<E>
Circuit.cell(initial: E): Cell<E>
Circuit.cell(name: Name, initial: E): Cell<E>
Circuit.basin(capacity: Integer): Basin<E>
Circuit.port(initial: E): Port<E>
Circuit.port(name: Name, initial: E): Port<E>
Circuit.pin(initial: E): Pin<E>
Circuit.pin(name: Name, initial: E): Pin<E>
Circuit.ticker(interval: Duration, target: Pipe<? super Long>): Ticker
Circuit.ticker(name: Name, interval: Duration, target: Pipe<? super Long>): Ticker
Circuit.subscriber(name: Name, callback: (Subject<Pipe<E>>, Registrar<E>)): Subscriber<E>
Circuit.subscriber(name: Name, pool: Pool<Pipe<? super E>>): Subscriber<E>
Circuit.sink(endpoint: Pipe<Capture<E>>): Sink<E>
Circuit.sink(name: Name, endpoint: Pipe<Capture<E>>): Sink<E>
Circuit.await()
Circuit.pulse(): Pulse?
Circuit.close()
Circuit.closeAwait()
```

The `conduit` operation creates a conduit for the specified emission type classifier (§4.5), with an
optional name. The type classifier MAY also be implicit — inferred by the projection from the
call-site context — in the same manner as the `flow` factory (§16.3 Factory Operations). The `bank`
operations create a Bank of conduits (§10.4): the single-argument form uses default per-pipe
routing; the two-argument form selects the routing behavior. Each call returns a distinct Bank
instance — two `bank` calls for the same type do not share identity. `bank` is an open-required
operation: if the circuit has accepted close, it MUST signal a closed-resource error synchronously.
The `pipe` operations create circuit-dispatched pipes; implementations may choose the internal
mechanism or fast path as long as the public ordering and circuit-context confinement guarantees
hold (§5.3, §6.1). Flow processing is applied via pipe composition (§6.2.6) rather than
circuit-level overloads. The no-argument form returns a circuit-owned pipe whose dispatched
emissions are discarded on the circuit context — work is scheduled but the resulting job is a
no-op — useful as a sink or placeholder where circuit participation is required but no downstream
processing is needed. The `pipe(targets)` overload returns a circuit-owned pipe that fans each
emission out to a fixed list of target pipes. The target list is snapshotted at creation; later
mutation of the caller's list MUST NOT affect the returned pipe. A null list or any null element
MUST be rejected synchronously, and any target that is not a provider-compatible Pipe MUST signal a
provider mismatch (§15.1) synchronously. Each emission enters this circuit's queue once and then
dispatches to the targets in list order; a target appearing more than once receives the emission
once per occurrence. Same-circuit targets observe list order on this circuit context, while
cross-circuit targets are enqueued to their own circuits in list order but execute on those
circuits' workers, so completion order across circuits is not guaranteed (§5.3). An empty list is
equivalent to `pipe()` (a no-op sink), and a single-element list is equivalent to `pipe(target)`.
The `pipe(name, receptor)` overload is the named-pipe form of
`pipe(receptor)`: the supplied name is bound to the returned pipe's subject (it does not name the
receptor), and emissions reach the receptor with the same sequencing and failure-isolation semantics
as the unnamed form. The `pipe(name, targets)` overload is the named-pipe form of `pipe(targets)`:
the supplied name is bound to the returned pipe's subject. Because the name MUST be honored, it
always mints a new circuit-owned pipe and does not collapse the empty-list or single-target cases of
`pipe(targets)` (an empty list yields a named no-op pipe; a single-target list yields a named
forwarder). Use a single-element list when a named single-target forwarder is required. List-order
dispatch, per-occurrence duplicates, creation-time snapshotting, provider mismatch (§15.1), and
cross-circuit ordering are otherwise identical to `pipe(targets)`. The `cell` operations create a
circuit-owned, initialized single-slot state holder (§11.2); the supplied initial value seeds the
cell and is observed by readers until the first accepted update is processed. The two-argument form
additionally assigns an explicit subject name; the one-argument form uses the owning circuit's name.
Cells have no absent state — there is no no-argument form. The `basin` operation creates a
circuit-owned bounded value buffer (§11.1) with a fixed positive capacity; the returned basin is
fed through its stable pipe and drained asynchronously through `Basin.drain`. Basins require no
initial value, may be empty, and have no lifecycle of their own beyond the owning circuit. The
`port`
operations create a circuit-owned, initialized state slot (§11.5) that grants queued mutation
authority — `replace`, `update`, `update(arg, fn)`, and `emit(pipe)` — without exposing a read
accessor; the supplied initial value seeds the port and is the first value transformations observe.
The two-argument form additionally assigns an explicit subject name; the one-argument form uses the
owning circuit's name. Ports have no absent state — there is no no-argument form. The `pin`
operations create a circuit-owned, initialized state handle (§11.6) that grants immediate
inspect/mutate access confined to the owning circuit's worker thread — `get` and `set` execute
synchronously, but only from inside the owning context; invocation from any other context raises an
illegal-context-use error (§15.1). The two-argument form additionally assigns an explicit subject
name; the one-argument form uses the owning circuit's name. Pins have no absent state — there is no
no-argument form. The
`ticker` operations create circuit-owned periodic emitters (§11.4): both forms take a strictly
positive interval and a target pipe; the three-argument form additionally assigns an explicit name,
while the two-argument form uses a default name. The
`subscriber` operation creates a callback handle for dynamic pipe discovery; the callback receives
the subject of a named pipe and a registrar for attaching downstream pipes or receptors. The
pool-backed subscriber form is equivalent to a callback that registers `pool.get(subject)` for each
discovered subject (§7.2). The `sink` operations create a sink (§10.5) — the output-closed dual of a
conduit — that funnels each emission into the supplied `Pipe<Capture<E>>` endpoint as a capture
(§11.1); the emission type is taken from the endpoint, and the two-argument form additionally
assigns an explicit subject name. `current` returns the circuit context identity (§5.7, §11.3).
`pulse`
submits a diagnostic probe and returns absence only once the circuit context has terminated (§5.7).
Projections MAY provide additional overloads — e.g.,
`conduit(name, type, routing)` for hierarchical dispatch (§10.3), or an explicit type-witness
overload `conduit(type)` analogous to `flow(type)` — as non-normative conveniences.

`conduit`, all `pipe` forms, both `cell` forms, the `basin` form, both `port` forms, both `pin`
forms, both `ticker` forms, both `subscriber` forms, and both `sink` forms are open-required
operations (§9.1). If the circuit has accepted close at the time of the call, they MUST signal a
closed-resource error synchronously and MUST NOT return an inert conduit, pipe, cell, basin, port,
pin, ticker, subscriber, or sink. This includes the conditional same-circuit form `pipe(target)`:
after close it MUST signal the closed-resource error rather than return the target as-is.

#### Emission Operations

**Pipe** — emission carrier:

```text
Pipe.emit(value: E)
```

The `emit` operation delivers a value through this pipe. Pipe itself exposes no composition surface;
Fibers and Flows are attached to a pipe, or to a Cell's update pipe, via composition methods on the
composition values themselves (`Fiber.pipe(pipe)`, `Flow.pipe(pipe)`, `Fiber.pipe(cell)`,
`Flow.pipe(cell)`) — see §6.2.6.

**Receptor** — emission callback:

```text
Receptor.receive(value: E)
```

**Cell\<E\>** — circuit-owned initialized single-slot state holder (§11.2):

```text
Cell.get(): E
Cell.pipe(): Pipe<E>
```

`get` returns the cell's current value and is total — it MUST NOT return an absent value (§1.2). A
cell is always created with a non-null seed via `Circuit.cell(initial)` or
`Circuit.cell(name, initial)`; `get` returns the seed
until the first accepted update is processed, then the most recently published value thereafter.
`pipe` returns the update pipe for this cell; repeated calls return the canonically identical pipe
(§1.2). Emissions submitted through this pipe are queued on the cell's owning circuit and processed
in circuit-context confinement (§5.1). Cells expose no other write capability — they are not
Receptors and not publicly emittable Pipes; the update pipe is the only ingress.

**Port\<E\>** — circuit-owned initialized state slot with queued mutation authority (§11.5):

```text
Port.replace(value: E)
Port.update(fn: E -> E)
Port.update(arg: A, fn: (E, A) -> E)
Port.emit(pipe: Pipe<? super E>)
```

A port exposes no read accessor — `get` is intentionally absent. All operations are queued (§11.5);
a call from any context returns before its state effect has been applied. `replace` queues a
whole-value replacement. `update(fn)` queues a transformation invoked with the current present value
on the owning circuit context; a non-null return is stored, a null/absent return or thrown exception
is isolated as an external callback failure (§15.4) and leaves the previous value unchanged. A
null/absent return SHOULD be surfaced as a structured fault carrying the port's subject and
`operation = "update"` (§15.4 framework-detected contract violations). `update(arg, fn)` is the same
with an explicit argument captured at call time and supplied to the function on the worker thread.
`emit(pipe)` queues an emission of the current stored value to the supplied target pipe; the target
pipe MUST be a provider-compatible pipe (§15.1 provider mismatch). All values passed to and returned
from these operations MUST be present (§15.2). Ports expose no other write capability — they are not
Receptors and not publicly emittable Pipes; the four queued operations are the only ingress.

**Pin\<E\>** — circuit-owned initialized state handle with immediate inspect/mutate authority,
confined to the owning circuit context (§11.6):

```text
Pin.get(): E
Pin.set(value: E)
```

Both operations execute immediately on the owning circuit's worker thread; no queueing. They MUST be
invoked from the owning circuit's context and MUST raise an illegal-context-use error (§15.1)
when invoked from any other context. `get` returns the current stored value and is total — it MUST
NOT return an absent value (§1.2). `set` replaces the stored value immediately; the next statement
in the same callback observes the new value. All values passed to these operations MUST be present
(§15.2). Pins expose no other capability — they are not Receptors and not publicly emittable Pipes;
the two operations above are the only access surface. A Pin is itself a managed substrate handle
(§6.1.1) and MAY be emitted as a value through any pipe; receivers outside the owning circuit may
store or forward the handle but cannot dereference it.

**Ticker** — circuit-owned periodic emitter (Resource):

```text
// no type-specific operations
```

Ticker has no operations beyond the universal `subject()`, `close()`, and `closeAwait()` contracts
(§16.3 Universal Operations). Its behavioral contract is defined at construction time via
`Circuit.ticker` (§16.3 Execution Operations) and governed by the scheduling semantics of §11.4.
Closing a ticker stops further scheduled emissions; ticks already admitted to the circuit before the
close MAY still be delivered.

#### Pipeline Operations

**Fiber\<E\>** — same-type per-emission recipe. A fiber carries a single emission type `E`; every
operator preserves that type and returns `Fiber<E>`.

```text
Fiber.guard(predicate: (E) → Boolean): Fiber<E>
Fiber.guard(initial: E, predicate: (E, E) → Boolean): Fiber<E>
Fiber.diff(): Fiber<E>
Fiber.diff(initial: E): Fiber<E>
Fiber.heartbeat(maxSilence: Duration): Fiber<E>
Fiber.change(key: (E) → K?): Fiber<E>
Fiber.limit(count: Long): Fiber<E>
Fiber.skip(count: Long): Fiber<E>
Fiber.takeWhile(predicate: (E) → Boolean): Fiber<E>
Fiber.dropWhile(predicate: (E) → Boolean): Fiber<E>
Fiber.reduce(initial: E?, operator: (E?, E) → E?): Fiber<E>
Fiber.relate(initial: E?, operator: (E?, E) → E?): Fiber<E>
Fiber.edge(initial: E, transition: (E, E) → Boolean): Fiber<E>
Fiber.pulse(predicate: (E) → Boolean): Fiber<E>
Fiber.steady(count: Integer): Fiber<E>
Fiber.steady(count: Integer, same: (E, E) → Boolean): Fiber<E>
Fiber.streak(required: Integer, matches: (E) → Boolean): Fiber<E>
Fiber.hysteresis(enter: (E) → Boolean, exit: (E) → Boolean): Fiber<E>
Fiber.inhibit(refractory: Integer): Fiber<E>
Fiber.delay(depth: Integer, initial: E): Fiber<E>
Fiber.integrate(initial: E?, accumulator: (E?, E) → E?, fire: (E?) → Boolean): Fiber<E>
Fiber.rolling(size: Integer, combiner: (E?, E) → E?, identity: E): Fiber<E>
Fiber.tumble(size: Integer, combiner: (E?, E) → E?, identity: E): Fiber<E>
Fiber.every(interval: Integer): Fiber<E>
Fiber.every(duration: Duration): Fiber<E>
Fiber.chance(probability: Double): Fiber<E>
Fiber.distinct(): Fiber<E>
Fiber.distinct(capacity: Integer): Fiber<E>
Fiber.replace(operator: (E) → E?): Fiber<E>
Fiber.peek(receptor: Receptor<? super E>): Fiber<E>
Fiber.tee(pipe: Pipe<E>): Fiber<E>
Fiber.route(predicate: (E) → Boolean, pipe: Pipe<E>): Fiber<E>
Fiber.when(predicate: (E) → Boolean, fiber: Fiber<E>): Fiber<E>
Fiber.above(comparator: (E, E) → Ordering, lower: E): Fiber<E>
Fiber.below(comparator: (E, E) → Ordering, upper: E): Fiber<E>
Fiber.min(comparator: (E, E) → Ordering, minimum: E): Fiber<E>
Fiber.max(comparator: (E, E) → Ordering, maximum: E): Fiber<E>
Fiber.range(comparator: (E, E) → Ordering, lower: E, upper: E): Fiber<E>
Fiber.deadband(comparator: (E, E) → Ordering, lower: E, upper: E): Fiber<E>
Fiber.clamp(comparator: (E, E) → Ordering, lower: E, upper: E): Fiber<E>
Fiber.high(comparator: (E, E) → Ordering): Fiber<E>
Fiber.low(comparator: (E, E) → Ordering): Fiber<E>
Fiber.fiber(next: Fiber<E>): Fiber<E>
Fiber.pipe(pipe: Pipe<? super E>): Pipe<E>
Fiber.pipe(cell: Cell<? super E>): Pipe<E>
```

An identity fiber is obtained from `Cortex.fiber()` and is the starting point for building a
per-emission recipe. The completed fiber is handed to `Fiber.pipe(pipe)` for direct attachment
(§6.2.6), to `Fiber.pipe(cell)` for direct attachment to a cell's update pipe, or to
`Flow.fiber(fiber)` / `Cortex.flow(fiber)` to be wrapped in a Flow.

**Flow\<I, O\>** — left-to-right composition surface. A flow carries two type parameters: the input
type `I` (upstream-facing) and the output type `O` (downstream-facing). A same-type flow (`I == O`)
is valid. Flow holds only operators that cross type boundaries or append an output-side stage:

```text
Flow.map(fn: (O) → P?): Flow<I, P>
Flow.scan(initial: () → S?, step: (S?, O) → S?): Flow<I, S>
Flow.scan(initial: () → S?, step: (S?, O) → S?, emit: (S?) → P?): Flow<I, P>
Flow.scan(initial: () → S?, step: (S?, O) → S?, emit: (S?, O) → P?): Flow<I, P>
Flow.relate(initial: O?, op: (O?, O) → P?): Flow<I, P>
Flow.window(count: Integer): Flow<I, Window<O>>
Flow.window(duration: Duration, capacity: Integer): Flow<I, Window<O>>
Flow.run(): Flow<I, Run<O>>
Flow.change(): Flow<I, Change<O>>
Flow.fiber(fiber: Fiber<O>): Flow<I, O>
Flow.fiber(factory: (Subject) → Fiber<O>): Flow<I, O>
Flow.flow(next: Flow<? super O, ? extends P>): Flow<I, P>
Flow.flow(factory: (Subject) → Flow<? super O, ? extends P>): Flow<I, P>
Flow.pipe(pipe: Pipe<? super O>): Pipe<I>
Flow.pipe(cell: Cell<? super O>): Pipe<I>
```

`map` appends a type-changing transformation at the output side (an `O → P` function, returning
`Flow<I, P>`); a return of absence drops the emission. `scan` appends a type-changing stateful fold
whose state type `S` is independent of input `O` (see §6.2.3); state-as-output, state-only
projection, and input-aware projection forms are defined. `relate` appends a type-changing
projection of the relation between consecutive output values: `prev` is seeded with `initial` and
advances to the current value on every emission (including emissions the projection drops), and
`op(prev, curr)` produces the downstream value, returning absence to drop the emission. The
`(prev, curr)` pair is visible to `op` but does not itself escape — only the projected `P`
is emitted — which is the form by which a transition between two values becomes a single derived
value. It is the type-changing sibling of `Fiber.relate` (identical state semantics, generalized
from `(O, O) → O` to `(O, O) → P`); a gating use such as transition naming returns absence on the
pairs that should not fire. `window(count)` appends a bounded rolling temporal Window at the output
side and returns `Flow<I, Window<O>>`; `count` must be positive.
`window(duration, capacity)`
appends a time-bounded rolling Window at the output side and returns `Flow<I, Window<O>>`;
`duration` must be positive and `capacity` must be greater than zero. There is intentionally no
duration-only overload — time-based retention without an explicit capacity is unbounded in memory
under high-rate feeds. `run()` appends a per-admission run-length stage and returns
`Flow<I, Run<O>>`: each admission is turned into a `Run` envelope carrying the emission and the
length of its consecutive run — 1 on the first admission and after every change, incrementing while
the value repeats. `change()` appends a run-boundary stage and returns `Flow<I, Change<O>>`: it
emits only at a boundary — an admission value-unequal to its predecessor — a `Change` envelope
carrying the value the closed run held (`from`), the value that opened the next run (`to`), and the
terminal
`length` the closed run reached; the first admission opens the first run and emits nothing, and the
open or final run is never reported, its length observable only through `run`. Both decide
consecutive repetition by value equality (callers wanting repetition over a derived key `map` to the
key first), are stateful per materialization, and produce fresh immutable envelopes — like
`Capture`, safe to retain, compare, and forward. `fiber(fiber)` appends a same-type Fiber recipe at
the output side, returning
`Flow<I, O>`; the Fiber's operators run last on each emission, after any preceding `map`, `scan`,
`window`, or upstream stages. `fiber(factory)` appends a per-attachment Fiber recipe created from
the materialized pipe subject (§6.2). `flow(next)` and
`Fiber.fiber` compose two recipes of the same kind end-to-end; the left-hand recipe runs first, its
surviving emissions are fed into the right-hand recipe, and framework-owned state slots are
materialized independently for each attachment. User-provided mutable Scan state is independent only
when the Supplier returns a fresh object per materialization; Window's rolling buffer is always
framework-owned per materialization. `flow(factory)` is the canonical subject-aware composition
primitive (§6.2): the factory is invoked once per attachment with the materialized pipe's subject
and returns the Flow segment to compose. Projections SHOULD NOT add per-operator subject-aware
overloads when this composition primitive suffices.

An identity flow (`I == O`) is obtained from `Cortex.flow()` and is the starting point for building
a Flow. The completed flow is handed to `flow.pipe(target)` or `flow.pipe(cell)` for composition
(§6.2.6).

**Window\<E\>** — callback-scoped temporal rolling window. A Window is produced by
`Flow.window(count)`, carries the current encounter order, and is valid only during the callback
that observes it.

```text
Window.first(): E
Window.last(): E
Window.size(): Integer
Window.isEmpty(): Boolean
Window.prefix(count: Integer): Window<E>
Window.suffix(count: Integer): Window<E>
Window.skip(count: Integer): Window<E>
Window.trim(count: Integer): Window<E>
Window.slice(offset: Integer, count: Integer): Window<E>
Window.reverse(): Window<E>
Window.forEach(action: (E) → Void): Void
Window.all(predicate: (E) → Boolean): Boolean
Window.any(predicate: (E) → Boolean): Boolean
Window.none(predicate: (E) → Boolean): Boolean
Window.count(predicate: (E) → Boolean): Integer
Window.fold(seed: R, op: (R?, E) → R?): R?
Window.reduce(identity: E, op: (E?, E) → E?): E?
```

`first()` and `last()` return the first and last visible values in the current encounter order and
MUST signal a no-such-element error if the Window is empty. `size()` runs in `O(1)` and reports the
number of visible values; `isEmpty()` is equivalent to `size() == 0`. Restriction operations that
accept a `count` require `count >= 0`; `slice` also requires `offset >= 0`. A zero count produces an
empty Window for `prefix(count)`, `suffix(count)`, and `slice(offset, count)`, and is a no-op for
`skip(count)` and `trim(count)`. If a requested restriction exceeds the visible size, the result is
clamped to the available values; if it skips or slices beyond the visible size, the result is empty.
`reverse()` returns a Window whose encounter order is reversed. `forEach(action)` eagerly invokes
the action for each visible value in encounter order. `all`, `any`, `none` short-circuit on the
first decisive match; an empty view returns `true`, `false`, `true` respectively (vacuous-truth
semantics for `all` and `none`). `count(predicate)` returns the number of visible values that
satisfy the predicate, in `[0, size()]`. `fold(seed, op)` is a left-to-right type-changing fold that
returns `seed` for an empty view; `reduce(identity, op)` is the same-type analogue that returns
`identity` for an empty view. Window fold/reduce operators MAY return absence; after that,
subsequent invocations of the supplied operator receive absence as the accumulator, and the final
result MAY be absence. Window instances do not define structural equality over visible values;
callers that need to retain or forward contents MUST copy values during the observing callback.
Window MUST NOT expose lazy traversal handles such as iterators, spliterators, or streams.

**Lifetime enforcement**: Window operators MUST signal an illegal temporal use error (§15.1) when
invoked on a view whose originating callback has returned (per the Window exception in §6.4.1). This
applies to every operator on the root Window emitted by `Flow.window(count)` and to all derived
views obtained from it (`prefix`, `suffix`, `slice`, `skip`, `trim`, `reverse`).

**Run\<E\>** — immutable per-admission run-length envelope. A Run is produced by `Flow.run()`.
Unlike Window it is **not** callback-scoped (it carries no temporal contract, §16.2): it MAY be
retained, compared, and forwarded.

```text
Run.emission(): E
Run.length(): Long
```

`emission()` returns the value whose consecutive run the envelope describes and MUST NOT be absent.
`length()` returns the consecutive-run length — the number of consecutive value-equal admissions
ending at this one — and MUST be `>= 1`. The length uses the `Long` scalar vocabulary (§8.3);
projections use their native 64-bit signed representation and SHOULD document behavior on overflow.

**Change\<E\>** — immutable run-boundary envelope. A Change is produced by `Flow.change()`. Like Run
it is **not** callback-scoped: it MAY be retained, compared, and forwarded.

```text
Change.from(): E
Change.to(): E
Change.length(): Long
```

`from()` returns the value held by the run that just closed; `to()` returns the value that opened
the next run; neither MUST be absent. `length()` returns the terminal length the closed run
reached — the length `Run.length()` would have reported for `from()` on the admission immediately
before the boundary — and MUST be `>= 1` (same representation contract as `Run.length()`).

#### Routing Operations

**Lookup\<T\>** — abstract base for name-indexed retrieval:

```text
Lookup.get(name: Name): T
```

The `get` operation returns the instance for the given name. Creation and caching semantics depend
on the concrete implementation: Pool creates on first access and caches; Bank creates the resource
on first access, caches it, and closes it when the bank closes. Implementations MUST be thread-safe
for concurrent retrieval. Projections MAY provide additional `get` overloads — e.g., `get(Subject)`
or `get(Substrate)` — as non-normative conveniences.

**Pool\<T\>** — composable name-based view (also Lookup\<T\>):

```text
Pool.pool(fn: (T) → U): Pool<U>
```

`Pool` inherits `get(name)` from Lookup (§10.1). The `pool(fn)` operation creates a derived pool
that caches transformed results per name (§10.1). A pool is obtained as a facet of a Conduit or
Sink, as a derived view via `pool(fn)`, or as a root pool via `Cortex.pool(fn)` (§10.1 Root pools).

**Bank\<R\>** — closeable name-indexed resource holder (also Lookup\<R\>, Resource):

```text
Bank.get(name: Name): R      [open-required]
Bank.close()
Bank.closeAwait()
```

`Bank` inherits `get(name)` from Lookup. `get` materializes the resource on first access and returns
the cached instance on subsequent lookups; both the identity guarantee and thread-safety requirement
of Lookup apply (§10.4). `get` is open-required (§9.1): calling it after the bank has accepted close
MUST signal a closed-resource error synchronously. `close` and `closeAwait` are inherited from the
universal Resource contract (§9.1).

**Conduit\<E\>** — pipe factory and source (also Pool\<Pipe\<E\>\>, Source\<E\>, Resource):

```text
Conduit.pool(flow: Flow<T, E>): Pool<Pipe<T>>
Conduit.pool(fiber: Fiber<E>): Pool<Pipe<E>>
Conduit.close()
Conduit.closeAwait()
```

`Conduit` extends `Pool<Pipe<E>>` and inherits `get(name)` and `pool(fn)`. As a Source it inherits
`subscribe` and `subject`, and as a Resource (via Source) it exposes `close()` and `closeAwait()`
with the semantics of §9.1. The `pool(Flow)` and `pool(Fiber)` overloads are pipe-aware forms of
`pool(fn)` available only on `Conduit` (since `Pool<T>` is generic in `T`, it cannot express the
`Pipe<E> → Pipe<T>` relationship). Both are semantically equivalent to `pool(flow::pipe)` /
`pool(fiber::pipe)` respectively — they prepend the flow or fiber to each channel's pipe, so
emissions of the upstream type flow through the operators and land in the conduit's `Pipe<E>`
after transformation (type-changing for Flow) or filtering/stateful processing (type-preserving for
Fiber).

#### Subscription Operations

**Source\<E, S\>** — subscribable event origin (also Resource):

```text
Source.subscribe(subscriber: Subscriber<E>): Subscription
Source.subscribe(subscriber: Subscriber<E>, onClose: (Subscription) → Unit): Subscription
Source.subject(): Subject<S>
```

The first `subscribe` overload registers a subscriber. The second additionally attaches an `onClose`
callback that fires exactly once when the returned subscription is terminated — whether by explicit
close, subscriber close, or source close. The callback receives the subscription being closed and
executes within the circuit context. If the owning circuit has already terminated and cannot accept
the cleanup work, the callback is not required to run.

Source-to-source mirroring — following a source's channel structure into a conduit of transformed
emissions — is not a distinct type; it is the composition of existing operations: a pool-backed
subscriber (`Circuit.subscriber(name, pool)`) bridging a source into a conduit's derived pool
(`Conduit.pool(flow)`, `Conduit.pool(fiber)`, or `Pool.pool(fn)`). The derived pool materializes the
recipe once per channel name, the bridging subscription stops the feed when closed, and the target
conduit is an ordinary source with its own lifecycle.

Both `subscribe` overloads are open-required operations (§9.1). If the source has accepted close at
the time of the call, they MUST signal a closed-resource error synchronously and MUST NOT admit
registration work.

**Registrar** — pipe attachment handle (temporal, valid only during subscriber callback):

```text
Registrar.register(pipe: Pipe<? super E>)
Registrar.register(receptor: Receptor<? super E>)
```

`register(pipe)` attaches a downstream pipe. Multiple pipe registrations receive each emission in
registration order. `register(receptor)` attaches a terminal receptor with the same channel
visibility, temporal validity, circuit-context confinement, and failure-isolation guarantees.
Implementations MAY store receptor registrations directly, wrap them in pipes, or use another
internal representation.

**Subscription** — cancellable subscription handle (also Resource):

```text
Subscription.close()
Subscription.closeAwait()
```

**Subscriber\<E\>** — pipe discovery callback (also Resource):

```text
// no type-specific operations
```

Subscriber has no operations beyond the universal `subject()`, `close()`, and `closeAwait()`
contracts defined above. Its behavioral contract is the callback passed at construction time
(`Circuit.subscriber(name, callback)`): the callback receives a `Subject<Pipe<E>>` and a
`Registrar<E>` when a named pipe is discovered on a subscribed source (§7.2, §7.3). The
Subscriber's subject identifies the handle itself; closing it cancels all subscriptions derived
from it and stops further callback invocations.

#### Derived Structure Operations

**Basin** — bounded value buffer (a Substrate, not a Resource):

```text
Basin.pipe(): Pipe<E>
Basin.drain(target: Pipe<? super E>)
```

A basin is created from a circuit (`Circuit.basin(capacity)`, §11.1). `pipe()` returns the emit-only
feed; emitting into it appends to the bounded ring buffer. `drain(target)` forwards the retained
values to `target` on the circuit worker thread and then clears the buffer, enqueuing the work and
returning immediately (§11.1). A basin holds plain values and does not mint captures.

**Capture\<E\>** — immutable emission provenance envelope:

```text
Capture.emission(): E
Capture.subject(): Subject<Pipe<E>>
Capture.current(): Subject<Current>
Capture.state(): State
```

A capture bundles an emitted value with its provenance (§11.1): the emitting channel pipe's subject
(typed `Subject<Pipe<E>>` — the pipe role, not the owning conduit or source), the emitting-context
subject (`current` — the external caller for an ingress emission, or the circuit itself for a
transit cascade), and a
`State` of optional provider-published measures (default empty). The context is carried as a
subject, not a live `Current`, keeping the capture an out-of-execution-scope reference record.

#### Context Operations

**Current** — execution context identity token:

```text
// no type-specific operations
```

Current has no operations beyond the universal `subject()` contract. It is obtained via
`Cortex.current()` for the calling execution context and via `Circuit.current()` for a circuit's
processing context. Current instances are interned identity tokens and MAY be retained or shared for
identity comparison (§11.3). The `subject()` of a Current returns the identity record of the
execution context that produced it, allowing callers to correlate work with its owning context
without exposing context-specific machinery.

**Pulse** — circuit diagnostic timing snapshot:

```text
Pulse.start(): Long
Pulse.enqueued(): Long
Pulse.dequeued(): Long
Pulse.stop(): Long
```

A Pulse is produced by `Circuit.pulse()` when a diagnostic probe traverses the circuit (§5.7). The
four values are monotonic timestamps in the projection's documented unit. They are intended for
difference arithmetic only; absolute values are not meaningful.

#### Identity Operations

**Subject** — component identity record:

```text
Subject.id(): Id
Subject.name(): Name
Subject.state(): State
Subject.type(): Type
```

**Name** — hierarchical interned identifier (also Extent):

```text
Name.name(segment: String): Name
Name.path(): String
```

**Extent** — hierarchical containment:

```text
Extent.part(): String
Extent.enclosure(): Extent?
Extent.extremity(): Extent
Extent.depth(): Integer
Extent.path(): String
```

#### Data Operations

**State** — immutable slot collection:

```text
State.state(name: Name, value: T): State
State.iterate(): Sequence<Slot>
State.value(slot: Slot<T>): T
```

The `state` operation MUST support all types defined in §8.3. Each call returns a new State
instance; the original MUST remain unchanged. Writing a slot with a (name, type) pair that already
exists in the state MUST upsert that logical slot — the returned state contains the newly written
slot in most-recently-written position, and any prior slot with the same (name, type) is removed.
Writing a slot whose (name, type, value) already matches the existing entry MUST return the same
state instance. Because of upsert semantics, the number of slots in a state is bounded by the number
of unique (name, type) pairs ever written, not by the number of writes. The `iterate` operation
returns slots in most-recently-written-first order, as defined in §8.1. The operation name
`iterate` is abstract — projections use their idiomatic traversal mechanism (e.g., `stream()` in
Java, `iter()` in Rust,
`__iter__` in Python). The `value` operation returns the value of the slot matching the given slot's
(name, type) pair, or the template slot's own value when no match exists.

**Slot** — named typed value:

```text
Slot.name(): Name
Slot.type(): Type
Slot.value(): T
```

#### Lifecycle Operations

**Resource** — closeable component:

```text
Resource.close()
Resource.closeAwait()
```

`close` is the asynchronous (non-blocking) close operation defined in §9.1. `closeAwait` returns
only after the close has fully taken effect, as defined in §9.1. Both are idempotent and
concurrency-safe.

**Scope** — hierarchical resource manager (also Extent). A scope's extent semantics: `part()`
returns the scope's name segment, `enclosure()` returns the parent scope's extent (or absent for a
root scope), and `depth()` / `path()` / `extremity()` traverse the scope nesting hierarchy:

```text
Scope.register(resource: R): R                  [R: Resource]
Scope.closure(resource: R): Closure<R>           [R: Resource]
Scope.scope(name?: Name): Scope
Scope.close()
```

The `closure` operation creates a single-use, block-scoped resource handle. Closure is temporal — it
is valid only during the scope that created it.

**Closure\<R\>** — block-scoped resource handle (temporal, single-use-scoped):

```text
Closure.consume(block: (R))              [R: Resource]
```

The `consume` operation executes the block with the managed resource of type R, then closes the
resource when the block exits (whether normally or via error). Implementations MUST invoke the block
exactly once while the resource is open, and MUST close the resource afterward. If the block signals
an error, the resource is still closed. If the owning scope is already closed, `consume` MUST be a
no-op. Consume is single-use — calling it more than once is an illegal temporal use (§6.4, §15.1).

### 16.4 Conformance Testing

A conformance test suite (Technology Compatibility Kit) validates these properties through
behavioral assertions. The TCK defines tests in terms of the required types and operations (§16.2,
§16.3) and the behavioral requirements (§16.1). Individual language implementations provide concrete
test harnesses that bind the abstract types to their language-specific projections.

---

## Appendix A. Projection Guidance (Non-Normative)

This appendix clarifies the boundary between specification requirements and language-specific
binding decisions. A **projection** is a conformant implementation in a specific language or
runtime. The specification constrains behavior; projections choose representation.

### A.1 Binding Obligations vs. Implementation Freedoms

| Topic                     | What the spec requires (normative)                                              | What projections may choose freely                                                          |
|---------------------------|---------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| **Type representation**   | Distinct, equality-comparable classifiers for each required type (§4.5)         | Java `Class`, Rust enum, Go string tag, integer constants, etc.                             |
| **Absent value**          | Drop semantics for mapping functions; absence violation detection (§1.2, §15.2) | `null`, `Option`, `Maybe`, nil, or similar sum types that distinguish "no value" from error |
| **Identity**              | Canonical identity within implementation boundary (§1.2)                        | Pointer equality, interned handles, content-addressed lookup                                |
| **Error signaling**       | Distinguishable error categories (§15.1)                                        | Exceptions, error returns, Result types, rejected futures                                   |
| **Execution context**     | Sequential, exclusive circuit context (§5.1)                                    | Thread, goroutine, coroutine, actor, event-loop turn                                        |
| **Concurrency mechanism** | Happens-before guarantees (§5.4)                                                | Memory barriers, channels, locks, runtime-specific ordering                                 |
| **Resource cleanup**      | Idempotent close, scope ordering (§9)                                           | `AutoCloseable`, `Drop`, `defer`, `with` statement, destructor                              |
| **Provider loading**      | A cortex instance exists as entry point (§3)                                    | `ServiceLoader`, dependency injection, explicit construction, registry                      |
| **Collection types**      | Ordered drain/state traversal with snapshot semantics (§11.1)                   | `Iterable`, `Stream`, `Iterator`, `Vec`, slice, generator                                   |
| **Window traversal**      | Callback-scoped eager traversal without lazy handles (§6.2.3)                   | Visitor/callback methods such as `forEach`; no iterators, spliterators, or streams          |
| **Numeric types**         | Specified precision and range (§8.3)                                            | Native primitives, wrapper types, newtype wrappers                                          |

**Type equality across isolation boundaries.** Section 4.5 states that type equality is value
equality and that the equality mechanism is projection-dependent. In runtimes that support module
isolation (Java's ClassLoader hierarchy, .NET's assembly loading, dynamic language module
reloading), the same abstract type may be represented by distinct runtime objects in different
isolation domains — for example, two
`Class<?>` objects loaded by different ClassLoaders in Java. Projections operating across such boundaries MUST ensure that their type equality mechanism produces consistent results. Approaches include: using interned string tags rather than runtime type metadata, establishing a canonical type registry shared across isolation domains, or restricting slot type comparison to a single domain. Projections that use runtime type metadata for slot matching (such as Java's `Class<?>`
with `==`) SHOULD document the boundary at which type identity is guaranteed — typically the single
ClassLoader or module layer. Cross-boundary slot matching is not addressed by this specification and
is a projection-specific concern.

### A.2 The Java Projection

The `Substrates.java` API in the reference implementation is one projection of this specification.
The following Java-specific decisions are **non-normative** — other projections are not bound by
them:

**Core binding decisions:**

- System-property provider selection with `ServiceLoader` fallback
- Static-final provider singleton initialized by Java class initialization
- `Substrates.cortex()` static entry point and `PROVIDER_PROPERTY` constant for provider selection
- Anonymous circuit and top-level scope factory calls use the Java runtime default name
  `Substrates`; subject identifiers provide uniqueness
- `AutoCloseable` on `Scope` (not on `Resource` directly) for resource cleanup
- `Class<?>` for type representation (`Subject.type()` returns `Class<S>`)
- `Routing` enum for the abstract routing selector (`PIPE` for per-pipe dispatch, `STEM` for
  hierarchical dispatch)
- `Stream` for extent and state traversal (`Extent.stream()`, `State.stream()` for `iterate()`);
  Window uses eager `forEach` and does not expose `Iterable`, `Iterator`, `Spliterator`, or
  `Stream`
- `CharSequence` for `Extent.path()` / `Name.path()` return types (spec uses abstract `String`)
- `NullPointerException` for absence violations
- `Fault` for synchronously detected substrate runtime errors and asynchronously reported isolated
  callback failures. `Fault.subject()` returns the closest known receiver/context subject, defaulting
  to the Circuit for callback failures, and `Fault.operation()` identifies the operation. The Java
  constructors accept `(Subject<?>, operation, message)` with an optional cause; Circuit fault
  receptors receive the original callback failure as that cause.
- `Optional<Pulse>` for `Circuit.pulse()`, where `Optional.empty()` represents a fully terminated
  circuit that could not be probed.
- Virtual threads for circuit context execution
- `@NotNull` annotations for absence constraints
- Nested interface style for API organization

**Tenure mapping:**

The Java projection uses a three-valued `@Tenure` annotation (`INTERNED`, `EPHEMERAL`, `ANCHORED`).
`Current` maps directly to `INTERNED`: each execution context has one stable Current token, and
references may be retained and shared for identity comparison and immutable subject inspection.
`Ticker` maps to `ANCHORED`: it is owned by its circuit and retained until the ticker or owning
circuit is closed.
`Closure` is marked `@Temporal` for its single-use contract. `Window` is marked `@Temporal` and
`@ReadOnly` for its callback-scoped inspection contract.

**Temporal violation detection:**

Registrar violations are detected via sentinel state. Since fibers and flows are standalone builder
values (not callback-scoped), there is no temporal contract on Fiber or Flow and therefore no
detection requirement. Window temporal violations are detected per the Window exception in §6.4.1;
the reference (alpha) SPI implements this via a per-operator lease check at `O(1)` cost per operator
entry. Detection lands on the user-callback side, not the framework emission hot path. The Java
projection still permits providers to back Window with mutable rolling storage and reuse Window
instances across callbacks — the lease check is what makes that reuse safe to detect.

**Non-normative API details, binding details, and extensions:**

The Java projection exposes these binding details and convenience operations beyond the abstract
surface:

- `Receptor.NOOP` — shared no-op receptor constant
- `Receptor.of()`, `Receptor.of(Class)`, `Receptor.of(Class, Receptor)` — typed no-op and identity
  factories for cleaner use with `var`
- `State.value(Slot<T>)` — template-based slot lookup as the Java projection of the abstract
  `State.value` operation
- `State.state(Slot<?>)` — direct slot insertion; `State.state(Enum<?>)` — enum convenience
- `Cortex.slot(Enum<?>)` — enum-to-name slot convenience
- `Extent.extent()`, `iterator()`, `fold()` / `foldTo()`, `stream()`, `within(...)`, and
  `enclosure(Consumer)` — extent traversal conveniences
- Multiple `Extent.path()` overloads — separator and mapper variants
- `Name` and `Subject` implement Java `Comparable`; this ordering is a Java binding detail, not a
  portable ordering contract
- `Name.name(Name)`, `name(Enum)`, `name(Iterable)`, `name(Iterator)`, and mapper variants —
  extender-shaped name conveniences; empty iterable/iterator input returns the receiver unchanged
- `Name.path(Function)` — name-part mapper convenience for custom path rendering
- `Subject.toString()` — string rendering via the subject's path representation
- Named overloads — `conduit(Name, ...)`, `scope()` (no-arg), `scope(Name)`, `circuit()`, and
  `circuit(Name)`
- `Cortex.name(Class)`, `name(Member)`, `name(Enum)`, `name(Iterable)`, `name(Iterator)`, and mapper
  variants — name creation conveniences. Class-array naming uses `Class#getCanonicalName()` when
  available and falls back to `Class#getName()` for arrays whose component type has no canonical
  name.
- `Cortex.fiber(Class<E>)` / `Cortex.flow(Class<E>)` — typed convenience overloads for
  `Cortex.fiber()` / `Cortex.flow()` that drive inference from a class witness
- `Lookup.get(Subject)` / `Lookup.get(Substrate)` — convenience overloads that extract the name
  from an existing substrate
- `Fiber.above(E)`, `below(E)`, `min(E)`, `max(E)`, `range(E, E)`, `deadband(E, E)`, and
  `clamp(E, E)` — natural-order convenience overloads for comparable emission values
- `Circuit.conduit(Class<E>)` / `Circuit.conduit(Name, Class<E>)` — Java binding forms for the
  abstract `Circuit.conduit(name?, type?)` operation; `Class` drives inference from a class witness
- `Circuit.conduit(Class<E>, Routing)` / `Circuit.conduit(Name, Class<E>, Routing)` — overloads
  selecting hierarchical dispatch via the `Routing` enum (see §10.3)
- `Circuit.bank(Class<E>)` — convenience factory for `Bank<Conduit<E>>` using per-pipe routing;
  `bank(Class<E>, Routing)` selects the routing behavior (see §10.4)
- `Circuit.basin(Class<E>, int)` — type-witness convenience equivalent to `basin(int)`; the class
  token drives Java inference and is not retained
- `Resource.closeAwait()` mechanics — the Java projection implements the normative blocking close
  (§9.1) by suspending the calling thread on the underlying virtual-thread valve barrier until the
  close job completes
- `@Abstract`, `@Experimental`, `@Extension`, `@Idempotent`, `@Identity`, `@Immutable`, `@New`,
  `@NotNull`, `@Provided`, `@Queued`, `@ReadOnly`, `@Temporal`, `@Tenure`, and `@Utility` —
  source-retained annotations documenting projection behavior
- `@SpecRef` and `@SpecDoc` — source-retained traceability metadata, defined by the separate
  `humainary-specs-api` artifact rather than by this API, since they belong to no single
  specification. `@SpecDoc` marks a type and names, by pinned URL, the document that references in
  that type's lexical subtree resolve against;
  `@SpecRef` names the content a declaration realizes or verifies. Projection-only behavior carries
  no `@SpecRef`

These are valid binding decisions for a Java projection but carry no weight for implementations in
other languages.

### A.3 Distributed Projections (Non-Normative)

A distributed projection spans multiple address spaces. Such projections must preserve:

- **Canonical identity**: Define a cross-boundary equivalence protocol (§1.2). Content-addressed
  hashing, distributed consensus, or application-level identity schemes may be appropriate.
- **Temporal contracts**: Callback boundaries (§6.4) must be clearly defined. Network round-trips
  during a temporal callback risk violating the contract.
- **Happens-before**: The ordering relations (§5.4.1) must extend across address space boundaries
  through whatever distributed ordering mechanism the projection employs.
- **Circuit-context confinement**: The single sequential execution guarantee (§5.1) must be
  preserved even if the circuit spans nodes.

---

*This specification describes what Substrates is — its structural primitives, behavioral contracts,
and lifecycle semantics — independent of any programming language or runtime. A conformant
implementation in any language is a projection of this specification, bound only by the behavioral
requirements, required types, and required operations defined herein.*

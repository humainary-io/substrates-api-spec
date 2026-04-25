# The Substrates Specification

**Version 2.3 — Draft**
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

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "
RECOMMENDED", "MAY", and "OPTIONAL" in this specification are to be interpreted as described in RFC

2119.

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
those recipes with type-changing transformations. Each primitive has a single, clear responsibility.

**Explicitness over magic.** Data flow is visible and traceable. There is no reflection-based
wiring, no implicit dependency injection, no hidden framework machinery. Every connection between
components is established through explicit operations defined in this specification.

## 3. Structural Overview

The topology of a Substrates system is a containment hierarchy.

A **Cortex** is the root. It is the entry point and factory for all runtime components. A cortex
creates circuits, names, scopes, and other foundational objects.

A **Circuit** is an execution engine owned by a cortex. Each circuit maintains a single sequential
execution context and a queue of pending work. All emission processing within a circuit occurs
within that context, sequentially.

A **Conduit** is a pipe factory owned by a circuit. A conduit creates and pools pipes by name — the
same name always resolves to the same pipe. Derived views over the pipe pool are created via
`pool(fn)`, caching the transformation result per name. A conduit is also a source — it supports
subscriptions that enable dynamic discovery of its named pipes.

A **Pipe** is an emission carrier. It accepts a typed value and delivers it to its target —
typically through the circuit's processing queue, but the delivery path depends on how the pipe was
created. Pipes are the primary mechanism for introducing values into the system.

A **Receptor** is a callback that receives emissions. Receptors are the consumption-side counterpart
to pipes.

The containment hierarchy is:

```
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
subject — this includes every cortex, current, pipe, scope, reservoir, subscriber, subscription, and
source (and therefore every circuit, conduit, and tap, which are sources). See the capability matrix
in §16.2 for the authoritative inventory. A subject carries:

- **Identifier**: A unique Id distinguishing this instance.
- **Name**: A hierarchical Name locating this component in the namespace.
- **State**: An immutable collection of typed slots (see §8).
- **Type**: The kind of substrate this subject represents (circuit, conduit, pipe, etc.).

Subjects form an extent (see §4.4). A pipe's enclosure is the subject of the source that owns it —
typically a conduit, but any Source subtype (Circuit, Conduit, Tap). The source's enclosure is in
turn the circuit's subject, which has the cortex's subject as its enclosure. This produces a
fully-qualified path for every component. The enclosure relation is the mechanism by which a
subscriber identifies the origin of an emission: a `Capture` carries the emitting pipe's subject,
and walking one step up the extent chain yields the source subject without requiring the source type
to be exposed on the emission path.

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
  `subject()`. Source *is* subject-bearing: being a source is precisely what confers identity on the
  subscribable branch of the hierarchy, so Source and its subtypes (Circuit, Conduit, Tap) all
  expose `subject()`. The Resource interface itself is not instantiated; concrete resource-bearing
  types expose `subject()` per §16.2. The type classifier is defined for all required types
  regardless; `Subject.type()` returns it only for types that have subjects. See §16.2 for the
  authoritative capability matrix.
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
`Subject.type()` / `Slot.type()` accessors. A language binding MAY expose additional capabilities (
display names, reflection metadata, hierarchical relationships) as non-normative extensions.

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
- Deterministic replay is achievable by replaying the input queue (see §5.6). The input queue
  includes all operations submitted to the circuit — emissions, subscription registrations,
  unsubscriptions, and close operations — not only emissions. Replaying emissions alone is
  insufficient because subscription operations affect pipeline topology (§7.6), which determines how
  subsequent emissions are routed.

The circuit context is an abstract concept. In a threaded runtime, it may be a dedicated thread. In
an async runtime, it may be a pinned task or coroutine. In an event-loop runtime, it may be a
sequence of non-preemptive turns. The implementation mechanism is not specified — only the
sequential, exclusive execution guarantee is REQUIRED.

### 5.2 Caller Context and Circuit Context

Operations in a Substrates system split between two execution contexts.

The **caller context** is any execution context that invokes operations on substrate components —
emitting values, creating conduits, subscribing to sources. Caller contexts enqueue work and return
immediately. They do not execute circuit logic.

The **circuit context** is the single sequential execution context owned by a circuit. It dequeues
work and processes it sequentially. All fiber/flow pipelines, subscriber callbacks, and emission
dispatches execute here.

The performance principle follows from this split: the circuit context is the bottleneck. Work
SHOULD be shifted to caller contexts where possible. Expensive computation, serialization, and
preparation SHOULD occur before emission. The circuit context SHOULD handle only routing, filtering,
and lightweight state updates.

### 5.3 Dual Queue Model

Each circuit maintains two queues.

The **ingress queue** receives emissions from caller contexts. It is a shared, concurrency-safe
queue. Caller contexts enqueue work here and return immediately.

The **transit queue** receives emissions generated during circuit-context processing. When
processing an emission triggers further emissions within the circuit context, those cascading
emissions enter the transit queue.

Processing priority: the transit queue MUST drain completely before the circuit context returns to
the ingress queue. This produces several properties:

- **Causality preservation**: All effects of an emission complete before the next external emission
  begins.
- **Stack safety**: Cascading emissions enqueue rather than recurse. Deeply cascading chains do not
  overflow the call stack.
- **Atomicity**: A cascading chain appears atomic to external observers.
- **Cyclic safety**: Feedback loops and recurrent topologies operate correctly. The queue breaks
  synchronous call chains that would otherwise produce infinite recursion.

When transit work itself emits, those emissions are added to the back of the transit queue. The
model is always iterative, never recursive.

### 5.4 Memory Model

This section defines the ordering and visibility guarantees of the specification in abstract terms,
independent of any language-specific memory model.

#### 5.4.1 Happens-Before Relations

The specification defines the following happens-before relations. These relations are the formal
basis for all ordering and visibility guarantees in the system.

1. **Emit → Process**: An emit operation on a caller context happens-before the processing of that
   emission within the circuit context.
2. **Process → Process**: Completion of emission processing within the circuit context
   happens-before the start of the next emission's processing.
3. **Transit → Ingress**: Completion of all transit queue processing happens-before the next ingress
   queue emission is dequeued.
4. **Process → Await**: Completion of all emissions enqueued before an await call happens-before the
   return of that await call to the caller context.

These relations are transitive: if A happens-before B and B happens-before C, then A happens-before
C.

#### 5.4.2 Visibility Guarantees

State modifications made within the circuit context during emission processing MUST be visible to:

- Subsequent emissions processed within the same circuit context (by relation 2).
- Caller contexts that complete an await operation enqueued after the modification (by relation 4).

State modifications made by a caller context before an emit operation MUST be visible to the circuit
context when processing that emission (by relation 1).

No other visibility guarantees are made. In particular, state modifications made within the circuit
context are NOT guaranteed visible to caller contexts that have not performed an await.

Implementations MUST provide these visibility guarantees through whatever synchronization mechanism
is appropriate for the target runtime — memory barriers, message passing, channel-based
communication, or runtime-specific ordering guarantees. The mechanism is not specified; the
guarantees are.

#### 5.4.3 Deterministic Ordering

Emissions MUST be observed in strict enqueue order. Earlier emissions MUST complete before later
ones begin. All observers within the circuit context MUST see emissions in the same sequence.

The enqueue order is determined by the concurrency-safe ingress queue. When multiple caller contexts
emit concurrently, the relative ordering of their emissions is determined by the queue's
synchronization mechanism and is not specified by this specification. Determinism is guaranteed
within a single caller context and for the total observed sequence — every run of the circuit
context sees a single, consistent order — but that order may differ across runs when multiple
callers emit concurrently.

**Scope of determinism**: This contract covers queue and dispatch ordering — the sequence in which
emissions are observed by receptors, fiber/flow stages, and subscribers. It does not extend to
value-level outcomes produced by nondeterministic user-supplied callbacks or operators. In
particular, probabilistic operators such as `Fiber.chance(probability)` (§6.2.3) are explicitly
allowed to produce different filter decisions across runs; the specification places no
reproducibility requirement on such operators. Implementations MAY offer seeded pseudo-random
variants as non-normative extensions.

### 5.5 Await

The **await** operation suspends the calling execution context until the circuit's queues are empty.
It establishes a happens-before relationship with all previously enqueued emissions (see §5.4.1,
relation 4).

Await is the primary coordination mechanism between caller contexts and the circuit context. After
await returns, all operations enqueued before the await call — emissions, subscription
registrations, unsubscriptions, and any other queued work — have completed, and their effects are
visible to the caller.

**Caller-side return is not effect-at-caller**: Queued operations (`emit`, `subscribe`,
`Subscription.close`, and any other operation that enqueues work per §5.1) return to the caller as
soon as the operation is enqueued. Their effect is not guaranteed to have taken place at the moment
of return. A caller that needs to observe the effect of a queued operation — for example, to verify
that a subscription has been installed, or that an emission has been dispatched — MUST call `await`
to synchronize. Callers MUST NOT assume that the effect of a queued operation is visible before
`await` returns.

Await MUST NOT be called from within the circuit context. An implementation that detects this
violation MUST signal an illegal context use error (§15.1). If detection is not feasible, the
behavior is undefined (deadlock is the likely outcome).

Multiple caller contexts MAY call await concurrently; each suspends independently. After circuit
closure, await MUST return immediately.

The suspension mechanism is runtime-dependent. In a threaded runtime, await may block the calling
thread. In an async runtime, it may yield the current task. In a distributed runtime, it may involve
a round-trip. The mechanism is not specified — only the synchronization and visibility guarantees.

### 5.6 Queue Capacity and Admission Control

The emit operation MUST NOT block the caller context. The emit operation MUST NOT discard the
emission while the circuit is open. Together these properties imply that the ingress queue is
effectively unbounded during normal operation — the specification does not define a capacity limit.

This is a deliberate consequence of the governing principle "determinism over throughput" (§2). A
bounded queue with blocking changes the ordering of caller context operations — a blocked caller
that retries creates a different enqueue sequence than one that succeeds immediately, violating
deterministic ordering (§5.4.3). A bounded queue with dropping discards emissions, violating the
total delivery guarantee that underlies deterministic replay (§5.1). Both mechanisms introduce
non-determinism that contradicts the foundational contract.

Admission control is addressed at the pipeline level, not the queue level. `Fiber.limit()` (§6.2.3)
caps the number of emissions processed by a given pipeline stage. `Fiber.every()` and
`Fiber.chance()` (§6.2.3) provide interval-based and probabilistic rate filtering respectively.
Both execute within the circuit context after
dequeue and therefore do not alter ingress ordering. These are the specified mechanisms for
controlling emission volume.

Implementations MAY provide bounded-capacity ingress queues with configurable load-shedding
policies (drop-newest, drop-oldest, reject-with-error) as non-normative extensions. Such extensions
MUST document their impact on the deterministic ordering (§5.4.3) and replay (§5.1) guarantees. An
implementation that offers bounded ingress queues under contention is no longer guaranteed to
satisfy §5.4.3 and MUST document this deviation.

## 6. Emission Path

The emission path describes how a value enters the system, is processed, and reaches consumers.

### 6.1 Pipe

A **Pipe** is the emission mechanism. It accepts a typed value through an `emit` operation and
delivers it to its target. The delivery path depends on how the pipe was created — conduit pipes (
created via `conduit.get(name)`) route through the circuit's queue, registered downstream pipes
execute within the circuit context during dispatch, and async circuit pipes are routed through the
circuit's queue to a target pipe or receptor.

Pipes are created by the framework, not by user code. They are obtained from channels, circuits, or
registration callbacks.

Pipes have no inherent concurrency contract — their execution context depends on how they were
created:

- Pipes obtained from channels route through the circuit's queue.
- Pipes created by circuits with a target pipe or receptor dispatch asynchronously via the circuit's
  queue.
- Registered downstream pipes execute within the circuit context when the named pipe dispatches.

The emit operation MUST NOT be called with an absent value (see §1.2). Implementations MUST signal
an absence violation error (see §15.2) if an absent value is detected.

### 6.2 Fiber and Flow

Two complementary types carry the emission-processing vocabulary:

- A **Fiber\<E\>** is a same-type per-emission recipe — a chain of stateless and stateful operators
  over a single type `E`. Fibers hold every per-emission operator (guard, diff, peek, reduce,
  every, chance, and the rest).
- A **Flow\<I, O\>** is the left-to-right composition surface. Flows carry the type-changing
  operator `map` plus two attachment operators (`fiber` attaches a Fiber at the output side;
  `flow` composes with another Flow). A Flow is optionally type-changing: `Flow<E, E>` is a
  valid same-type Flow.

Both Fibers and Flows are standalone immutable values obtained from cortex factories (§16.3). They
may be retained, shared across contexts, and materialized multiple times. Per-materialization state
for stateful Fiber operators is allocated at materialization, not at configuration.

All Fiber and Flow stages execute within the circuit context that materialized them. They process
emissions sequentially and deterministically. Exceptions raised by operator functions (predicates,
comparators, binary operators, mappers, peek receptors, fire predicates, etc.) are subject to the
external callback isolation rule in §15.4 — the circuit's dispatch loop MUST NOT be affected, and a
throwing operator is treated as having dropped the emission for that receptor chain.

Client-supplied operator functions — predicates, mappers, comparators, binary operators, key
functions, fire predicates, peek receptors, and any other callable passed to a Fiber or Flow
operator — MUST be **stateless** (pure functions of their arguments). A Fiber or Flow is a value
that MAY be materialized across multiple conduits and multiple circuits, and the operator function
instance is captured by the value and shared across every materialization. Any mutable state
captured inside an operator function is therefore shared across materializations — including across
circuits running on different threads — and subject to data races. The framework owns any state that
a stateful operator logically requires: Reduce holds the accumulator per materialization, Relate
holds the previous input per materialization, Diff holds the last-emitted value per materialization,
and so on. Implementations MUST NOT rely on operator functions to carry state of their own, and
clients MUST NOT fold additional state into the function itself.

#### 6.2.1 Fiber and Flow Typing

A Fiber is parameterized by a single emission type `E`. Every Fiber operator preserves that type: a
`Fiber<E>` operator returns a `Fiber<E>`. An **identity fiber** is a fiber with no operators — it
passes every value through unchanged. Identity fibers are obtained from the cortex factory (§16.3);
operators are chained onto an identity fiber to build a per-emission recipe.

A Flow is parameterized by two types: an **input type** `I` (the type of values entering the
pipeline from upstream) and an **output type** `O` (the type delivered downstream). The `map`
operator is **type-changing**: it appends a transformation downstream and returns a `Flow<I, P>`
where `P` is the new downstream-facing type. The `fiber` operator appends a same-type Fiber at the
output side, returning a `Flow<I, O>`. The `flow` operator composes with another Flow end-to-end,
producing a `Flow<I, P>`
when the composed flow's output type is `P`. An **identity flow** has `I == O`; it is obtained from
the cortex factory and is the starting point for building a Flow pipeline.

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

#### 6.2.3 Stateful Fiber Operators

Stateful operators maintain internal state across emissions. Because they execute exclusively within
the circuit context, their state requires no synchronization. State is allocated per
materialization; a single Fiber value materialized multiple times produces independent
per-materialization state.

**Diff**: Emits only when the current value differs from the previous emission. The first emission
always passes. An optional initial value MAY be provided for comparison with the first emission.
Equality is determined by the projection's standard value equality mechanism — `equals()` in Java,
`__eq__` in Python, `==` for value types in Go, `Eq` trait in Rust, or equivalent. Canonical
identity (§1.2) is not sufficient; value equality is REQUIRED.

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
operator. The result becomes both the new accumulator state and the emitted value. When the initial
value is absent, the accumulator starts absent and the operator is invoked with an absent first
argument on the first emission — callers who supply an absent initial value MUST ensure the operator
handles an absent accumulator without signaling an error.

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
once when the counter reaches `N`; otherwise the candidate resets to the new value with counter 1 (
and emits immediately if `N == 1`). For `count = 1`, Steady degenerates to Diff.

**Steady (bi-predicate)**: Generalized temporal-confirmation. A count and a bi-predicate are
provided. The bi-predicate compares each incoming value against the *candidate* — the first value
that started the current run — not against the immediately preceding value. This anchored comparison
detects drift: a signal that moves gradually away from the candidate resets the counter even if each
individual step is small.

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
without signaling an error. Distinguished from Reduce: Reduce emits on every step; Integrate
accumulates silently and emits only at fire points. Intended for batching, chunking, alternative-bar
construction, and spiking-neuron patterns.

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
`combiner` and emits the result. The first `size - 1` inputs are warm-up and produce no output. The
fold is recomputed in O(size) per emission, enabling any `BinaryOperator` including non-invertible
ones (max, min, concatenation).

**Tumble**: Count-triggered aggregation. A positive integer `size`, a binary combiner, and an
identity value are provided. A running accumulator starts at `identity`. Each emission is folded
into the accumulator via `combiner`. After exactly `size` emissions the accumulator is emitted
downstream and reset to `identity`; the intermediate `size - 1` emissions produce no output. If the
stream ends with fewer than `size` inputs since the last emission, the in-flight partial window is
never emitted. Distinguished from Rolling: Rolling produces overlapping windows (one emission per
input after warm-up); Tumble produces non-overlapping windows (one emission per `size` inputs).
Distinguished from Integrate: Integrate fires when the accumulator state satisfies a predicate;
Tumble fires after a fixed number of inputs regardless of content.

**Every (interval)**: Emits every Nth value, dropping others. For example, every(3) emits the 3rd,
6th, 9th values.

**Chance (probability)**: Probabilistically passes each emission with the specified probability. A
probability of 0.0 drops all emissions. A probability of 1.0 passes all emissions. Per-emission
decisions are made using an implementation-defined uniform random source. Reproducibility across
runs, across materializations of the same Fiber value, or across separate channels is NOT required —
implementations MAY use a non-deterministic source (e.g., a freshly seeded PRNG per
materialization). Applications requiring deterministic sampling SHOULD use Every (interval)
instead.

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

Flow operators follow a **left-to-right builder** convention for type-changing stages: `map`
appends a transformation downstream of this flow. Its signature — `(O) → P` — maps the current
output type `O` into a new downstream type `P`. The textual chain order matches execution order.
Reading a flow chain from emit side to downstream: successive `map` stages produce successive
downstream type-widenings; a `fiber(Fiber)` at the output side runs its Fiber operators last on
each emission before dispatch.

#### 6.2.6 Pipe Composition

Both Fibers and Flows are applied to a pipe via composition methods on the composition values
themselves:

- `fiber.pipe(pipe)` returns a new pipe of the same type `E` whose emissions are processed by the
  fiber's operator chain before reaching the original pipe. This is the direct attachment for
  type-preserving recipes; no widening through Flow is required.
- `flow.pipe(pipe)` returns a new pipe whose input type is the flow's upstream type `I` — which may
  differ from the original pipe's type when the flow carries a `map`. The flow's stages (
  including any attached Fiber) execute before dispatch to the original pipe.

Each composed pipe has its own independent per-materialization state, ensuring stateful operators (
diff, reduce, limit, integrate, relate, etc.) maintain separate state per composed pipe. All
processing executes within the circuit context.

Fibers and Flows are constructed from identity values obtained from the cortex factory (§16.3), have
operators chained onto them, and are then handed a pipe via `fiber.pipe(...)` / `flow.pipe(...)`.
The value itself is independent of any pipe until composed. After composition, the returned pipe is
the only reference to the underlying materialized state.

### 6.3 Pipe Dispatch

After an emission passes through all fiber/flow stages, the named pipe dispatches it to all
registered downstream pipes. Registered pipes are invoked sequentially in registration order within
the circuit context. An exception thrown by any one downstream pipe or receptor MUST NOT prevent
dispatch to siblings registered on the same channel; see §15.4.

### 6.4 Temporal Contracts

Several types in the specification carry a **temporal contract** — they are valid only during a
bounded lifetime and MUST NOT be used beyond that lifetime. The specification defines three temporal
lifetime scopes:

**Callback-scoped**: Valid only during a specific callback invocation. The object MUST NOT be
retained after the callback returns.

- **Registrar**: Valid only during the subscriber callback.

**Execution-context-scoped**: Valid only within the execution context that obtained the object. The
object MUST NOT be used from a different context.

- **Current**: Valid only within the execution context that called `Cortex.current()`.

**Single-use-scoped**: Valid for exactly one invocation of its primary operation. After that
invocation, further use is illegal. Validity is also bounded by the owning scope's lifetime — if the
scope closes before the operation is invoked, the operation becomes a no-op.

- **Closure**: Valid for exactly one call to `consume`. Subsequent calls are illegal. If the owning
  scope is already closed, `consume` MUST be a no-op.

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
violations; undefined behavior is not an acceptable choice for this type. The general SHOULD-detect
framework above remains applicable to other temporal types whose validity-checks would lie on a hot
path.

Implementations MAY reuse callback-scoped temporal objects across callbacks. This reuse is safe
precisely because the temporal contract prohibits retention.

## 7. Subscription Model

The subscription model enables dynamic discovery of channels and adaptive topology construction.

### 7.1 Source

A **Source** is any component that supports subscription. Sources have identity (via
substrate/subject) and emit events. The specification defines three source types: circuits (which
emit state values), conduits (which emit domain values through channels), and taps (which emit
transformed values).

A conformant implementation MUST provide the Source operations (`subscribe`, `reservoir`) on
Circuit. Circuit state emission semantics for lifecycle transitions are defined in §7.1.1.
Additional state emissions beyond lifecycle transitions are implementation-defined. Implementations
SHOULD document what additional state events a circuit emits and under what conditions.

#### 7.1.1 Circuit State Emissions

A circuit acting as a source SHOULD emit state values representing lifecycle transitions. The
defined lifecycle phases are:

- **Active**: The circuit is open and processing emissions. This is the initial phase. An
  implementation MAY emit an active state upon subscription, but is not REQUIRED to.
- **Closing**: The circuit has been marked for closure (§9.3, step 1). New emissions from caller
  contexts are no longer processed.
- **Closed**: The circuit context has terminated (§9.3, step 4). All resources are released.

Lifecycle state emissions MUST be processed within the circuit context. A closing state emission
MUST happen-before the circuit context terminates. This is consistent with transit queue priority (
§5.3) — lifecycle transitions are circuit-internal events and SHOULD be dispatched via the transit
queue, ensuring they complete before pending ingress emissions.

The emitted value is a State (§8). The specific slot structure representing the lifecycle phase is
projection-dependent. Projections SHOULD define a canonical slot name and type for the lifecycle
phase classifier.

Implementations MAY emit additional state values representing topology changes (conduit creation,
subscription changes), processing metrics (queue depth, emission count), or other operational
information. These additional emissions are non-normative and MUST be documented by the
implementation.

**Minimum testable contract**: A conformant implementation MUST satisfy the following on the circuit
lifecycle source, independent of any projection-specific state slot structure:

1. **Closing-before-terminated ordering**: A subscriber registered to a circuit's lifecycle source
   before the circuit begins closing MUST observe the closing transition before the circuit context
   terminates. This is a direct consequence of §5.3 transit queue priority — lifecycle transitions
   are circuit-internal and dispatch via the transit queue, so they MUST complete before the context
   drains and terminates.
2. **Single-closing guarantee**: For any given circuit instance, the closing transition MUST be
   emitted at most once. Repeated close calls are idempotent (§9.1) and MUST NOT produce additional
   lifecycle emissions.
3. **No post-terminated emissions**: Once the circuit context has terminated, no further lifecycle
   state values MUST be delivered to any subscriber on that circuit.

Beyond this minimum, additional conformance testing (e.g., the exact state slot representation,
emission of an `active` phase on subscription, or ordering relative to other circuit emissions) is
deferred to a future revision. The lifecycle phase definitions above establish the semantic
framework for that future work; the three requirements above are testable today and are part of the
2.0 conformance surface.

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
NOT proceed, MUST NOT enqueue work, and MUST NOT install the subscriber on the source. This is
distinct from the queued-operation drop semantics of §9.1 (post-close): those exist because
close/emit races are inherent and unknowable at call time, whereas cross-circuit misuse is a
deterministic programming error that silent behavior would render undiagnosable — a caller doing
`source.subscribe(foreignSub); circuit.await();` and observing zero callbacks would have no way to
distinguish a dropped subscription from one that simply hasn't delivered yet (§5.5).

The check MUST happen before the subscription is registered with the source and before any pipe is
created on behalf of the subscriber, so that a rejected subscription leaves no observable side
effects on either circuit.

### 7.3 Subscription Lifecycle

The subscription lifecycle proceeds as follows:

1. A subscriber is created from a circuit, receiving a name and a callback function.
2. The subscriber is subscribed to a source. This returns a subscription handle. The subscribe
   operation is asynchronous — it enqueues a registration job to the circuit context and returns
   immediately. The registration does not take effect at the moment of return; its effect becomes
   observable only after the circuit context processes the registration job (see §5.5 and §7.6).
3. When a named pipe within the source processes an emission whose ingress-queue enqueue position
   falls within the subscription's visibility window (§7.6), and the named pipe has not yet rebuilt
   for this subscription, a rebuild is triggered within the circuit context.
4. During rebuild, the subscriber's callback is invoked with the pipe's subject and a registrar.
5. The callback uses the registrar to attach downstream pipes or receptors.
6. Registered pipes receive the emission that triggered the rebuild and all subsequent emissions
   from that named pipe that fall within the subscription's visibility window.

Key properties of this model:

- **Lazy invocation**: Callbacks are invoked on first emission, not on pipe creation. Creating a
  pipe via `get(name)` does not trigger callbacks. This avoids overhead for pipes that never emit.
- **Dynamic discovery**: Subscribers do not need prior knowledge of pipe names. They discover pipes
  as those pipes become active.
- **Circuit-context execution**: All callbacks execute within the circuit context. No
  synchronization is needed within callback logic.

### 7.4 Registrar

A **Registrar** is a temporal handle provided during a subscriber callback. It allows downstream
pipes or receptors to be attached to the named pipe identified by the callback's subject.

Registration semantics:

- Multiple downstream pipes MAY be registered for the same named pipe. All receive every emission (
  fan-out).
- Registration order determines invocation order.
- The same pipe instance MAY be registered multiple times, creating independent callbacks.
- The registrar carries a temporal contract (see §6.4). Using it after the callback returns is an
  illegal operation.

### 7.5 Subscription Handle

A **Subscription** is a cancellable handle returned by the subscribe operation. Closing a
subscription:

- Stops all future subscriber callbacks from this subscription.
- Removes all pipes registered by this subscription from active channels.
- Is idempotent — repeated close calls MUST be safe.

Subscription close is an asynchronous operation: it enqueues a close job to the circuit context and
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

A subscription's **visibility window** is the half-open interval of ingress-queue positions
`[subscribe_enqueue, close_enqueue)`, where `subscribe_enqueue` is the ingress-queue position of the
subscription's registration job and `close_enqueue` is the ingress-queue position of the
subscription's close job (or unbounded if the subscription has not been closed).

A subscription receives exactly those emissions whose ingress-queue enqueue position falls within
this window. Emissions enqueued before the subscribe job, or at-or-after the close job, are not
visible to the subscription.

This is a direct consequence of the FIFO ordering of the input queue (§5.1, §5.4.3). Subscribe,
emit, and close are all enqueued operations, and the circuit context processes them in enqueue
order. A single caller context that executes `subscribe(); emit(X); await()` observes the emission
`X` on the new subscriber because, by per-caller FIFO, the subscribe job is enqueued before the emit
job; the circuit processes them in that order; the registration is installed before `X` is
dispatched. Conversely, a caller that executes `emit(X); subscribe(); await()` does not observe `X`
on the new subscriber, because `X` is enqueued before the registration job and is dispatched before
the subscription is installed. Symmetrically for close:
`subscribe(); emit(X); close(); emit(Y); await()` delivers `X` but not `Y` to the subscription.

Across multiple concurrent caller contexts, the relative enqueue order of their operations is
determined by the ingress queue's synchronization mechanism and is not specified (§5.4.3). The
visibility window rule still applies — each subscription's window is defined by its own enqueue
positions — but the set of emissions that fall within a given window may differ across runs when
multiple callers are racing.

#### 7.6.2 Version-Tracked Lazy Rebuild

Subscription changes (add/remove) use lazy rebuild with version tracking. A version counter is
incremented when subscriptions change. Named pipes detect version mismatches on their next emission
and rebuild their downstream pipe lists.

This produces eventual consistency: subscription changes are not immediately visible to all named
pipes. Each named pipe discovers the change on its next emission. The benefit is lock-free operation
with minimal coordination overhead. The visibility window rule in §7.6.1 is preserved because the
version increment is itself processed in ingress-queue order within the circuit context.

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
existing entry MUST replace that entry in place (upsert), removing the prior slot. An implementation
MAY return the same state instance when the operation would produce a semantically equivalent
result (when an equal slot with the same value already exists). ("Persistent" here refers to the
persistent data structure property of structural sharing, not to durable storage.)

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

This set defines the minimum required slot types. Projections MAY support additional slot types (
e.g., unsigned integers, byte arrays, timestamps) as non-normative extensions, provided the required
types are supported.

## 9. Lifecycle Management

### 9.1 Resource

A **Resource** is any component with explicit cleanup requirements. The close operation releases
resources and terminates associated operations.

Close semantics:

- **Idempotent**: The first call performs cleanup. Subsequent calls MUST be no-ops.
- **Concurrency-safe**: Close MAY be called from any execution context.
- **Queued execution**: For circuit-managed resources, close submits a cleanup job to the circuit
  context and returns immediately. The actual cleanup occurs asynchronously.
- **Post-close operation semantics**: Operations on a resource whose close has been queued or
  processed MUST NOT throw an error in the caller context. Because `emit`, `subscribe`, `close`, and
  similar operations are queued and non-blocking (§5.5, §5.6), the caller thread has no
  deterministic way to observe whether a resource has been closed at the moment of an operation
  call, and the implementation has no synchronous opportunity to report an error to the caller.
  Queued operations whose ingress-queue position falls strictly after the component's own close job
  MUST be silently dropped by the component when processed on the circuit context — no delivery, no
  topology change, no observable effect beyond any onClose-style callback symmetry defined by the
  specific operation type. The component MAY additionally report the drop via a side channel (e.g.,
  a fault emission or a circuit-state event on the circuit thread). Operations enqueued strictly
  *before* the close job — pre-close operations still being drained — MAY be processed to completion
  or silently dropped at the implementation's discretion; implementations MUST document this choice.
  Callers that need to confirm no further operations will take effect MUST use `await` (§5.5) for
  synchronization.
- **Synchronous operation semantics**: The queued-operation rule above applies to non-blocking
  operations that enqueue onto the circuit context. Operations that are synchronous by nature —
  factory operations (e.g. `Circuit.conduit`) and other mutator operations that MUST return a value
  to the caller — fall outside it, because the caller observes the result of the operation
  synchronously. After close, an implementation MAY raise a projection-appropriate error, or return
  an inert instance whose subsequently-invoked queued operations are themselves dropped per the rule
  above. Implementations MUST document their choice for each such operation.

Types satisfying the closeable-lifecycle role defined by this specification: Source (and therefore
its subtypes Circuit, Conduit, and Tap), Subscriber, Subscription, Reservoir, and Scope. Because
Conduit is a Source, it is closeable — closing a conduit releases its pipe pool and cancels any
outstanding subscriptions to it, subject to the post-close operation semantics above. Scope
satisfies the closeable-lifecycle role but is not a Source (§9.2) — it provides hierarchical
resource management rather than subscribable emissions. The spec does not mandate any specific
projection mechanism for this role: a projection MAY realize it via a dedicated `Resource`
interface, via the host language's standard closeable type (e.g., Java's `AutoCloseable`), or via
any equivalent affordance. See §16.2 for the authoritative capability matrix.

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
   side channel (e.g., a circuit-state event or fault emission on the circuit thread), or use any
   other mechanism that prevents processing without surfacing an error to the caller thread. An
   implementation MUST document its post-close emission behavior.
2. The circuit context is signaled to terminate. The circuit SHOULD emit a closing lifecycle state
   to its subscribers (see §7.1.1) before terminating.
3. Emissions already enqueued at the time of marking MAY or MAY NOT complete — this is explicitly
   implementation-defined. An implementation MUST document its choice.
4. After the circuit context terminates, conduits and their channels are released.

Close is non-blocking — it marks the circuit for closure and returns immediately.

**Portable quiescence**: The portable way to guarantee that all pending work completes before
shutdown is to call `await()` before `close()`. After `await()` returns, all previously enqueued
emissions have completed (§5.5). A subsequent `close()` then operates on an empty queue, making the
drain question moot. Implementations SHOULD treat this as the recommended shutdown sequence.

After `close()`, `await()` MUST return immediately without suspending.

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
result, caching the transformed result per name. Derived pools enable domain-specific wrappers (
e.g., `conduit.pool(Sensor::new)`) and decorated pipes (e.g., `conduit.pool(fiber::pipe)`), while
preserving the identity guarantee: the same name in a derived pool MUST always yield the canonically
identical transformed result.

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

## 11. Derived Structures

### 11.1 Tap

A **Tap** mirrors a source's structure with transformed emissions. A tap is created from any source
with a **pipe function** that wires the tap's target pipe to the source's emission type. For each
named pipe in the source, the tap creates a corresponding mirrored pipe with its own distinct
subject. The mirrored pipe's subject carries the same name as the source pipe's subject but has a
distinct identity (different Id) — tap pipes are not the same entities as their source pipes.

The pipe function receives the tap's target pipe (`Pipe<T>`) and returns a source-compatible pipe (
`Pipe<E>`). This enables type transformation and per-emission processing to be composed within the
function via pipe composition (`Flow.pipe(Pipe)`, `Fiber.pipe(Pipe)`), rather than being
configured as separate parameters on the tap itself.

Taps are resources — closing a tap unsubscribes from the source and releases mirrored pipes.

Taps are also sources — they support their own subscribers, enabling chained transformations.

### 11.2 Reservoir

A **Reservoir** is an in-memory buffer that captures emissions from a source. It is created from a
source and subscribes to that source, recording each emission along with the subject of the pipe
that produced it, forming captures.

A **Capture** pairs an emitted value with the subject of the pipe that emitted it. The subject is
typed by the pipe role — specifically `Subject<Pipe<E>>` — distinguishing it from the subjects of
other substrate components such as conduits or sources.

The drain operation returns all captures accumulated since the last drain (incremental). Drain has
the following behavioral requirements:

- **Snapshot semantics**: Drain returns a fixed collection of captures. The returned collection MUST
  NOT be affected by emissions that arrive after drain is invoked.
- **Ordering guarantee**: Captures MUST appear in emission order — the order in which the circuit
  context processed the emissions.
- **Single-context requirement**: Drain MUST be called from a single execution context. This is a
  conformance requirement — concurrent drain calls from different contexts produce undefined
  behavior. Implementations are not required to detect violations.
- **Incremental**: Each drain returns only captures accumulated since the previous drain (or since
  creation, for the first drain). Captured values are released after drain returns.

The typical usage pattern is: `await()` to ensure all pending emissions are processed, then
`drain()` to retrieve the captures. Without a preceding `await()`, the set of captures is
non-deterministic — it includes whatever the circuit context has processed at the time of the call.

Reservoirs are resources — closing a reservoir unsubscribes from the source and releases the buffer.

### 11.3 Current

A **Current** represents the execution context from which substrate operations originate. It is a
context-local reference — each execution context has at most one current, and obtaining it is
observational (it does not create or modify state).

Current carries a temporal contract — it is valid only within the execution context that obtained it
and MUST NOT be used from a different context. Current is identity-bearing (it has a subject) but is
not required to be serializable or stable beyond the lifetime of its execution context. It is purely
observational — it identifies the calling context for the substrate, not for external systems.

The projection mechanism is runtime-dependent — `Thread.currentThread()` in Java, goroutine-local
state in Go, coroutine context in Kotlin, isolate reference in Dart, or equivalent. The mechanism is
not specified; only the identity and temporal guarantees are REQUIRED.

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

- **Interned**: Instances are pooled by key. The same key yields a canonically identical result (
  §1.2). The owning container maintains a reference.
- **Ephemeral**: Instances are not cached by their creator. Each creation is independent. Only
  external references keep them alive.
- **Anchored**: Tenure depends on attachment. Standalone instances are ephemeral; attached instances
  are retained by the substrate they are bound to.
- **Context-scoped**: A single instance exists per execution context for the context's lifetime.
  Unlike interned types, context-scoped instances are not pooled globally — they are bound to the
  context that produced them and MUST NOT be retained or shared across contexts.

Tenure classifications for the core types:

| Type         | Tenure         |
|--------------|----------------|
| Cortex       | Interned       |
| Circuit      | Ephemeral      |
| Conduit      | Ephemeral      |
| Pipe         | Anchored       |
| Subscriber   | Anchored       |
| Subscription | Anchored       |
| Reservoir    | Ephemeral      |
| Tap          | Ephemeral      |
| Scope        | Anchored       |
| Name         | Interned       |
| Id           | Interned       |
| Subject      | Anchored       |
| State        | Anchored       |
| Slot         | Anchored       |
| Capture      | Ephemeral      |
| Closure      | Ephemeral      |
| Current      | Context-scoped |

## 14. Async Pipe Dispatch

Circuits can create pipes that asynchronously dispatch emissions to a target pipe or receptor via
the circuit's queue. This breaks synchronous call chains and enables:

- Deep hierarchical propagation without stack overflow.
- Cyclic topologies (feedback loops, recurrent networks).
- Offloading work from caller contexts to the circuit context.

When creating an async pipe where the target already belongs to the same circuit, the target SHOULD
be returned as-is — no additional indirection is added.

Per-emission processing is applied as a separate step via pipe composition (§6.2.6):
`fiber.pipe(circuit.pipe(target))` or `flow.pipe(circuit.pipe(target))` produces a pipe whose
emissions are filtered, transformed, and rate-limited before crossing the circuit queue boundary.
All operator stages execute within the circuit context after emissions are dequeued. Composition is
orthogonal to async dispatch: any pipe (obtained from conduit `get`, circuit `pipe`, or subscriber
`register`) can serve as the target of `Fiber.pipe(pipe)` / `Flow.pipe(pipe)` to add a processing
stage upstream of it.

## 15. Error Model

The specification defines an abstract error model with portable categories. Each category represents
a class of contract violation. Some categories require unconditional detection; others permit
undefined behavior where detection would degrade valid-path performance. The enforcement level for
each category is specified in §15.1. The signaling mechanism is projection-dependent — exceptions in
Java/C#/Python, error returns in Go, `Result` types in Rust, rejected futures in async runtimes, or
service-level error codes in networked projections. The categories are normative; the mechanism is
not.

### 15.1 Error Categories

| Category                 | Enforcement                               | When Signaled                                                                                                                                                                                                                                                         |
|--------------------------|-------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Provider mismatch**    | MUST detect                               | Objects from incompatible provider implementations are mixed — e.g., a name created by one cortex is passed to a circuit created by a different cortex.                                                                                                               |
| **Absence violation**    | MUST detect                               | A required parameter receives an absent value (§1.2) where one is not permitted.                                                                                                                                                                                      |
| **Illegal temporal use** | SHOULD detect (MUST detect for Registrar) | A temporal object (§6.4) is used after its callback scope has ended. Detection is RECOMMENDED in general; undefined behavior is acceptable where detection would degrade valid-path performance. **Registrar violations MUST be detected and signaled** — see §6.4.1. |
| **Illegal context use**  | SHOULD detect                             | An operation is invoked from a prohibited execution context — e.g., `await()` called from within the circuit context (§5.5). Detection is RECOMMENDED where feasible; if not feasible, the behavior is undefined.                                                     |
| **Configuration error**  | MUST detect                               | An error occurs during fiber or flow configuration. Configuration errors MUST propagate to the caller. Configuration occurs eagerly when pipes, conduits, or taps are created.                                                                                        |

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
   subscriber callback MUST NOT leave partially-registered pipes visible as if registration had
   completed normally — implementations MAY either discard the partial registration or retain
   whatever was registered before the exception, but MUST NOT present a partial state as a completed
   one.

4. **Observability**. Implementations MAY surface failures via a fault channel, a circuit state
   event, a configurable error handler, a logging facility, or any combination of these. This
   specification does not mandate a specific observation mechanism, and implementations MAY elect to
   report no information beyond what the language or runtime already provides (e.g.,
   uncaught-exception handlers). Applications that require structured fault observation SHOULD rely
   on the mechanism documented by their chosen implementation.

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
2. **Deterministic ordering**: Emissions MUST be observed in strict enqueue order. Earlier emissions
   MUST complete before later ones begin.
3. **Dual queue priority**: The transit queue MUST drain completely before the ingress queue is
   consulted.
4. **Name interning**: Structurally equivalent names MUST resolve to canonically identical names (
   §1.2).
5. **Pipe pooling**: Named pipes with the same name within a conduit MUST be canonically identical (
   §1.2).
6. **Lazy rebuild**: Subscription changes MUST propagate to named pipes on their next emission via
   version-tracked rebuild.
7. **Temporal contracts**: Registrar instances MUST be valid only during the subscriber callback in
   which they are provided. Implementations MUST detect and signal Registrar temporal contract
   violations (see §6.4.1). For other temporal types, enforcement follows the general SHOULD-detect
   framework.
8. **Idempotent close**: All resource close operations MUST be idempotent and concurrency-safe.
9. **Scope ordering**: Resources registered with a scope MUST close in reverse registration order.
10. **State immutability**: State operations MUST preserve immutability — prior instances MUST
    remain unchanged. Implementations MAY return the same instance for semantically equivalent
    writes (e.g., upserting a slot whose name, type, and value match an existing entry).
11. **Happens-before guarantees**: Implementations MUST provide the happens-before relations defined
    in §5.4.1 through appropriate synchronization mechanisms.
12. **Await from circuit context**: Calling await from within the circuit context MUST signal an
    illegal context use error per the enforcement level in §15.1 (SHOULD detect; MUST signal when
    detection is feasible).
13. **External callback isolation**: Uncaught exceptions from client-supplied callbacks (receptors,
    subscriber callbacks, onClose handlers, Fiber/Flow operator functions) MUST NOT propagate to the
    circuit's dispatch loop. Dispatch MUST continue to sibling receptors on the same channel and to
    subsequent queued operations. See §15.4.
14. **Operator function purity**: Client-supplied Fiber and Flow operator functions MUST be
    stateless (pure functions of their arguments). A Fiber or Flow value MAY be materialized across
    multiple conduits and multiple circuits; operator function instances are shared across
    materializations. Implementations MUST rely on the framework-owned per-materialization state for
    stateful operators (Reduce's accumulator, Relate's previous input, Diff's last-emitted value,
    etc.) and MUST NOT require operator functions to carry state of their own. See §6.2.

### 16.2 Required Types

A conformant implementation MUST provide concrete realizations of the following abstract types:

| Type         | Category     | Description                             |
|--------------|--------------|-----------------------------------------|
| Cortex       | Factory      | Root entry point and factory            |
| Circuit      | Execution    | Sequential execution engine             |
| Conduit      | Routing      | Pipe factory, pool, and source          |
| Pool         | Routing      | Composable name-based view              |
| Pipe         | Emission     | Emission carrier                        |
| Receptor     | Consumption  | Emission callback                       |
| Fiber        | Pipeline     | Same-type per-emission operator chain   |
| Flow         | Pipeline     | Input-side composition surface          |
| Subscriber   | Subscription | Pipe discovery callback                 |
| Subscription | Subscription | Cancellable subscription handle         |
| Registrar    | Subscription | Pipe attachment handle (temporal)       |
| Source       | Subscription | Subscribable event origin               |
| Tap          | Derived      | Transformed source mirror               |
| Reservoir    | Derived      | Emission buffer                         |
| Capture      | Derived      | Emission-subject pair                   |
| Current      | Context      | Execution context reference (temporal)  |
| Name         | Identity     | Hierarchical interned identifier        |
| Id           | Identity     | Opaque unique token                     |
| Subject      | Identity     | Component identity record               |
| Extent       | Identity     | Hierarchical containment structure      |
| State        | Data         | Immutable slot collection               |
| Slot         | Data         | Named typed value                       |
| Scope        | Lifecycle    | Hierarchical resource manager           |
| Closure      | Lifecycle    | Block-scoped resource handle (temporal) |
| Resource     | Lifecycle    | Closeable component                     |

Types marked **(temporal)** carry temporal contracts as defined in §6.4. The specific lifetime
scope (callback-scoped, execution-context-scoped, or single-use-scoped) is specified per type in
§6.4.

#### Type Capability Matrix

The following matrix is the authoritative inventory of which required types are substrate
components (expose `subject()`), subscribable origins (satisfy the Source contract), closeable
resources (satisfy the Resource contract), and temporal. All other sections describing these
capabilities are derived from this table.

| Type         | Substrate (has subject) | Source | Resource (closeable) | Temporal |
|--------------|:-----------------------:|:------:|:--------------------:|:--------:|
| Cortex       |            ✓            |        |                      |          |
| Circuit      |            ✓            |   ✓    |          ✓           |          |
| Conduit      |            ✓            |   ✓    |          ✓           |          |
| Tap          |            ✓            |   ✓    |          ✓           |          |
| Source       |            ✓            |   ✓    |          ✓           |          |
| Pipe         |            ✓            |        |                      |          |
| Current      |            ✓            |        |                      |    ✓     |
| Scope        |            ✓            |        |          ✓           |          |
| Reservoir    |            ✓            |        |          ✓           |          |
| Subscriber   |            ✓            |        |          ✓           |          |
| Subscription |            ✓            |        |          ✓           |          |
| Registrar    |                         |        |                      |    ✓     |
| Receptor     |                         |        |                      |          |
| Fiber        |                         |        |                      |          |
| Flow         |                         |        |                      |          |
| Pool         |                         |        |                      |          |
| Capture      |                         |        |                      |          |
| Name         |                         |        |                      |          |
| Id           |                         |        |                      |          |
| Subject      |                         |        |                      |          |
| Extent       |                         |        |                      |          |
| State        |                         |        |                      |          |
| Slot         |                         |        |                      |          |
| Closure      |                         |        |                      |    ✓     |
| Resource     |                         |        |                      |          |

Notes:

- **Source** is the junction point where identity and lifecycle meet for the subscribable branch:
  being a Source is what confers `subject()` and `close()` on Circuit, Conduit, and Tap. Source
  itself is an abstract role — projections expose it via whichever mechanism is idiomatic (interface
  inheritance, trait, protocol conformance).
- **Resource (closeable)** in the table column refers to the abstract closeable-lifecycle role —
  satisfying the Resource contract (§9.1) and exposing `close()`. The spec does not mandate any
  specific projection mechanism: a projection MAY realize this role via a dedicated `Resource`
  interface, via the host language's standard closeable type (e.g., Java's `AutoCloseable`), or via
  any equivalent affordance. Rows marked Resource ✓ are the types that MUST satisfy the role.
- **Scope** satisfies the closeable-lifecycle role but is *not* a Source — it provides hierarchical
  resource management (§9.2), not subscribable emissions. (In the Java projection, Scope binds
  directly to `AutoCloseable` rather than implementing the `Resource` interface — see Appendix A.2.)
- **Fiber**, **Flow**, **Receptor**, and **Pool** are abstract protocols (standalone values or
  structural types), not substrate components; they have no `subject()` of their own.
- **Capture**, **Name**, **Id**, **Subject**, **Extent**, **State**, and **Slot** are value or
  identity records, not substrate components; they have no `subject()` of their own.
- Types with `(temporal)` in §6.4 — **Registrar**, **Current**, **Closure** — are marked in the
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
- `Ordering` denotes a three-valued comparison result (less than, equal to, greater than). See
  §6.2.4.
- Scalar type names (`Boolean`, `Integer`, `Double`) are specification vocabulary matching §8.3.
  They are not Java wrapper class names — projections use native types as specified in the §8.3
  table.

#### Universal Operations

The operations below are inherited by every type that carries the corresponding capability in the
§16.2 matrix, and are therefore NOT repeated in each type's operation block. A conformant
implementation MUST expose them on every matching type.

```
<any Substrate ✓ type>.subject(): Subject<S>
<any Resource ✓ type>.close()
```

- **`subject()`** — returns the identity record of this substrate component (§4.3). The type
  parameter `S` is the type of the receiving substrate, so the returned subject is typed to its
  bearer (e.g., `Subject<Circuit>`, `Subject<Pipe<E>>`). Every type marked Substrate ✓ in §16.2 —
  Cortex, Circuit, Conduit, Tap, Source, Pipe, Current, Scope, Reservoir, Subscriber, Subscription —
  MUST expose `subject()`.
- **`close()`** — releases resources and terminates associated operations with the semantics of
  §9.1 (idempotent, concurrency-safe, post-close drop). Every type marked Resource ✓ in §16.2 —
  Circuit, Conduit, Tap, Source, Scope, Reservoir, Subscriber, Subscription — MUST expose `close()`.

Per-type operation blocks below list only operations that are *specific* to that type. Where a block
shows `close()` or `subject()` explicitly (for example in the Source and Circuit blocks), it is
included for clarity of the surrounding prose, not to exempt other types from the universal
contract. Types whose only operations are the universal ones have an operation block that notes this
explicitly rather than being omitted.

#### Factory Operations

**Cortex** — root factory:

```
Cortex.circuit(name?: Name): Circuit
Cortex.name(value: String): Name
Cortex.scope(name?: Name): Scope
Cortex.current(): Current
Cortex.state(): State
Cortex.slot(name: Name, value: T): Slot<T>
Cortex.fiber(): Fiber<E>
Cortex.flow(): Flow<E, E>
Cortex.flow(fiber: Fiber<E>): Flow<E, E>
```

The `slot` operation MUST support all types defined in §8.3. The `fiber` operation returns an empty
identity fiber over a single emission type `E`; per-emission operators are chained onto it to build
a recipe (§6.2), which is then attached via `Fiber.pipe(pipe)` or attached to a Flow via
`Flow.fiber(fiber)`. The `flow()` operation returns an empty identity flow whose input and output
types coincide; operators are chained onto it to build a composition chain (§6.2), which is then
applied to a pipe via pipe composition (§6.2.6). The `flow(fiber)` operation returns an identity
flow with `fiber` already attached — a convenience for transitioning from a Fiber recipe to a Flow
that can be further composed with type-changing operators. Note: `Cortex.state()`
creates an empty State, while `State.state(name, value)` adds a slot and returns a new State. The
reuse of the name `state` across these types is deliberate — the operation semantics are unambiguous
because the receiver types differ. Projections MAY provide additional convenience operations — e.g.,
extra `name()` overloads for classes, enums, or iterables, `state(Slot)` for direct slot insertion,
or `fiber(type)` / `flow(type)` overloads that use a type classifier to drive inference — as
non-normative extensions. The operations listed above define the minimum portable surface.

#### Execution Operations

**Circuit** — sequential execution engine (also Source\<State, Circuit\>, Resource):

```
Circuit.conduit(name?: Name, type?: Type): Conduit<E>
Circuit.pipe(): Pipe<E>
Circuit.pipe(target: Pipe<E>): Pipe<E>
Circuit.pipe(receptor: Receptor<E>): Pipe<E>
Circuit.subscriber(name: Name, callback: (Subject, Registrar<E>)): Subscriber<E>
Circuit.await()
Circuit.close()
```

The `conduit` operation creates a conduit for the specified emission type classifier (§4.5), with an
optional name. The type classifier MAY also be implicit — inferred by the projection from the
call-site context — in the same manner as the `flow` factory (§16.3 Factory Operations). The `pipe`
operations create async dispatch pipes through the circuit's queue; flow processing is applied via
pipe composition (§6.2.6) rather than circuit-level overloads. The no-argument form returns a
circuit-owned pipe whose dispatched emissions are discarded on the circuit thread — work is
scheduled but the resulting job is a no-op — useful as a sink or placeholder where queue
participation is required but no downstream processing is needed. The `subscriber` operation creates
a
callback handle for dynamic pipe discovery; the callback receives the subject of a named pipe and a
registrar for attaching downstream pipes. As a Source, a circuit is parameterized with State —
subscriptions to a circuit observe the circuit's lifecycle state transitions (§7.1.1). Projections
MAY provide additional overloads — e.g., `conduit(name, type, routing)` for hierarchical dispatch (
§10.3), or an explicit type-witness overload `conduit(type)` analogous to `flow(type)` — as
non-normative conveniences.

#### Emission Operations

**Pipe** — emission carrier:

```
Pipe.emit(value: E)
```

The `emit` operation delivers a value through this pipe. Pipe itself exposes no composition surface;
Fibers and Flows are attached to a pipe via composition methods on the composition values
themselves (`Fiber.pipe(pipe)`, `Flow.pipe(pipe)`) — see §6.2.6.

**Receptor** — emission callback:

```
Receptor.receive(value: E)
```

#### Pipeline Operations

**Fiber\<E\>** — same-type per-emission recipe. A fiber carries a single emission type `E`; every
operator preserves that type and returns `Fiber<E>`.

```
Fiber.guard(predicate: (E) → Boolean): Fiber<E>
Fiber.guard(initial: E, predicate: (E, E) → Boolean): Fiber<E>
Fiber.diff(): Fiber<E>
Fiber.diff(initial: E): Fiber<E>
Fiber.change(key: (E) → K?): Fiber<E>
Fiber.limit(count: Long): Fiber<E>
Fiber.skip(count: Long): Fiber<E>
Fiber.takeWhile(predicate: (E) → Boolean): Fiber<E>
Fiber.dropWhile(predicate: (E) → Boolean): Fiber<E>
Fiber.reduce(initial: E?, operator: (E?, E) → E): Fiber<E>
Fiber.relate(initial: E?, operator: (E?, E) → E?): Fiber<E>
Fiber.edge(initial: E, transition: (E, E) → Boolean): Fiber<E>
Fiber.pulse(predicate: (E) → Boolean): Fiber<E>
Fiber.steady(count: Integer): Fiber<E>
Fiber.steady(count: Integer, same: (E, E) → Boolean): Fiber<E>
Fiber.hysteresis(enter: (E) → Boolean, exit: (E) → Boolean): Fiber<E>
Fiber.inhibit(refractory: Integer): Fiber<E>
Fiber.delay(depth: Integer, initial: E): Fiber<E>
Fiber.integrate(initial: E?, accumulator: (E?, E) → E, fire: (E?) → Boolean): Fiber<E>
Fiber.rolling(size: Integer, combiner: (E, E) → E, identity: E): Fiber<E>
Fiber.tumble(size: Integer, combiner: (E, E) → E, identity: E): Fiber<E>
Fiber.every(interval: Integer): Fiber<E>
Fiber.chance(probability: Double): Fiber<E>
Fiber.replace(operator: (E) → E?): Fiber<E>
Fiber.peek(receptor: Receptor<E>): Fiber<E>
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
Fiber.pipe(pipe: Pipe<E>): Pipe<E>
```

An identity fiber is obtained from `Cortex.fiber()` and is the starting point for building a
per-emission recipe. The completed fiber is handed to `Fiber.pipe(pipe)` for direct attachment (
§6.2.6), or to `Flow.fiber(fiber)` / `Cortex.flow(fiber)` to be wrapped in a Flow.

**Flow\<I, O\>** — left-to-right composition surface. A flow carries two type parameters: the input
type `I` (upstream-facing) and the output type `O` (downstream-facing). A same-type flow (`I == O`)
is valid. Flow holds only operators that cross type boundaries or append an output-side stage:

```
Flow.map(fn: (O) → P?): Flow<I, P>
Flow.fiber(fiber: Fiber<O>): Flow<I, O>
Flow.flow(next: Flow<? super O, ? extends P>): Flow<I, P>
Flow.pipe(pipe: Pipe<O>): Pipe<I>
```

`map` appends a type-changing transformation at the output side (an `O → P` function, returning
`Flow<I, P>`); a return of absence drops the emission. `fiber` appends a same-type Fiber recipe at
the output side, returning `Flow<I, O>`; the Fiber's operators run last on each emission, after any
preceding `map` or upstream stages. `Flow.flow` and `Fiber.fiber` compose two recipes of the same
kind end-to-end; the left-hand recipe runs first, its surviving emissions are fed into the
right-hand recipe, and fresh execution state is allocated for each recipe's stateful operators at
every attachment.

An identity flow (`I == O`) is obtained from `Cortex.flow()` and is the starting point for building
a Flow. The completed flow is handed to `flow.pipe(target)` for composition (§6.2.6).

#### Routing Operations

**Pool\<T\>** — composable name-based view:

```
Pool.get(name: Name): T
Pool.pool(fn: (T) → U): Pool<U>
```

The `get` operation returns the cached instance for a name or creates a new one (§10.1). The
`pool(fn)` operation creates a derived pool that caches transformed results per name (§10.1).
Projections MAY provide additional `get` overloads — e.g., `get(Subject)` or `get(Substrate)` — as
non-normative conveniences.

**Conduit\<E\>** — pipe factory and source (also Pool\<Pipe\<E\>\>, Source\<E\>, Resource):

```
Conduit.pool(flow: Flow<T, E>): Pool<Pipe<T>>
Conduit.pool(fiber: Fiber<E>): Pool<Pipe<E>>
Conduit.close()
```

`Conduit` extends `Pool<Pipe<E>>` and inherits `get(name)` and `pool(fn)`. As a Source it inherits
`subscribe`, `reservoir`, `tap`, and `subject`, and as a Resource (via Source) it exposes `close()`
with the semantics of §9.1. The `pool(Flow)` and `pool(Fiber)` overloads are pipe-aware forms of
`pool(fn)` available only on `Conduit` (since `Pool<T>` is generic in `T`, it cannot express the
`Pipe<E> → Pipe<T>` relationship). Both are semantically equivalent to `pool(flow::pipe)` /
`pool(fiber::pipe)` respectively — they prepend the flow or fiber to each channel's pipe, so
emissions of the upstream type flow through the operators and land in the conduit's `Pipe<E>`
after transformation (type-changing for Flow) or filtering/stateful processing (type-preserving
for Fiber).

#### Subscription Operations

**Source\<E, S\>** — subscribable event origin (also Resource):

```
Source.subscribe(subscriber: Subscriber<E>): Subscription
Source.subscribe(subscriber: Subscriber<E>, onClose: (Subscription) → Unit): Subscription
Source.reservoir(): Reservoir<E>
Source.tap(fn: (Pipe<T>) → Pipe<E>): Tap<T>
Source.tap(flow: Flow<E, T>): Tap<T>
Source.tap(fiber: Fiber<E>): Tap<E>
Source.subject(): Subject
```

The first `subscribe` overload registers a subscriber. The second additionally attaches an `onClose`
callback that fires exactly once when the returned subscription is terminated — whether by explicit
close, subscriber close, or source close. The callback receives the subscription being closed and
executes within the circuit context (unless the owning circuit has already terminated, in which case
the projection MAY fall back to synchronous invocation on the caller context — see §15.1 for the
enforcement model).

The `reservoir` and `tap` operations are available on all sources (circuits, conduits, taps). The
first `tap` overload takes a pipe function that receives the tap's target pipe and returns a
source-compatible pipe; this enables type transformation and flow composition to be expressed within
a single function (§11.1). The second overload accepts a `Flow<E, T>` directly — semantically
equivalent to `tap(flow::pipe)` — and is the preferred ergonomic form when the transformation is
already expressed as a flow. The third overload accepts a `Fiber<E>` for the type-preserving case —
semantically equivalent to `tap(fiber::pipe)` — enabling stateful or filtering per-emission
operators (`diff`, `guard`, `limit`, `above`, `below`, etc.) to attach directly without wrapping
in a flow.

**Registrar** — pipe attachment handle (temporal, valid only during subscriber callback):

```
Registrar.register(pipe: Pipe<E>)
Registrar.register(receptor: Receptor<E>)
```

**Subscription** — cancellable subscription handle (also Resource):

```
Subscription.close()
```

**Subscriber\<E\>** — pipe discovery callback (also Resource):

```
// no type-specific operations
```

Subscriber has no operations beyond the universal `subject()` and `close()` contracts defined above.
Its behavioral contract is the callback passed at construction time (
`Circuit.subscriber(name, callback)`): the callback receives a `Subject` and a `Registrar<E>` when a
named pipe is discovered on a subscribed source (§7.2, §7.3). The Subscriber's subject identifies
the handle itself; closing it cancels all subscriptions derived from it and stops further callback
invocations.

#### Derived Structure Operations

**Reservoir** — emission buffer (also Resource):

```
Reservoir.drain(): Sequence<Capture<E>>
```

**Capture\<E\>** — emission-subject pair:

```
Capture.emission(): E
Capture.subject(): Subject<Pipe<E>>
```

A capture pairs an emitted value with the subject of the pipe that produced it. The subject's type
parameter (`Pipe<E>`) identifies the substrate role — the subject belongs to the pipe that emitted
the value, not to the conduit or source that owns the pipe.

**Tap** — transformed source mirror (also Source, Resource):

```
Tap.close()
```

#### Context Operations

**Current** — execution context reference (temporal, execution-context-scoped):

```
// no type-specific operations
```

Current has no operations beyond the universal `subject()` contract. It is obtained via
`Cortex.current()` and is temporal: instances MUST NOT be retained or shared across execution
contexts (§6.4). The `subject()` of a Current returns the identity record of the execution context
that produced it, allowing callers to correlate work with its owning context without exposing
context-specific machinery.

#### Identity Operations

**Subject** — component identity record:

```
Subject.id(): Id
Subject.name(): Name
Subject.state(): State
Subject.type(): Type
```

**Name** — hierarchical interned identifier (also Extent):

```
Name.name(segment: String): Name
Name.path(): String
```

**Extent** — hierarchical containment:

```
Extent.part(): String
Extent.enclosure(): Extent?
Extent.extremity(): Extent
Extent.depth(): Integer
Extent.path(): String
```

#### Data Operations

**State** — immutable slot collection:

```
State.state(name: Name, value: T): State
State.iterate(): Sequence<Slot>
State.value(slot: Slot<T>): T
```

The `state` operation MUST support all types defined in §8.3. Each call returns a new State
instance; the original MUST remain unchanged. Writing a slot with a (name, type) pair that already
exists in the state MUST replace that slot in place (upsert) — the returned state contains the newly
written slot and any prior slot with the same (name, type) is removed. Writing a slot whose (name,
type, value) already matches the existing entry MUST return the same state instance. Because of
upsert semantics, the number of slots in a state is bounded by the number of unique (name, type)
pairs ever written, not by the number of writes. The `iterate` operation returns slots in
most-recently-written-first order, as defined in §8.1. The operation name `iterate` is abstract —
projections use their idiomatic traversal mechanism (e.g., `stream()` in Java, `iter()` in Rust,
`__iter__` in Python). The `value` operation returns the value of the slot matching the given
slot's (name, type) pair, or the template slot's own value when no match exists.

**Slot** — named typed value:

```
Slot.name(): Name
Slot.type(): Type
Slot.value(): T
```

#### Lifecycle Operations

**Resource** — closeable component:

```
Resource.close()
```

**Scope** — hierarchical resource manager (also Extent). A scope's extent semantics: `part()`
returns the scope's name segment, `enclosure()` returns the parent scope's extent (or absent for a
root scope), and `depth()` / `path()` / `extremity()` traverse the scope nesting hierarchy:

```
Scope.register(resource: R): R                  [R: Resource]
Scope.closure(resource: R): Closure<R>           [R: Resource]
Scope.scope(name?: Name): Scope
Scope.close()
```

The `closure` operation creates a single-use, block-scoped resource handle. Closure is temporal — it
is valid only during the scope that created it.

**Closure\<R\>** — block-scoped resource handle (temporal, single-use-scoped):

```
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

| Topic                     | What the spec requires (normative)                                            | What projections may choose freely                                                          |
|---------------------------|-------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| **Type representation**   | Distinct, equality-comparable classifiers for each required type (§4.5)       | Java `Class`, Rust enum, Go string tag, integer constants, etc.                             |
| **Absent value**          | Drop semantics for map/tap mappers; absence violation detection (§1.2, §15.2) | `null`, `Option`, `Maybe`, nil, or similar sum types that distinguish "no value" from error |
| **Identity**              | Canonical identity within implementation boundary (§1.2)                      | Pointer equality, interned handles, content-addressed lookup                                |
| **Error signaling**       | Distinguishable error categories (§15.1)                                      | Exceptions, error returns, Result types, rejected futures                                   |
| **Execution context**     | Sequential, exclusive circuit context (§5.1)                                  | Thread, goroutine, coroutine, actor, event-loop turn                                        |
| **Concurrency mechanism** | Happens-before guarantees (§5.4)                                              | Memory barriers, channels, locks, runtime-specific ordering                                 |
| **Resource cleanup**      | Idempotent close, scope ordering (§9)                                         | `AutoCloseable`, `Drop`, `defer`, `with` statement, destructor                              |
| **Provider loading**      | A cortex instance exists as entry point (§3)                                  | `ServiceLoader`, dependency injection, explicit construction, registry                      |
| **Collection types**      | Ordered, iterable drain result with snapshot semantics (§11.2)                | `Iterable`, `Stream`, `Iterator`, `Vec`, slice, generator                                   |
| **Numeric types**         | Specified precision and range (§8.3)                                          | Native primitives, wrapper types, newtype wrappers                                          |

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

- `ServiceLoader`-based provider discovery
- `AutoCloseable` on `Scope` (not on `Resource` directly) for resource cleanup
- `Class<?>` for type representation (`Subject.type()` returns `Class<S>`)
- `Stream` for drain results and state traversal (`State.stream()` for `iterate()`)
- `CharSequence` for `Extent.path()` return type (spec uses abstract `String`)
- `NullPointerException` for absence violations
- Virtual threads for circuit context execution
- `@NotNull` annotations for absence constraints
- Nested interface style for API organization

**Tenure mapping:**

The Java projection uses a three-valued `@Tenure` annotation (`INTERNED`, `EPHEMERAL`, `ANCHORED`).
The specification's Context-scoped tenure (§13) is mapped to `INTERNED` in Java, since `Current`
instances are effectively interned per execution context (thread). The behavioral restriction — that
Context-scoped instances MUST NOT be retained or shared across contexts — is conveyed through the
separate `@Temporal` annotation.

**Temporal violation detection:**

Registrar violations are detected via sentinel state. Since fibers and flows are standalone builder
values (not callback-scoped), there is no temporal contract on Fiber or Flow and therefore no
detection requirement.

**Non-normative API extensions:**

The Java projection provides additional types and operations beyond the required surface:

- `Receptor.NOOP` — shared no-op receptor constant
- `Receptor.of()`, `Receptor.of(Class)`, `Receptor.of(Class, Receptor)` — typed no-op and identity
  factories for cleaner use with `var`
- `State.value(Slot<T>)` — template-based slot lookup (projection of `value` operation)
- `State.state(Slot<?>)` — direct slot insertion; `State.state(Enum<?>)` — enum convenience
- `Extent.fold()` / `foldTo()` — extent traversal conveniences
- Multiple `Extent.path()` overloads — separator and mapper variants
- Named overloads — `conduit(Name, ...)`, `scope()` (no-arg), `circuit(Name)`
- `Cortex.name(Class)`, `name(Member)`, `name(Enum)`, `name(Iterable)`, `name(Iterator)` — name
  creation conveniences
- `Cortex.fiber(Class<E>)` / `Cortex.flow(Class<E>)` — typed convenience overloads for
  `Cortex.fiber()` / `Cortex.flow()` that drive inference from a class witness
- `Fiber.limit(int)` — native integer width overload (spec defines `Long` only)
- `Pool.get(Subject)` / `Pool.get(Substrate)` — convenience overloads that extract the name from an
  existing substrate
- `Circuit.conduit(Class<E>)` / `Circuit.conduit(Name, Class<E>)` — typed convenience overloads for
  `Circuit.conduit()` / `Circuit.conduit(Name)` that drive inference from a class witness
- `Circuit.conduit(Class<E>, Routing)` / `Circuit.conduit(Name, Class<E>, Routing)` — overloads
  selecting hierarchical dispatch via the `Routing` enum (see §10.3)
- `@Queued`, `@New`, `@Provided`, `@Tenure` — annotations documenting runtime behavior

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

# The Substrates Specification — Design Rationale

**Companion to SPEC.md Version 3.0.5**
**Copyright © 2025–2026 William David Louth / Humainary**

---

This document explains the design decisions behind the Substrates specification. Where the
specification defines *what* a conformant implementation must do, this document explains *why* those
requirements exist. It positions the specification against related work and documents the tradeoffs
that shaped the design.

This document is entirely non-normative. No conformance requirements are defined here.

## 1. Why Determinism Over Throughput

The specification's first governing principle (§2) privileges deterministic ordering over maximum
throughput. This is the foundational tradeoff from which most other design decisions follow.

### The Problem with Throughput-First Designs

Contemporary concurrent frameworks — thread pools, actor runtimes, async task schedulers — optimize
for CPU utilization by dispatching work to any available execution context. This maximizes
throughput for stateless, embarrassingly parallel workloads. However, it makes the processing order
of concurrent messages dependent on opaque runtime variables: OS thread scheduling, garbage
collection pauses, cache coherency delays, and network latency. The result is that two runs of the
same program with the same inputs may produce different intermediate states, different event
orderings, and different observable behaviors.

For stateful systems — where the order in which events are processed determines the final state —
this non-determinism is not a performance characteristic but a correctness problem. Financial ledger
processing, multi-agent coordination, industrial control, and digital twin synchronization all
require that a given sequence of inputs produces the same sequence of outputs, every time.

### The Substrates Position

Substrates resolves this by mandating that every emission within a circuit is processed sequentially
in strict admission order (§5.4.3). Earlier emissions complete before later ones begin. All
observers see the same sequence. This eliminates an entire class of concurrency bugs — not by
managing them with locks and barriers, but by making them structurally impossible.

The cost is that a single circuit cannot exploit parallelism across its emissions. The specification
addresses this through two mechanisms:

1. **Multiple circuits**: Independent circuits execute concurrently on separate execution contexts.
   Parallelism is achieved by partitioning work across circuits, not by parallelizing within one.
2. **Caller-side work shifting**: The specification explicitly advises (§5.2) that expensive
   computation, serialization, and preparation should occur in the caller context before emission.
   The circuit context handles only routing, filtering, and lightweight state updates — minimizing
   the sequential bottleneck.

### Replay as a Structural Consequence

Deterministic ordering makes replay a free architectural property rather than an additional feature.
If the ingress queue is captured as an ordered log, replaying that log into the same circuit
topology reproduces the exact same state transitions (§5.1). This enables:

- **Testing**: Deterministic replay makes concurrent systems testable without flaky timing-dependent
  assertions.
- **Digital twin synchronization**: A remote replica needs only the ordered event stream, not
  continuous state broadcasts, to maintain perfect synchronization.
- **Forensic debugging**: A failure can be reproduced exactly by replaying the input log up to the
  point of failure.

The specification clarifies (§5.1, §7.6) that the replay log must capture all circuit operations —
emissions, subscription registrations, unsubscriptions, and close operations — not just emissions.
Subscription operations affect pipeline topology, which determines how subsequent emissions are
routed.

### Buffered Capture as the Digital Twin Bridge

Buffered capture is the concrete mechanism that makes digital twin synchronization practical. A
`Sink` subscribed to a source mints a `Capture` for every emission — alongside the subject of the
channel that produced it, preserving circuit-context processing order — and a `Basin<Capture<E>>`
(§11.2) retains them. Its incremental, consuming drain forwards only the captures accumulated since
the last drain, with snapshot semantics — the forwarded batch is unaffected by subsequent emissions.

This maps directly to the digital twin pattern: the primary system emits through its normal
topology; a sink attached to the same source captures the ordered emission stream into a basin; the
remote replica replays that stream. Because the drain preserves emission order and the circuit
guarantees deterministic processing, the replica converges to the same state as the primary. The
synchronization payload is the ordered event stream — not continuous state snapshots — reducing
bandwidth requirements to the volume of actual state changes.

## 2. Why Circuit-Context Confinement

The specification mandates that each circuit owns exactly one sequential execution context, and that
all emissions, flow operations, subscriber callbacks, and state transitions execute exclusively
within that context (§5.1). This is the circuit-context confinement guarantee.

### Eliminating Data Races by Construction

A data race occurs when two concurrent operations access the same memory location, at least one is a
write, and there is no happens-before ordering between them. Traditional approaches mitigate this
through manual synchronization — mutexes, read-write locks, atomic variables — which is error-prone,
cognitively expensive, and introduces secondary failure modes (deadlock, livelock, priority
inversion).

Circuit-context confinement eliminates data races structurally: if only one operation can execute at
a time within a circuit, concurrent mutation is impossible. State accessed within the circuit
context requires no synchronization. Developers write sequential logic; the framework guarantees
sequential execution.

### Relationship to the Actor Model

The Actor Model (Hewitt, 1973) shares the principle of isolated state with message-passing
communication. However, traditional actor implementations (Erlang/OTP, Akka) do not guarantee
processing order when multiple actors send messages to the same target concurrently. The mailbox
ordering is determined by arrival time, which depends on scheduling and network latency.

Substrates differs in two ways:

1. **Deterministic ingress ordering**: The ingress queue imposes a total order on external
   submissions. While the relative ordering of concurrent callers is determined by the queue's
   synchronization mechanism (and may vary across runs), each run sees a single, consistent order —
   there is no per-message non-determinism within a run.
2. **Causal completion via transit priority**: Actors process messages independently — message A's
   side effects may interleave with message B's processing. In Substrates, the transit queue ensures
   that all cascading effects of emission A complete atomically before emission B begins (§5.3).

### Virtual Threads and the Sequential Bottleneck

The Java projection implements the circuit context as a virtual thread — a lightweight execution
context managed by the runtime rather than the OS. This is a projection decision, not a
specification requirement (§5.1 defines the circuit context abstractly). Virtual threads are
well-suited because:

- They are cheap to create (one per circuit is practical even with thousands of circuits).
- They do not consume OS thread resources while parked.
- The sequential execution guarantee means the virtual thread never needs to coordinate with other
  threads for internal state access.

The specification's performance target — sub-3ns emission latency at 336M ops/sec — demonstrates
that the sequential bottleneck is manageable when the circuit context is kept lightweight and work
is shifted to caller threads.

## 3. Why the Ingress / Transit Ordering Model

The ingress / transit ordering model (§5.3) separates external submissions (ingress) from internal
cascading emissions (transit), with transit taking strict priority. The specification defines this
as a logical ordering rule, not a particular implementation data structure — implementations may
realize it via separate queues, a single mailbox with priority markers, actor turns, or any other
mechanism. This design solves three problems simultaneously.

### Problem 1: Causal Fragmentation

In a single-priority model, an external emission E1 that triggers internal emissions I1 and I2 would
interleave with subsequent external emission E2:

```text
Single ordering: E1, E2, I1, I2  (causal chain of E1 is fragmented)
```

Transit priority ensures causal completion:

```text
Ingress/transit: E1, I1, I2, E2  (E1's effects complete atomically)
```

This means external observers see atomic state transitions. E2 observes the fully resolved state of
E1, never a partial intermediate state.

### Problem 2: Stack Overflow in Recursive Topologies

In naive event-driven architectures, cascading emissions are handled via immediate callback
execution — recursive function calls on the stack. In cyclic topologies (feedback loops, recurrent
networks), this produces unbounded recursion and stack overflow.

Transit ordering converts recursion to iteration. Cascading emissions are admitted as transit work,
not immediately invoked on the call stack. Processing remains iterative regardless of cascade depth.
This is essential for the specification's target use case of neural-like computational networks,
where cyclic signal propagation is the norm, not the exception.

### Problem 3: Coordination Without Locks

The ingress/transit split enables lock-free operation on the hot path. Transit work is produced and
consumed only by the circuit thread — no synchronization required. The ingress admission path
requires synchronization only at the admission boundary. Once work enters the circuit context, all
processing (including cascading) is lock-free.

### Relationship to Berkeley Reactor / Lingua Franca

The Berkeley Reactor model (Lohstroh & Lee, 2019) also seeks to tame actor-model non-determinism
through formal causal orderings and synchronous-reactive principles. Both Reactors and Substrates
use controlled event schedulers to achieve determinism.

The key difference is structural. Reactors focus on the scheduling of logical time — reactions are
ordered by a global logical clock. Substrates introduces physical routing boundaries (Cortex →
Circuit → Conduit → Pipe → Receptor) that make the topology itself an architectural artifact. The
routing path of a signal is as observable and constrained as its timing. This containment hierarchy
enables hierarchical resource management (§9), scoped lifecycle control, and stable topological
reasoning that pure scheduling models do not provide.

## 4. Why Unbounded Queues

The specification mandates (§5.6) that emit must not block the caller and must not discard emissions
while the circuit is open. This implies an effectively unbounded ingress queue.

### Why Not Backpressure?

Reactive Streams (reactive-streams.org) addresses the producer-consumer speed mismatch through
demand-based backpressure: consumers signal how many elements they can accept via `request(n)`, and
producers must not exceed that demand. This is appropriate for data pipeline workloads where
throughput adaptation is the goal.

Substrates rejects queue-level backpressure for two reasons:

1. **Blocking breaks determinism**: If a caller blocks because the queue is full, the order in which
   callers resume depends on the runtime scheduler — introducing non-determinism into the ingress
   sequence.
2. **Dropping breaks replay**: If emissions are discarded under load, the replay log is incomplete
   and the deterministic replay guarantee fails.

### Pipeline-Level Admission Control

The specification provides admission control within the circuit context, after dequeue, where it
does not affect ingress ordering:

- **Fiber.limit ()** (§6.2.3): Caps the number of emissions processed by a pipeline stage. After the
  limit, the pipeline is permanently blocked at that stage.
- **Fiber.every () / Fiber.chance ()** (§6.2.3): Filter emissions by interval — either count-based
  (every Nth emission) or processing-time-based (every elapsed `Duration`) — or by probability
  respectively. Reduce downstream volume without affecting ingress.

These operators are deterministic — given the same input sequence, they produce the same output
sequence. They are the specified mechanism for controlling emission volume.

Beyond admission control, the Flow pipeline enables deterministic complex event processing (CEP)
within the circuit context. Comparison operators (§6.2.4) provide filtering with running state:
`high()` and `low()` maintain watermarks, passing only values that exceed all previous maximums or
minimums. Combined with `range()` and `deadband()` for inclusive and exclusive-band filtering,
`clamp()` for signal saturation, `guard()` for predicate logic, `diff()` and `change()` for value-
and key-based change detection, `relate()` for derivative-style lookback (deltas, moving averages),
and `integrate()` for accumulate-and-fire batching, these operators compose into CEP patterns
without requiring external stream processing engines. Transition detection is provided by `edge()`
and `pulse()` for arbitrary-predicate and rising-edge transitions; temporal confirmation by
`steady()` for stable-value runs and `streak()` for predicate-match runs. Threshold gating combines
these primitives with admission control: `streak(N, p)` fires on the Nth consecutive match of a
predicate, while `guard(p).every(N)` fires on every Nth cumulative match — the consecutive and
cumulative variants of the same threshold pattern, expressed without introducing a parallel
threshold abstraction. Gating and rate control are provided by `hysteresis()` for two-threshold
regime latching and `inhibit()` for refractory periods after event detection. Temporal displacement
(`delay()`), windowed aggregation (`rolling()` for sliding windows, `tumble()` for non-overlapping
batches), and Flow `window(count)` / `window(duration, capacity)` for callback-scoped rolling
inspection — count-bounded and time-bounded respectively — round out the vocabulary for threshold
detection, trend analysis, anomaly filtering, and spiking-neuron models. Because all pipeline state
is confined to the circuit context, it requires no synchronization and executes at raw computation
speed.

## 5. Why Lazy Rebuild Subscriptions

The subscription model (§7) uses version-tracked lazy rebuild rather than eager synchronization.
This is a performance-critical design choice.

### The Alternative: Eager Rebuild

An eager model would rebuild every channel's pipe list immediately when a subscription changes. In a
conduit with thousands of channels, a single subscribe/unsubscribe operation would trigger thousands
of rebuilds — most for channels that may never emit again. This creates a latency spike proportional
to the number of channels.

### Version-Tracked Lazy Rebuild

Instead, subscription changes increment a version counter. Each channel (inlet) caches its own
version. On emission, the channel compares its cached version to the hub's current version. If they
differ, it rebuilds its pipe list. If they match, it proceeds directly — O (1) per emission.

This means:

- **Subscription cost is O (1)**: Register the subscription and increment the version counter. No
  per-channel work.
- **Rebuild cost is amortized**: Each channel rebuilds at most once per subscription change, and
  only when it actually emits.
- **Idle channels never rebuild**: Channels that never emit after a subscription change incur zero
  cost.

The tradeoff is eventual consistency (§7.6): subscription changes are not immediately visible to all
channels. Each channel discovers the change on its next emission. For the specification's
deterministic model, this is acceptable because the rebuild happens within the circuit context — it
is causally ordered with respect to the emission that triggers it.

## 6. Why Temporal Contracts

The specification defines temporal contracts (§6.4) for callback-scoped handles such as Registrar
and Window. Registrar is valid only during the subscriber callback and must not be retained. Window
is valid only during the receptor or Fiber callback that observes it; callers retain copied values,
not the Window reference. (In 1.0, Flow also carried a temporal contract tied to a configurer
callback. In 2.0, flows became standalone builder values obtained from `Cortex.flow()` and applied
via `Flow.pipe(pipe)`, so no temporal contract applies to Flow.)

### The Design Intent

Temporal contracts serve three purposes:

1. **Object reuse**: Because the spec prohibits retention, implementations may reuse callback-scoped
   objects across callbacks. This eliminates allocation in the emission hot path — critical for
   achieving sub-3ns emission latency.
2. **Lifecycle clarity**: The temporal contract makes the ownership model explicit. A Registrar
   belongs to the subscriber callback; a Window belongs to the callback that observes it. Retaining
   either would create a dangling reference to internal framework state.
3. **Enforcement flexibility**: The specification (§6.4.1) allows implementations to choose between
   detection (throwing on misuse) and undefined behavior, depending on whether detection imposes
   overhead on the valid path. The Java projection detects Registrar violations via sentinel state.

### Comparison to Rust's Borrow Checker

The temporal contract pattern is conceptually similar to Rust's lifetime system, where references
are valid only within a defined scope. The difference is enforcement: Rust verifies lifetimes at
compile time; Substrates defines them in the specification and relies on projection-specific
enforcement. In languages without compile-time lifetime tracking (Java, Python, Go), runtime
detection or documentation must substitute.

## 7. Why Containment Hierarchy

The specification organizes components in a strict containment hierarchy: Cortex → Circuit → Conduit
(§3). This is not merely organizational — it has structural consequences.

### Hierarchical Resource Management

Closing a circuit closes its conduits and the pipes they pool (§9.3). Closing a scope closes all
registered resources in reverse order (§9.2). This hierarchical cleanup prevents resource leaks and
ensures that the teardown sequence mirrors the construction sequence.

### Scoped Determinism

Each circuit is an independent deterministic domain. Cross-circuit communication goes through emit
(admission to the target circuit's ingress path), which preserves the deterministic guarantee within
each circuit while allowing concurrent execution across circuits. The containment hierarchy makes
this boundary explicit — a developer can reason about determinism at the circuit level without
considering the entire system.

### Stable Routing

Pipe pooling within conduits (§12) guarantees that the same name always resolves to the same pipe.
This means routing decisions are stable — once a name-to-pipe mapping exists, it persists for the
conduit's lifetime. Subscribers can rely on this stability for dynamic topology construction without
worrying about name resolution races.

### Why Name Depth Has a Floor, Not a Ceiling

Names are interned and compared by canonical identity (§4.1), so implementations encode name
identity compactly — the reference implementation packs the depth and a counter into a single
integer. Depth is therefore a bounded resource in practice, and a specification that said nothing
about it would leave portable code depending on whatever bound one provider happened to pick.

The specification guarantees a floor rather than mandating unboundedness. Sixteen segments sits well
beyond what namespace-shaped identifiers need: a fully qualified member name such as
`io.humainary.substrates.api.Substrates.Extent.path` is seven, and realistic instrument names run to
three or four. Sixteen leaves roughly a factor of two of headroom over the deepest plausible use
while staying cheap enough that a constrained implementation can meet it without a compact encoding
becoming the binding constraint. An implementation willing to pay for more may offer more — the
reference implementation allows 32 — but portable code may not assume it.

## 8. Terminology

The specification uses terminology inspired by neuroscience and physical systems: cortex, circuit,
conduit, pipe, receptor. This is deliberate but should not be over-interpreted.

### Why These Names

The names were chosen for conceptual clarity, not as claims about biological simulation:

- **Cortex**: The outermost boundary — a factory and root context. Named for the cerebral cortex as
  the outermost layer of neural organization.
- **Circuit**: A self-contained processing unit with internal state and deterministic execution.
  Named for neural circuits — localized networks with specific computational functions.
- **Conduit**: A structured pathway that organizes and routes signals to named destinations. Named
  for physical conduits — channels that guide flow.
- **Pipe**: A named, typed carrier for emissions. Pipes are pooled by name within a conduit; each
  named pipe is a stable emission point for the lifetime of its conduit.
- **Receptor**: A callback that responds to received signals. Named for synaptic receptors —
  endpoints that trigger responses to incoming signals.

### What the Names Do Not Imply

The specification does not claim to model biological neural networks. The terminology provides an
intuitive mapping for understanding signal flow through a governed topology. The specification's
actual target use cases — observability, digital twin synchronization, multi-agent coordination —
are software engineering problems, not neuroscience problems.

That said, the design intent documented in the project (but not in the specification) includes
exploration of neural-like computational networks: spiking network implementations, recurrent
topologies with feedback loops, and hierarchical organization with different timescales. The
specification's architectural properties — deterministic ordering, cyclic safety, dynamic topology
via subscriptions, sub-3ns emission latency — were shaped by these aspirations. The terminology
reflects this lineage without binding the specification to it.

---

*This document accompanies the Substrates Specification (SPEC.md) and explains the reasoning behind
its design decisions. It is non-normative and carries no conformance requirements.*

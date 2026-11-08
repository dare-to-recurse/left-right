# Wait-free feasibility assessment

Assessment of `left-right` 0.11.8 at commit `9126fc39045f11a009424ffb39de120cc3d44c7a`, dated 2026-11-08. This assessment uses all three papers in ./research_papers/ and the implementation in this checkout. Proposed designs below are engineering deductions, not algorithms proved by the papers or implemented in this investigation.

**The read coordination is already wait-free; publication is blocking. The papers do not provide a direct conversion of this implementation into a fully wait-free primitive.** A stronger publication guarantee is plausible through a substantial redesign, but it requires changing how snapshots are retained and reclaimed, constraining operations and resources, and reconsidering parts of the API. Adding helping, replacing the operation log with a wait-free queue, or removing the registry mutex would not by itself solve the problem.

The central limitation is the combination of two reusable copies and caller-controlled borrowed references. A reader can retain a `ReadGuard` indefinitely. The writer cannot safely reuse that reader's copy, and another thread cannot finish the caller's arbitrary computation or revoke its references. The papers' helping techniques complete recorded data-structure operations; they cannot make such a borrow disappear.

My recommendation is to preserve the current design when the requirement is bounded read coordination. If successful publications must also complete independently of readers, develop a separate snapshot implementation with an explicit resource contract, or replace the generic wrapper with a specialized wait-free data structure. Treat a more predictable `try_publish` as a smaller, useful improvement with a different guarantee.

**The guarantee must be stated per operation.** Here, wait-free means that an operation completes in a bounded number of its own execution steps, independently of other threads' speeds, delays, or suspension. Lock-free guarantees system-wide progress, but permits an individual operation to starve. Neither guarantee alone supplies a wall-clock deadline: scheduling, page faults, allocation, and the implementation of machine atomics remain relevant. These distinctions follow the progress discussions in [MS96], [KP11], and [TP14].

A bound may depend on the number of participants, the input or snapshot size, and the number of operations in a batch. Wait-free does not mean constant time. For a concrete implementation, those dependencies must be identified; arbitrary user callbacks cannot be assumed to terminate merely because the surrounding synchronization is bounded. The read-side classification below assumes bounded primitive atomic operations on the supported target and valid handle/counter usage.

| Operation or component | Guarantee supported by this implementation | Evidence and qualification |
| --- | --- | --- |
| `ReadHandle::enter()` | Wait-free coordination, O(1) | The outer acquisition increments the epoch, executes a fence, loads the pointer, and constructs a guard. The nested path also has no retry loop or mutex. See [read acquisition][read-enter]. |
| `ReadGuard` dereference and release | Wait-free coordination, O(1) | Dereference returns the stored reference. Release decrements a local nesting count and, for the last guard, increments the epoch. The caller's computation through `&T` is outside this bound. See [guard implementation][guard]. |
| `ReadGuard::map` / `try_map` | Bounded guard bookkeeping only | Their supplied closures can do arbitrary work. These methods do not establish a bound for the closure. See [guard mapping][guard-map]. |
| `ReadHandle::clone()`, factory `handle()`, and handle destruction | Blocking lifecycle operations | Creating and removing registered readers locks the epoch slab; creation also allocates. This is distinct from acquiring/releasing a guard on an existing handle. See [registration][registration], [handle destruction][read-drop], and [factory][factory]. |
| `append()` / `extend()` | No general wait-free guarantee | Before the first publication, they execute `absorb_second` directly. Afterwards they append to a local `VecDeque`, which can allocate. An arbitrary iterator passed to `extend` need not terminate. See [append][append] and [extend][extend]. |
| `publish()` and a `flush()` that publishes | Blocking | They acquire the epoch mutex and can wait indefinitely for an unchanged reader epoch. Replaying operations also invokes arbitrary `Absorb` methods. See [publication][publish], [waiting][wait], and [flush][flush]. |
| `try_publish()` | One eligibility scan without retry, but not a wait-free method | It still calls `epochs.lock().unwrap()`, can resize storage, and on success executes callbacks and snapshots epochs again. Its documentation's claim that it never waits is therefore too broad. See [publication attempt][try-publish]. |
| Writer `take()` and destruction | Blocking | They can publish pending work, disconnect the root, wait for readers, and invoke destruction hooks. See [take and destruction][take-inner]. |

Consequently, calling the whole public API lock-free would be misleading under the modern progress definition. Wait-free reads can coexist with a blocking writer, but they do not make the writer lock-free. In particular, a running writer can be the only thread attempting an operation and still make no progress because a suspended reader retains a guard. A thread suspended while holding the registry mutex supplies another blocking execution.

**The implementation waits before reusing the previous read copy.** This detail matters when constructing the failure case. `publish()` waits for the epochs saved after the preceding swap, updates the now-unreferenced write copy, swaps it into the published root, and records another epoch snapshot. The first publication skips the epoch wait. It does not wait for all readers of the newly retired copy before returning. See [publication][publish] and [update and swap][update-swap].

The following execution requires no contention between writers:

1. The root points to copy A. A reader enters, receives a reference into A, and leaves its epoch odd.
2. The writer prepares B and publishes it. The saved epoch for that reader is odd. The reader still legitimately holds A.
3. The writer appends another change and calls `publish()`. It now needs to modify A.
4. The reader remains suspended, or simply retains its guard. In `wait()`, the saved epoch is odd and the current epoch remains equal to it. The writer repeats the loop forever.

Yielding after the initial spins does not change this dependency. There is no failed CAS whose failure demonstrates somebody else's progress. There need not be any concurrent modification at all.

This is also a storage limitation, not merely an unfortunate waiting-loop implementation. A second guard can retain B while the first retains A. An update requiring a third distinct state then has no writable copy. For arbitrary `T`, the library cannot overwrite either borrowed object, move the caller's existing references, or expose a partially updated value. **Successful wait-free publication is incompatible with retaining this fixed two-copy reuse scheme and unrestricted snapshot lifetimes.** This is a limitation of that combination, not an impossibility theorem about all snapshot implementations.

I confirmed the concrete execution with a temporary Rust program using this checkout. It retained A with value 0, published B with value 1, acquired B through a nested `enter()`, and queued an increment. All 10,000 calls to `try_publish()` returned `false`; the next attempt succeeded after both guards were released, producing value 2. The finite experiment illustrates the dependency. The unchanged-epoch branch in `wait()` establishes why an indefinitely retained guard permits indefinite waiting.

**The three papers contribute different techniques.** They should not be treated as three interchangeable ways to upgrade this crate.

[MS96], Michael and Scott's *Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms*, gives the lock-free queue used as the starting point by [KP11]. Its key mechanism is recoverable partial progress: once a node is linked, another participant can advance a lagging tail. CAS failures and failed consistency checks can be charged to changes made by other operations. This supports system-wide progress, but does not prevent one operation from repeatedly losing. The paper also presents a separate two-lock queue; that algorithm is blocking. See Section 2, Figure 1, and Section 3.3, PDF pp. 3-6, printed pp. 269-272.

For this crate, the relevant lesson is to make incomplete shared updates discoverable and completable. The current `wait()` has no analogous pending update to complete: it waits for external use of an object to end. Also, the existing `VecDeque<O>` belongs to the single writer. Replacing it with the Michael-Scott queue adds a concurrent queue where one is not currently needed, and still does not provide wait-freedom.

The [authors' maintained pseudocode](https://www.cs.rochester.edu/research/synchronization/pseudocode/queues.html) clarifies that the original memory-management scheme assumes a type-preserving allocator that does not return memory to the operating system. Its reclamation assumptions must not be translated into unrestricted Rust deallocation. This is an additional primary-source check, separate from the supplied PDF.

[KP11], Kogan and Petrank's *Wait-Free Queues With Multiple Enqueuers and Dequeuers*, adds per-thread operation descriptors, phase-based priority, and helping. A participant announces its pending operation and helps operations with phase numbers no greater than its own. The protocol bounds how many competing operations can precede a pending operation. Its queue-specific state transitions ensure that helpers can linearize an operation, record its completion, and finish the structural changes without applying it twice. See Sections 3.1-3.2 and 5.3, PDF pp. 2-7 and 11.

The transferable ideas are explicit requests, a bounded participant set, protection against indefinite overtaking, and results that remain discoverable if the executing helper stops. A pending `ReadGuard` does not supply the equivalent of an enqueue/dequeue descriptor. It grants arbitrary future access to `T`, potentially through references into its interior. Setting its epoch to even from another thread would invalidate that access. Copying its object elsewhere would not redirect existing references.

Using [KP11]'s queue to accept updates could make **submission** wait-free under the queue's assumptions. It would not make updates visible if a sole consumer subsequently blocks in this crate's `publish()`. Waiting for a publication acknowledgement restores that same blocking dependency. Queueing and completing an update are separate guarantees.

[TP14], Timnat and Petrank's *A Practical Wait-Free Simulation for Lock-Free Data Structures*, is the closest match to the requested transformation. It accepts an already linearizable, lock-free algorithm in a specific normalized representation. Each operation has a CAS generator, a CAS executor, and a wrap-up stage. Generator and wrap-up must satisfy the paper's definition of parallelizable; they may perform suitable auxiliary CAS operations, so this requirement is stronger and more precise than merely being deterministic. Owner CAS operations use versioning. See Definition 3.4 and Definition 4.1, PDF pp. 4-6, printed pp. 360-362.

The simulation initially executes a fast path. A contention-failure counter detects lack of progress and triggers a slow path using a wait-free help queue. Helpers agree on operation records and execute critical CAS operations with version information and a modified bit, so success can be discovered even if the thread performing the CAS stops before recording it. Fast-path operations participate in helping too; the paths cannot be designed independently. See Section 5, PDF pp. 6-10, printed pp. 362-366.

| Requirement of the [TP14] transformation | Why the current implementation does not satisfy it |
| --- | --- |
| An already lock-free operation | `publish()` waits for reader quiescence and a mutex. The paper's progress proof depends on the input algorithm being lock-free. |
| Shared updates represented by CAS operations | The root uses an atomic swap, while replay uses ordinary exclusive mutation of `T` and mutable/consumed `O` values. Replacing just the swap with a CAS does not describe or make the other work helpable. |
| Parallelizable generator and wrap-up | `Absorb` callbacks are arbitrary Rust code with `&mut T`; they are not safe to execute concurrently on the same copy. Determinism does not imply duplicate execution is harmless. |
| A valid contention-failure counter | The reader wait can run forever without another thread modifying anything. Definition 3.5 requires the counter to be bounded by concurrent modifications and to increase periodically during an indefinitely running method. Counting unchanged epoch polls cannot satisfy both requirements. |
| A transferable computation record | The operation log records updates, but not the state of a caller's computation through a borrowed reference, nor a resumable arbitrary `Absorb` invocation. |
| Exactly-once effects and safe versioned reuse | The current ownership protocol concerns two copies and two absorptions. It has no descriptor protocol for helpers, additional snapshot generations, or abandoned helper state. |

This rules out a mechanical application of [TP14] to `publish()`. Its Section 7 proof first reduces executions to executions of the original lock-free algorithm; that premise fails here. Enqueuing a blocked publication for help would leave every helper waiting for the same reader.

`Absorb` creates a further compatibility problem. `absorb_first` may modify an operation, `absorb_second` consumes it, and the two destruction hooks support values shared between exactly two copies. The trait does not require cloneable operations, side-effect-free callbacks, bounded execution, or the thread-safety bounds needed for arbitrary helpers. See [the trait contract][absorb]. The [aliased deque example][alias-example] actually destroys removed values during second absorption. With a third retained snapshot, that destruction could occur while an older guard still references the value. More copies therefore require a new ownership contract, not just a larger array of `T`.

The fast-path/slow-path idea remains useful after these prerequisites are solved. For example, a new snapshot-acquisition protocol could try a cheap path a bounded number of times and then announce a helpable acquisition request. However, limiting retries and falling back to the existing blocking `publish()` would not produce a wait-free algorithm.

**There are three realistic directions, with different outcomes.**

| Direction | What it could deliver | Main cost or limit |
| --- | --- | --- |
| Tighten the existing publication-attempt API | An attempt that promptly reports failure when publication cannot proceed | No guarantee that a queued update ever becomes visible while a reader retains its guard |
| Introduce a bounded snapshot implementation with more versions | Potentially wait-free acquisition and successful single-writer publication | New protection/reclamation protocol, more memory, restrictions on participants, snapshots, and update code |
| Use a specialized wait-free object or a helpable operation interface | Wait-free completed operations using [KP11] directly for a queue, or [TP14] for a suitable normalized lock-free structure | Substantial API and representation changes; arbitrary borrowed access to generic `T` no longer supplies the operation interface |

For the first direction, replace the blocking registry acquisition in `try_publish()` with an attempt that can return failure, or move to fixed registration outside the critical path. Preallocate epoch and log storage as appropriate, and specify bounds for successful replay. This would correct the immediate mismatch between the method's description and its mutex use. A standard `try_lock` alone does not establish the rest of a formal wait-free bound: allocation and callbacks still matter. Even a properly bounded failure-returning method is a different contract from `publish()`, which promises to expose the queued changes. Wrapping it in a retry loop restores unbounded waiting.

For successful publication, **the most natural design to investigate keeps the single writer but stops requiring it to recycle a particular reader-held copy.** The writer builds a new version in exclusively owned storage and publishes its root atomically; guards keep older versions alive. There is no writer-versus-writer CAS starvation to solve in this design. The main challenges are bounded snapshot acquisition, finding safe storage, and reclamation.

A credible prototype would need the following explicit contract and mechanisms:

1. **A bound on simultaneously protected snapshots.** Fix a capacity R and register participants before the guaranteed operation phase, or provide a separately proved bounded registration mechanism. Bound guards as well as handles. In the current implementation nested `enter()` reloads the root and can observe a newer version, as the probe demonstrated. With unrestricted publication, permitting every nested guard to retain a different version could retain arbitrarily many versions even on one thread. Requiring nested guards to share one snapshot, or limiting simultaneous snapshot reservations, changes that behavior and must be explicit.
2. **Enough independent storage.** Retain pinned versions and reuse only unpinned versions. With at most R pinned versions, one current version, and one construction version, R+2 copies is a useful storage-counting intuition. It is not a proved pool size for an actual algorithm: in-flight acquisitions, helper reservations, and retirement must also be included. Derive the bound from the chosen protocol. Whole-copy storage can cost O(R times snapshot size); persistent structures may reduce copying but need their own ownership and reclamation argument.
3. **Bounded acquisition with help if needed.** A reader must announce/protect a version before it can be reused. A load-pointer, announce-pointer, recheck-pointer loop alone can retry forever under repeated publication. A slow path would need a shared acquisition request that the reader itself or writer can complete, with an immutable, safely reserved result. It must still finish if the writer stops. The papers motivate this descriptor technique but do not supply a ready-made snapshot protocol for this API.
4. **Bounded construction and cleanup.** Replace the two-copy `Absorb` contract with bounded copying/applying operations and ownership valid for any retained generation. Preallocate or use a separately justified allocator for the guaranteed path. Bound batch size and state size where a fixed operational bound is required. A suspended guard must retain only its allowed storage, not prevent recycling all later retired versions.
5. **A defined publication boundary.** Preserve buffered `append` semantics if desired: prepare the entire batch before installing the new root. Prove that each acquisition returns a coherent snapshot valid at some point during its call, that completed publications are visible to subsequent acquisitions, and that older guards remain valid. Publishing only an operation descriptor is insufficient if readers then need unbounded replay or assistance to obtain their snapshot.
6. **A separate shutdown and ownership-transfer contract.** A fast logical close can disconnect future acquisitions while existing snapshots remain owned. Immediate destruction of all referenced storage cannot be promised. `take()` currently transfers an actual backing object after readers depart; preserving that transfer while outstanding references survive requires waiting. Alternatives include a failure-returning `try_take`, a separately bounded independent copy, or a quiescence-required shutdown phase. Those are API/semantic changes.

For such a prototype, a publication bound should have an explicit shape, for example:

```text
steps_publish <= C_reserve(R)
               + C_copy(S)
               + B * C_apply(S)
               + C_install
               + C_retire(R, S)
```

Here S bounds relevant state size and B bounds the captured batch. This is a target proof decomposition, not a bound established for an existing algorithm. Every term must have a bound independent of another thread taking steps. In particular, `C_reserve` cannot conceal waiting for a guard to drop, and `C_retire` cannot merely move unbounded work into a background worker on which future allocation depends. A size-dependent bound can still establish wait-freedom; a useful real-time limit needs concrete resource and execution bounds as well.

For the specialized-operation direction, the paper techniques become more directly applicable. Replace open-ended borrowed access with owned, describable operations and bounded results. Establish a lock-free representation, identify owner versus auxiliary CAS operations, provide parallelizable preparation/wrap-up, and then apply [TP14]'s help queue and versioned execution protocol. If the desired object is a FIFO queue, [KP11] already supplies the queue-specific construction. For arbitrary sequential `T`, a copy-and-publish universal construction is another conceptual route, but it entails copying, replay restrictions, and a separate storage proof; [KP11] Section 2 and [TP14] Section 4.1 discuss its practicality limitations.

Simply replacing `enter()` with a closure is insufficient. A closure that executes on its caller's thread can still suspend while holding a borrow. To benefit from helping, the operation must have transferable inputs, bounded execution, and a result other threads can safely record. If helpers duplicate preparation on private copies, any external effects, losing-copy destruction, and operation ownership must also tolerate that duplication. Rust's `Send` and `Sync` bounds alone do not supply these semantic properties.

**Memory reclamation is part of the progress argument.** [KP11] presents Java code, discusses phase-counter wraparound in Section 3.3, and proposes a hazard-pointer adaptation in Section 3.4, while leaving its exact integration details outside the paper. [TP14] Section 8 says it does not specifically address general memory management and discusses reclaiming added descriptors and help-queue nodes using hazard pointers. These passages do not establish a complete Rust allocation/reclamation implementation for this crate.

In particular:

- Retaining every old version forever avoids unsafe reuse but consumes unbounded memory. Ordinary epoch reclamation can similarly accumulate retired versions behind a suspended participant. Neither is a satisfactory bounded-memory publication solution.
- Hazard pointers can protect particular versions, but the entire protect/validate/retire protocol must be proved bounded. The name of the reclamation technique is not enough; a repeated validation loop can starve.
- An `Arc` can express snapshot ownership, but safely acquiring an owned reference from a concurrently replaced root is itself a synchronization problem. Loading a raw pointer and incrementing its reference count after reclamation is unsafe. Final `Arc` destruction may also execute arbitrary `T::drop` work.
- Extra generation counters need a wraparound/reuse argument, and the required combined pointer/version operations must be supported with the claimed progress on the target. [KP11]'s practical treatment of a 64-bit phase counter is not a mathematical proof for unlimited execution.
- A third buffer postpones exhaustion; it does not solve unrestricted retained snapshots. Conversely, indefinite retention by one participant does not inherently require leaking every later version: a properly bounded per-version protection design can isolate what that participant pins.

This also limits what can be inferred from the papers' performance measurements. [TP14]'s small measured overhead applies to its tested Java structures and workloads. It does not predict the cost of cloning an arbitrary Rust `T`, changing this crate's highly optimized read path, or adding full Rust memory management. Benchmark read acquisition/release, successful publication, retained-snapshot memory, and tail latency separately. Compare against this checkout under both brief and deliberately stalled readers; throughput alone cannot establish a progress guarantee.

**A proof and validation plan should precede a wait-free claim.** First define the guaranteed operation set: steady-state acquisition/release and publication, or also registration, allocation, destruction, and `take`. State the capacity and callback assumptions, then prove snapshot lifetime safety, atomic batch visibility, and a step bound for every retry or helping loop. For a helper scheme, include the case where a helper stops after making a change but before publishing its completion status.

Adversarial tests should cover readers stopped before and after announcing protection, a guard held across arbitrarily many publications, all allowed snapshot reservations occupied, nested acquisitions, descriptor reuse, and shutdown with outstanding guards. Explicitly instrument protocol steps and check the derived bound; a timeout test is only a diagnostic. No finite set of successful schedules proves wait-freedom.

The repository has a Loom test, but [its synchronization shim][loom-sync] currently translates requested fences to acquire fences, including requested sequentially consistent fences. A redesigned ordering protocol must first validate or update that model. Loom can explore safety interleavings; a small bounded exploration is not a replacement for the progress proof. Miri and sanitizer checks would additionally be appropriate for a new unsafe ownership implementation.

For this assessment, the locked offline build passed, and the existing `counter` and `deque` integration tests passed (two tests total):

```sh
cargo build --locked --offline --target-dir /tmp/left-right-wait-free-target
cargo test --locked --offline --target-dir /tmp/left-right-wait-free-target --test counter --test deque
```

The retained-guard probe also passed with `crossbeam-utils` 0.8.22 and `slab` 0.4.12, matching the repository lockfile. Its essential execution was:

```rust
use left_right::Absorb;

struct Add(i32);
impl Absorb<Add> for i32 {
    fn absorb_first(&mut self, op: &mut Add, _: &Self) {
        *self += op.0;
    }
    fn sync_with(&mut self, first: &Self) {
        *self = *first;
    }
}

fn main() {
    let (mut w, r) = left_right::new::<i32, Add>();
    let old = r.enter().unwrap();
    w.append(Add(1));
    w.publish(); // First swap can proceed while old retains A.
    let current = r.enter().unwrap(); // Nested acquisition observes B.
    assert_eq!((*old, *current), (0, 1));

    w.append(Add(1));
    for _ in 0..10_000 {
        assert!(!w.try_publish());
    }
    assert!(w.has_pending_operations());
    drop(current);
    drop(old);
    assert!(w.try_publish());
    assert_eq!(*r.enter().unwrap(), 2);
}
```

No library implementation was changed. The experimental snapshot directions remain unimplemented and unproved; the negative conclusion about the existing wait is supported directly by its control flow, not by extrapolating the probe's iteration count.

The source references below use PDF page numbers counted from the first page; printed page numbers are supplied where useful.

- **[MS96]** Maged M. Michael and Michael L. Scott, *Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms*, PODC 1996, pp. 267-275. Supplied file: `research_papers/248052.248106.pdf`. Main evidence: Section 2, Figure 1, and Section 3.3 (PDF pp. 3-6). [Author-hosted copy](https://cs.rochester.edu/u/scott/papers/1996_PODC_queues.pdf).
- **[KP11]** Alex Kogan and Erez Petrank, *Wait-Free Queues With Multiple Enqueuers and Dequeuers*, PPoPP 2011. Supplied file: `research_papers/wfquque-ppopp.pdf`. Main evidence: Sections 3.1-3.4 (PDF pp. 2-8) and Section 5.3 (PDF p. 11). [Author-hosted copy](https://csaws.cs.technion.ac.il/~erez/Papers/wfquque-ppopp.pdf).
- **[TP14]** Shahar Timnat and Erez Petrank, *A Practical Wait-Free Simulation for Lock-Free Data Structures*, PPoPP 2014, pp. 357-368. Supplied file: `research_papers/2692916.2555261.pdf`. Main evidence: Definitions 3.4-3.5 and 4.1 (PDF pp. 4-6), Section 5 (PDF pp. 6-10), and Sections 7-8 (PDF pp. 10-11). [Author-hosted copy](https://csaws.cs.technion.ac.il/~erez/Papers/wf-simulation-ppopp14.pdf).

[MS96]: /home/daniel/nvme_storage/repos/rust/left-right/research_papers/248052.248106.pdf
[KP11]: /home/daniel/nvme_storage/repos/rust/left-right/research_papers/wfquque-ppopp.pdf
[TP14]: /home/daniel/nvme_storage/repos/rust/left-right/research_papers/2692916.2555261.pdf
[read-enter]: /home/daniel/nvme_storage/repos/rust/left-right/src/read.rs:120
[guard]: /home/daniel/nvme_storage/repos/rust/left-right/src/read/guard.rs:110
[guard-map]: /home/daniel/nvme_storage/repos/rust/left-right/src/read/guard.rs:56
[registration]: /home/daniel/nvme_storage/repos/rust/left-right/src/read.rs:87
[read-drop]: /home/daniel/nvme_storage/repos/rust/left-right/src/read.rs:55
[factory]: /home/daniel/nvme_storage/repos/rust/left-right/src/read/factory.rs:36
[append]: /home/daniel/nvme_storage/repos/rust/left-right/src/write.rs:482
[extend]: /home/daniel/nvme_storage/repos/rust/left-right/src/write.rs:534
[publish]: /home/daniel/nvme_storage/repos/rust/left-right/src/write.rs:359
[wait]: /home/daniel/nvme_storage/repos/rust/left-right/src/write.rs:249
[flush]: /home/daniel/nvme_storage/repos/rust/left-right/src/write.rs:465
[try-publish]: /home/daniel/nvme_storage/repos/rust/left-right/src/write.rs:325
[take-inner]: /home/daniel/nvme_storage/repos/rust/left-right/src/write.rs:149
[update-swap]: /home/daniel/nvme_storage/repos/rust/left-right/src/write.rs:381
[absorb]: /home/daniel/nvme_storage/repos/rust/left-right/src/lib.rs:194
[alias-example]: /home/daniel/nvme_storage/repos/rust/left-right/tests/deque.rs:83
[loom-sync]: /home/daniel/nvme_storage/repos/rust/left-right/src/sync.rs:5

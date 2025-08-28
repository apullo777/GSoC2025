# GSoC2025

<h1 align="center">Google Summer of Code 2025:  Work Product Submission</h1>

- **Contributor**: Peter Chang (<a href="https://github.com/apullo777">@apullo777</a>)</h3>
- **Mentors**: Serathius ([@serathius](https://github.com/serathius)), Madhav Jivrajani ([@MadhavJivrajani](https://github.com/MadhavJivrajani))
- **Organization**: [CNCF](https://summerofcode.withgoogle.com/programs/2025/organizations/cncf)
- **Project**: [Generic High-Performance Caching Library for etcd](https://summerofcode.withgoogle.com/programs/2025/projects/9A3Z5dv1)

## Project goals
etcd is a key-value store that serves as the authoritative source of cluster state for distributed systems like Kubernetes. Clients read data with `Get` and range queries, and use `Watch` to subscribe to change events. Each update gets a monotonically increasing revision so clients can read or watch at specific points in time.

Management components in distributed systems typically read current state and watch for updates to maintain desired configurations. At scale, this creates problems: (1) many concurrent queries overload etcd, and (2) catching up fallen-behind watch subscribers is expensive, and further complicated by etcd compaction and network issues.
   
Kubernetes solves this with a local watch cache in the API server: it maintains a live event stream and in-memory snapshot so management components can read/watch without hitting etcd directly. However, this solution is tightly integrated into Kubernetes and not easily reusable by other projects needing scalable watch semantics. 

This project aims to provide an experimental watch cache for etcd so other infrastructure projects can adopt Kubernetes’ watch-cache pattern without depending on the Kubernetes codebase. The main goals were:

- Combine multiple local watchers into a single etcd watch, maintain event history, and replay events for slow watchers.

- Keep the latest state and support exact stale reads through snapshots.

- Make the cache usable by other projects.

**Github Issue**: [[GSoC] Develop a caching library for etcd](https://github.com/etcd-io/etcd/issues/19371) 
   
## Current status
   
### Where the code lives (repo: [go.etcd.io/etcd/](https://github.com/etcd-io/etcd))

```
cache/                           # package root
├── cache.go                     # main Cache type, New(), control loop, Watch/Get glue
├── cache_test.go                # unit tests for cache
├── config.go                    # configurable knobs + defaults (WithHistoryWindowSize, etc.)
├── demux.go                     # demultiplexer: fanout, replay, compact/purge logic
├── demux_test.go                # unit tests for demux
├── watcher.go                   # per-client watcher structure and buffered delivery
├── predicate.go                 # key predicate helpers (prefix, range, fromKey)
├── ready.go                     # readiness gating helper (WaitReady/Set/Reset) — used by Watch/Get
├── ready_test.go                # unit tests for the ready helper
├── ringbuffer.go                # append-only generic ring history (used for events & snapshots)
├── ringbuffer_test.go           # unit tests for ringbuffer
├── store.go                     # BTree-backed store (Restore/Apply/Get) + snapshot ring
├── store_test.go                # unit tests for store
└── snapshot.go                  # snapshot representation & helpers

tests/
└── integration/
    └── cache_test.go            # end-to-end integration tests exercising Get, Watch, replay, unsupported opts.
```

### Core components implemented

- **Watch demultiplexer**: Demux accepts many local watch subscriptions, combines them into one upstream etcd watch, and fans out events to subscribed watchers (via `demux.Broadcast`). If a watcher falls behind - either because it misses revisions or its delivery buffer overflows — the demux marks it lagging, moves it into a separate lagging set, and schedules periodic resync attempts at a configurable interval (via `demux.resyncLoop`). On each resync the demux tries to locate the watcher’s requested start revision in the local event history and replays the matching slice of events in-order to the watcher (See `demux.go`, `watcher.go`.)

- **Event stream history**: Demux keeps an append-only history of recent watch events so lagging watchers can be resynced without reopening upstream watch. When the history reaches its configured capacity it evicts the oldest events to make room for newer ones. If a lagging watcher’s requested revision is older than the retained history (i.e., compacted), the watcher is canceled with a compaction error; when replay completes successfully the watcher is moved back to the active set. (See `ringbuffer.go`, `demux.go`.)
   
- **B-Tree state mirror & snapshot history**: After receiving an initial snapshot, the cache builds an in-memory B-Tree of key/value state and incrementally applies watch events to keep it up to date (via `store.Apply`). The store itself subscribes to the demux as a normal watcher. It also maintains point-in-time B-Tree snapshots in a ring so callers can serve range queries efficiently when the requested revision is still available in history. (See `ringbuffer.go`, `store.go`, `snapshot.go`.)
   
- **Single-cache control loop**: The cache loop (`Cache.getWatchLoop`) performs an initial snapshot request to establish the current state, subscribes to upstream watch events based on that snapshot’s revision, applies them to the snapshot history (store), and pushes events into the demux's event stream history. Because the store is a demux watcher, `store.Apply` and `demux.Broadcast` proceed concurrently, reducing lock contention between updating the store and broadcasting events to watchers. (See `cache.go`.)

- **Cache initialization & re-initialization**: Building on the initial snapshot + upstream-watch control loop described above, a `ready` helper was added on top of this initialization/rebuild process to gate `Watch/Get` request until the cache is connected to the upstream watch and healthy. On upstream errors like compaction the cache notifies affected watchers, purge history/watchers if needed, and then re-populate the store and rebuild the upstream watch. (See `ready.go`, `cache.go`, `watcher.go`).

### What clients can do today:
The cache exposes the same `Get` and `Watch` API surface as etcd’s client. For both APIs a revision of 0 denotes the current/latest state. Both `Get` and `Watch` support single-key queries, explicit ranges (`WithRange()`), prefix queries (`WithPrefix()`), and `WithFromKey()` when allowed by the cache prefix.
   
- **Get**: The cache currently only supports serializable reads (`Get(..., WithSerializable())`), with revision 0 `Get` returns the current/latest state and the cache’s latest revision in the response header. `Get(..., WithRev(N))` serves exact stale reads when N is retained in the cache’s snapshot history; future revisions return `ErrFutureRev`, and revisions older than the history window are reported as compacted (`ErrCompacted`).

- **Watch**: `Watch` started at revision 0 receives live updates from the cache’s latest state; `Watch(..., WithRev(N))` will be replayed from history if N is available (`ErrCompacted` returned otherwise). 
   
Unsupported / rejected options (for now): advanced `Get/Watch` options such as `WithPrevKV`, `WithProgressNotify`, `WithCreatedNotify`, count/limit/sort, filtering options, and other non-serializable/advanced behaviors are intentionally rejected (tests assert `ErrUnsupportedRequest`).

### Tests & validation

- **Unit tests** live next to each component (`*_test.go`) and validate correctness and concurrency.
- **Integration tests** (under `tests/integration`) assert supported behaviors, replay semantics, and unsupported-option handling. They reuse the etcd repo’s integration test helpers (for example `integration.BeforeTest` and `integration.NewCluster`) to spin up a real etcd cluster and client, then run the cache against that cluster. 
   
- **Robustness tests**: We planned to reuse the etcd project’s [robustness testing tools](https://github.com/etcd-io/etcd/tree/main/tests/robustness) to validate the cache’s correctness (model-based linearizability checks and advanced watch validation), but decided this should be postponed to later stage once the core features are finished. Mentor provided a draft integration showing how robustness tests could be wired up (see [PR #20354](https://github.com/etcd-io/etcd/pull/20354)) and a permanent approach will require writing a separate, cache-specific test scenario, while it already helped surface real bugs (for example, see [issue #20488](https://github.com/etcd-io/etcd/issues/20488)).
   
### Known issues (not yet resolved)
- **Resync / history-eviction race that can compact the store watcher and trigger a purge**: under certain timings (with `PerWatcherBufferSize = 0`, or larger `ResyncInterval`) the store watcher can be moved to lagging with a `nextRev` that becomes older than the history’s oldest after broadcast/event bursts and history eviction. When `resyncLaggingWatchers` runs it finds lagging watchers whose `nextRev < oldestRev`, the store watcher gets compacted/closed and that path triggers a `Purge`, which can drop events before client watchers can replay them. This involves the ring-history eviction + demux/resync + watcher interaction. (Files: `ringbuffer.go`, `demux.go`, `watcher.go`, `cache.go`.)
- **Flaky tests & postponed fix**: tests like `TestCacheLaggingWatcher` & `TestCacheWatchOldRevisionCompacted` (temporarily removed because it was too flaky) fail intermittently due to the above issue. These two flaky-race tests share the same underlying root cause, and are non-trivial to fix safely. As discussed with mentors, this requires someone with deeper concurrency experience to design a correct approach. So we documented it and postponed a fix for follow-up. (See `tests/integration/cache_test.go`).

## Next steps
After GSoC, the cache is functionally useful but needs additional work to make it robust and feature-complete. These include: testing (more unit/integration, end-to-end, and robustness tests), metrics (cache size, latency tracking, buffer monitoring), benchmarks (performance testing for watch operations and read throughput), and new features like custom codecs, secondary indexing, and consistent reads.

Here are some possible next steps:

- **Consistent reads**: Support linearizable reads from the cache while maintaining the same strong consistency guarantees as reading directly from etcd, so callers can request up-to-date data without performance penalties.

- **Add min/max revision to demux (for efficiency)**: Letting the demux seek directly to the relevant revision avoids scanning irrelevant event ranges.

- **Generate progress notification from cache (for robustness tests)**: Allow clients to request for progress updates by sending them the current cache revision even when no events exist.
   
- **Re-enable unsupported Watch/Get options**: Restore options like `PrevKV`, `Limit`, and `Sort` so `Watch/Get` semantics match upstream, and need tests to prove these features work.

- **Indexing**: Maintain secondary indexes on updates so non–primary-key queries can be answered quickly without full scans.

- **Encoding/decoding**: Pluggable Codec (Marshal/Unmarshal) so projects can use custom serializations or compression.

## What code got merged (or not) upstream   
   
PRs submitted to etcd by apullo777: See [here](https://github.com/etcd-io/etcd/pulls?q=is%3Apr+author%3Aapullo777) and detailed list below:

| Pull Request  | Description | Status  |
| :-----------  | :----------:  | -------: |
| [#20160](https://github.com/etcd-io/etcd/pull/20160)          | cache: implement MVP watch demux |  Merged  |
| [#20274](https://github.com/etcd-io/etcd/pull/20274)          | cache: batch events with identical revision into one watch response  |  Merged  |
| [#20284](https://github.com/etcd-io/etcd/pull/20284)          | cache: refactor PeekLatest/PeekOldest |  Merged  |   
| [#20285](https://github.com/etcd-io/etcd/pull/20285) | cache: remove AfterRev entry predicate | Merged | 
| [#20297](https://github.com/etcd-io/etcd/pull/20297) | cache: refactor cache_test.go (cache/ -> tests/integration/) | Merged | 
| [#20310](https://github.com/etcd-io/etcd/pull/20310) | cache: fix waitGroup goroutine registration | Merged | 
| [#20318](https://github.com/etcd-io/etcd/pull/20318) | cache: add cache unit tests with mocked client.Watcher | Merged | 
| [#20326](https://github.com/etcd-io/etcd/pull/20326) | cache: enable range, prefix, fromKey key filtering in cache watch | Merged | 
| [#20345](https://github.com/etcd-io/etcd/pull/20345) | cache: refactor tests into predix vs. no-prefix cases | Merged | 
| [#20347](https://github.com/etcd-io/etcd/pull/20347) | cache: enable Watch on arbitrary start_revision | Merged | 
| [#20350](https://github.com/etcd-io/etcd/pull/20350) | cache: improve tests to validate atomic grouping and monotonic revisions | Merged | 
| [#20362](https://github.com/etcd-io/etcd/pull/20362) | Expose opWatch & boolean accessors for Watch opts so the cache can validate unsupported watch flags | Merged | 
| [#20363](https://github.com/etcd-io/etcd/pull/20363) | cache: handle unsupported watch opts | Merged | 
| [#20372](https://github.com/etcd-io/etcd/pull/20372) | cache: implement storage for last-observed state | Merged | 
| [#20378](https://github.com/etcd-io/etcd/pull/20378) | Expose cancelReason so cache can validate unsupported watch | Merged | 
| [#20385](https://github.com/etcd-io/etcd/pull/20385) | modify the comment for the now public CancelReason | Merged | 
| [#20390](https://github.com/etcd-io/etcd/pull/20390) | cache: rename ErrUnsupportedWatch, add ErrKeyRangeInvalid | Merged | 
| [#20398](https://github.com/etcd-io/etcd/pull/20398) | expose IsSortSet so that cache can validate unsupported get flags | Merged | 
| [#20399](https://github.com/etcd-io/etcd/pull/20399) | cache: rename validateWatchRange as validateRequestRange | Merged | 
| [#20419](https://github.com/etcd-io/etcd/pull/20419) | cache: refactor validateWatch to use a switch for unsupported watch ops | Merged | 
| [#20443](https://github.com/etcd-io/etcd/pull/20443) | cache: refactor TestCacheWithoutPrefixGet for different initialization scenarios and added shared applyEvents helper | Merged | 
| [#20447](https://github.com/etcd-io/etcd/pull/20447) | cache: add unit test injecting mid-way compaction | Merged | 
| [#20448](https://github.com/etcd-io/etcd/pull/20448) | cache: consolidate ErrKeyRangeInvalid assertion in TestCacheWithPrefixGet | Merged | 
| [#20453](https://github.com/etcd-io/etcd/pull/20453) | cache: refactor ready status using isReady with readyChan and mutex to prevent races | Merged | 
| [#20465](https://github.com/etcd-io/etcd/pull/20465) | cache: generalize ready channel to signal any state change instead of ready-only | Merged | 
| [#20476](https://github.com/etcd-io/etcd/pull/20476) | cache: preserve cached snapshot after watch errors | Merged | 
| [#20477](https://github.com/etcd-io/etcd/pull/20477) | cache: make ring buffer generic for events and snapshots | Merged | 
| [#20480](https://github.com/etcd-io/etcd/pull/20480) | cache: refactor ringbuffer to add iterator methods | Merged | 
| [#20483](https://github.com/etcd-io/etcd/pull/20483) | cache: make store a normal demux watcher to reduce lock contention | Merged | 
| [#20491](https://github.com/etcd-io/etcd/pull/20491) | cache: preserve resumable guarantee with empty history watchers | Merged | 
| [#20498](https://github.com/etcd-io/etcd/pull/20498) | cache: implement binary search for ringBuffer range iterations | Open | 
| [#20499](https://github.com/etcd-io/etcd/pull/20499) | cache: remove early compaction check from Cache.Watch | Merged | 
| [#20502](https://github.com/etcd-io/etcd/pull/20502) | cache: move batching to demux so that ringBuffer stores single item | Merged | 
| [#20507](https://github.com/etcd-io/etcd/pull/20507) | cache: migrate storage layer to B-tree | Merged | 
| [#20519](https://github.com/etcd-io/etcd/pull/20519) | cache: change watcher.eventQueue to respCh (clientv3.WatchResponse) | Merged | 
| [#20543](https://github.com/etcd-io/etcd/pull/20543) | cache: add unit tests and validation for store | Merged | 
| [#20564](https://github.com/etcd-io/etcd/pull/20564) | cache: implement snapshots & enable cache.Get(rev>0) | Open | 
| [#20566](https://github.com/etcd-io/etcd/pull/20566) | cache: rename TestCacheWithoutPrefixGet test cases for clarity | Merged |  


## Important things I learned during the project
Below are things I learned during GSoC that I wish I had known before starting, so I'm writing them down to help future participants:
   
- **Communication is key**. It's much better to ask clarifying questions about next steps rather than making assumptions about what needs to be done.
- **Seek timely feedback instead of making massive changes in isolation**. Small, steady commits that receive regular feedback are far more valuable than large modifications done without input. It's also helpful to communicate implementation plan before starting major work.
- **Always prioritize minimal solutions and avoid unintended side effects**. This is especially crucial when modifying code that has already been reviewed and approved by mentors or the community.
- **Balance new functionality with code readability**. Adding features often comes at the cost of code clarity, so it's important to find the right equilibrium between the two.
- **Strategic procrastination helps**. Sometimes it's wise to postpone difficult problems until later development stages when the overall architecture is more mature and better equipped to handle complex solutions.
- **Maintain quality by reducing scope**. When tasks prove tougher than expected, discuss with mentors and reduce scope rather than compromising quality.
- **Expect challenges and stay resilient**. Mistakes are common under time pressure and when under-prepared. Even with great guidance, implementation can fail and cause frustration. Focus forward, take responsibility, and stay curious. Most importantly, remember to rest for long-term success.


## Acknowledgement
I am deeply grateful to Google Summer of Code, the etcd community, and my mentors for providing me with this precious opportunity to learn and grow as an open source contributor. I feel incredibly fortunate to have worked with mentors who possess the best qualities one could hope for - exceptional technical skills and standards combined with genuine kindness. Their good taste in code design and unwavering commitment to quality created an environment where every interaction became a learning opportunity. Without their thoughtful investment in my growth and patient guidance, I would have been completely lost and unable to move forward with this project. 
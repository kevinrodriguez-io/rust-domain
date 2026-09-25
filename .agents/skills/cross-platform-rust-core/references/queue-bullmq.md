# Queue: BullMQ v6 on Redis, OSS only

Expansion guide. The starting scope is Android and iOS. Apply this only when adding a server.

Verified 2026-09-19. Pins in versions.md. **Use OSS BullMQ. No Pro licence.**

## Redis requirements

- **Redis 6.2.0 or newer.** BullMQ is "full Redis compliant with version 6.2.0 or newer."
- **`maxmemory-policy` must be `noeviction`.** BullMQ's docs: "BullMQ… cannot work properly if Redis evicts keys arbitrarily. Therefore is very important to configure the `maxmemory-policy` setting to `noeviction`. This is the **only** setting that guarantees the correct behavior of the queues." Any other policy silently corrupts queue state.
- **Install `ioredis` explicitly.** BullMQ v6 made it an *optional peer dependency*. This is the most common v6 upgrade failure.

Dragonfly is a supported drop-in alternative. BullMQ v6 also supports node-redis (peer dep `redis >= 5.0.0`), Bun's built-in client, and Valkey Glide through its `IRedisClient` adapter interface. Default to ioredis unless you have a reason not to.

## v6 breaking changes

If you are extending an existing service, check all of these before bumping:

| Removed / changed | Replacement |
|---|---|
| `Connection` constructor parameter | Optional `BackendFactory` (Redis **or** PostgreSQL backends) |
| `Queue#client`, `Queue#redisVersion`, `Queue#databaseType`, `Worker#blockingClient`, `FlowProducer#client` | `getBackend()` → `RedisQueueBackend` |
| `Worker#waitUntilReady()` returned the client | Now resolves to `void` |
| `ioredis` as a direct dependency | Optional peer dependency — install it yourself |
| Legacy repeatable jobs: `repeat` option, `Repeat` class, `getRepeatableJobs()`, `removeRepeatable()`, `removeRepeatableByKey()` | **Job Schedulers** |
| The `paused` job state in `JobType` and `getJobCounts()` | Jobs in a paused queue report as `waiting` |
| `Worker#resume()` was sync | Now async — must be awaited |
| `debounce`, `Job#debounceId`, `debounced` event | `deduplication`, `Job#deduplicationId`, `deduplicated` event |
| `Job#discard()` | Throw `UnrecoverableError` |
| `RepeatOptions` accepting `currentDate`, `utc`, `nthDayOfWeek` | Use `tz: 'UTC'` instead of `utc: true` |
| Exports `Scripts`, `createScripts`, `JobJsonRaw`, `RedisJobOptions` | Backend APIs |
| Telemetry `JobFinishedTimestamp`, `TelemetryAttributes.JobStatus` | `JobState`; `Meter#createGauge()` is now required for adapters |

Dashboards that count the `paused` state will silently read zero after upgrading.

## Ordering: what you actually get

**FIFO guarantees start order, not completion order.** BullMQ's own words: "The jobs are processed in the same order as they are inserted into the queue… however, if you have more than one worker or a concurrency factor larger than 1, even though the workers will start the jobs in order, they may be completed in a slightly different order."

Combined with the stalled-job path below, **true end-to-end ordering requires concurrency 1 across the queue**, which sacrifices all throughput.

**This is fine, because the push transports do not preserve order either.** APNs "stores only one notification per bundle ID" when a device is offline — queued notifications collapse into one, "usually but not reliably the latest" — and priority 5 and 1 notifications "might get grouped and delivered in bursts." So ordered sends would feed a transport that reorders and collapses them anyway.

**Design for order-insensitivity.** Use `apns-collapse-id` / FCM `collapse_key` plus a monotonic state version in the payload so the latest state wins, rather than trying to force FIFO.

## At-least-once and stalled jobs

On pickup BullMQ locks the job, and the worker must periodically renew the lock (`stalledInterval`, "which normally you should not need to modify"). If the event loop is blocked long enough that renewal fails, the job is marked **stalled**, moved back to `waiting`, and **processed again by another worker**. Past its maximum stall count it goes to `failed`.

**Therefore every push can be delivered twice.** Requirements that follow:

1. Set `apns-collapse-id` (≤ 64 bytes) and/or FCM `collapse_key`.
2. Carry an application-level dedup key and check it before sending.
3. Never treat "job completed" as "user saw exactly one notification."
4. Keep processors off the event loop. A CPU-blocked worker manufactures duplicates. Use sandboxed processors for CPU-heavy work.

## The OSS toolkit

| Lever | API | Semantics |
|---|---|---|
| Global concurrency | `queue.setGlobalConcurrency(n)` | Cap across **all** workers. A worker's local factor never overrides it |
| Global rate limit | `queue.setGlobalRateLimit(max, durationMs)` | Queue-wide throughput cap. Also `getGlobalRateLimit()`, `getRateLimitTtl()`, `removeGlobalRateLimit()` |
| Worker rate limit | `{ limiter: { max, duration } }` | **Already global per queue** — 10 workers with `max: 10` still process 10 jobs/second total |
| Local concurrency | `{ concurrency: n }` | Per-instance; mutable at runtime via `worker.concurrency = n` |
| Deduplication | `deduplicationId` | Replaces the removed `debounce` |
| Priority / delayed / LIFO | job options | All OSS |
| Job Schedulers | — | Recurring work |
| Flows | `FlowProducer`, `addBulk` | Parent/child dependency ordering |
| Bulk enqueue | `Queue.addBulk` | Fewer Redis round-trips than `add` in a loop |
| Cancellation | `async (job, token, signal)`, `worker.cancelJob(id, reason)` | Correct graceful-shutdown mechanism for in-flight sends |

Two traps:

- **Rate-limited jobs stay in the `waiting` state**, not `delayed`. Monitoring that reads a growing `waiting` count as a backlog will misdiagnose a healthy rate-limited queue.
- **The limiter's `groupKey` was removed in BullMQ 3.0** — "group keys support is removed to improve global rate limit." Many tutorials still show `limiter: { groupKey: 'customerId' }` for per-customer limits. It does not exist on v6; per-group rate limiting is a Pro feature.

## What Pro would add, and why you do not need it

Pro adds groups (per-group FIFO with round-robin across groups), local group concurrency, group rate limiting, max group size, group pausing, prioritized groups, batch *processing* via `job.getBatch()`, and observables.

The one real gap is **round-robin fairness** — preventing a noisy tenant from starving others. Options, in order of preference:

1. **Accept unordered, make sends idempotent.** Correct for most push workloads, and no starvation if throughput is adequate.
2. **Fixed shard set** — N queues (say 8 or 16), `shard = hash(tenantId) % N`, one or more workers per shard, per-shard rate limits. A hot tenant degrades only its shard, at a bounded and known cost. This is a design pattern, not a BullMQ feature. Reversible: if you later buy Pro, groups replace the hash.
3. One queue per tenant — true isolation but does not scale.
4. `setGlobalConcurrency(1)` — total serialization; only for a deliberately slow, strictly-ordered path.

Note on "batches": `Queue.addBulk` and `FlowProducer.addBulk` are **OSS**. Only batch *processing* (many jobs in one worker callback) is Pro. "Worker `concurrency` runs several jobs in parallel, but each processor call still receives **one** job."

## Throttling to match FCM

FCM's fanout capacity "is divided among projects and not across fanout requests," and Google explicitly recommends "only have one active fanout in progress at a time." Use `setGlobalConcurrency(1)` on a dedicated topic-fanout queue, kept separate from the per-device send queue so it does not throttle everything.

FCM's downstream quota is 600,000 messages/minute/project by default, and "quotas are per-minute, but these minutes are not aligned to the clock" — console graphs "are not precisely time aligned with quota minutes, meaning 429s may be served when traffic appears to be under quota." Do not debug 429s from the graph; use `setGlobalRateLimit` to stay clear.

## Connection lifetime is architectural

APNs may treat a provider that "opens and closes its connection to APNs repeatedly" as a denial-of-service attack and temporarily block it; connections should be reused "for many hours to days." So the APNs client must be a **long-lived, process-scoped HTTP/2 connection pool created once per worker process**. Constructing a client inside a job processor is the default mistake in this architecture.

Do not forget: every BullMQ class consumes at least one Redis connection, and `Worker` and `QueueEvents` create additional duplicated connections internally for blocking commands.

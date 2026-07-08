# Before

- Kepler - Agency
- PBAC
- Hub needed dynamic sharing
- A new beta product, not exposed to clients
- A relatively new system deployed nimbly


## Use Cases

- Multiple orgs, multiple workspaces, multiple products
- Modular, for monorepo
- Context based on Auth0 org
- NC for tuples
- Store per PS envs, bootstrap idempotent tuples, inject admin
- Each check has a corresponding function
- List by filter-then-BatchCheck, never ListObjects
  - Pagination is by db, infinite scroll gets around this.

# During

- Downtime?!
- Complex model by design
- Complex model -> 1 check exceeded max default conns (30) -> Conns "saturate"? -> Healthcheck pings -> k8s kills
- POC:
  - Model of sufficient complexity
  - Show how it blows up

# Future

- Scaled: Increased db, fine tuned conns, cache, monitoring
- Updated model within_app, within_product, etc.
- Core auth logic published as package
- Continue to fine tune

---

- can_access is a nested AND where one operand (can_access from app) is itself an AND (core.fga:40 + :32).
- intersection() launches operands as concurrent goroutines (internal/graph/check.go:234-243), so the tree fans out breadth-first in parallel.
- Each leaf read holds a pooled connection for the iterator's full lifetime (SQLTupleIterator over *sql.Rows, sqlcommon.go:306), not just one fast query.
- ResolveNodeBreadthLimit (default 10) is per-node, not global, and MaxConcurrentReadsForCheck defaults to math.MaxUint32 — no admission control in front of the pool.
- Result: outer checks hold connections while blocked waiting on child checks that can't get connections. Classic hold-and-wait cycle. Failure is binary — fine below the connection ceiling, total wedge above it,
  surfacing as uniform deadline_exceeded with tiny datastore_query_count.

The k8s cascade on top of it is a nice second act: the health-check DB ping also can't acquire a connection, so the pod fails its probe, gets killed, restarts, re-maxes on the same fan-out, and loops. The failure mode of an undersized pool isn't graceful degradation — it's a crash loop. That's a concrete, reproducible lesson worth showing maintainers, possibly as a "model of sufficient complexity → watch it blow up" POC (which your outline already gestures at).

### Inaccurate / Incomplete
- **"Every branch holds a connection" (slide 231, outline:41):** overstated. In our model two of three AND operands hold zero connections — `user_in_context from org` is served from the contextual tuple in-memory (`CombinedTupleReader`); computed-userset and cache-hit branches hold none. Only tupleset reads (`org`, `app`) and the direct `member` lookup hit the pool, and only while actively reading rows — not for the branch's full lifetime. The "iterator's full lifetime" phrasing in outline:41 is imprecise: connection is held during row iteration, released on drain/Stop.
- **"Deadlock!" (slide 251):** it's a timeout-cleared resource deadlock, not permanent. Every wait point respects context cancellation; the 3s request timeout releases connections (→ `deadline_exceeded`). Slides 247 and 251 slightly contradict — reconcile.
- **`MAX_CONCURRENT_READS_FOR_CHECK` as "the fix" (slides 238/280):** sufficient for the minimal repro (empirically confirmed by README) — bounding reads breaks hold-and-wait, reads queue instead of deadlocking.
  - Mechanism nuance: `BoundedTupleReader` releases its limiter slot via `defer b.done()` when the iterator is *returned*, while the pgx connection is acquired lazily on first `Next()` (`fetchBuffer`). Slot lifetime and connection lifetime are decoupled for streaming reads. So the cap works as a *concurrency throttle on how many checks enter their read phase at once*, not a hard cap on held connections. Under different batch/gather concurrency it's one lever, not a full connection guarantee.
  - Also relevant: **`ResolveNodeBreadthLimit` (default 10):** bounds how many AND/OR branch goroutines run concurrently per node in a single check's resolution tree. Per-node, not global. (noted in outline above).
  - Possibly relevant: **`MaxConcurrentChecksPerBatchCheck` (default 50):** bounds how many items in a single BatchCheck resolve at once. Does not affect the client-side `gather` of N RPCs.
- **Model remediation (slides 291-309):** the diff removes `can_access from app` (fewer reads per product check). "Check app access once per request, then batch" is app-side logic, not the model change. Slide conflates the two.

## TODO Plan

- What happened
- Remediation
- Flesh out use cases
--
- Flesh out accuracy (above)
- Update slides to reflect new fix
-> Very rough draft DONE
- Flesh out each section
- And Slides


### Considerations

Why not #member

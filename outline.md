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

The can_access_within_app optimization

core.fga:42-47 is a deliberate model-level workaround for check cost that maintainers would recognize:

```
define can_access: member from org and user_in_context from org and can_access from app
define can_access_within_app: member from org and user_in_context from org
```

You factored the expensive can_access from app subtree out of the per-product check so it can be evaluated once per request instead of once per product, then batch-check the cheaper relation across 30+ products (authz.py:1093 check_can_access_products). The model carries a comment explicitly flagging this as a pattern to apply "across the board." That's an interesting design tension to surface: the cleanest model isn't the cheapest model, and you're trading model purity for amortized check cost.

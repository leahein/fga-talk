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

The k8s cascade on top of it is a nice second act: the health-check DB ping also can't acquire a connection, so the pod fails its probe, gets killed, restarts, re-maxes on the same fan-out, and loops. The failure mode of an undersized pool isn't graceful degradation — it's a crash loop. That's a concrete, reproducible lesson worth showing maintainers, possibly as a "model of sufficient complexity → watch it blow up" POC (which your outline already gestures at).

## TODO Plan

- What happened
- Remediation
- Flesh out use cases
- Flesh out accuracy (above)
- Update slides to reflect new fix
- Very rough draft DONE
- Flesh out each section
- And Slides
- Diagram
- draft DONE
- Downtime Model + Check (product -> artifact), @fga-max-conns
--
-> Throughline! Weave full story
- Images??
- Practice
- Time it


### Considerations

Why not #member

# FGA @ Kepler

---

## Goal

Understand the past, present, and future use of OpenFGA at Kepler Group.

TODO: Weave into story

---

## Kepler Group

A digital marketing agency.

---

### Kyu

Kepler is a part of **Kyu**, a global network of agencies, including BIMM, Sid Lee, and others.

---

### Kyu Hub

An AI-powered platform designed for collaboration across Kyu companies.

Note:
This is what we'll be talking about today, and where we use OpenFGA.

---

## PBAC

- Because Kepler is an agency working with clients, we have strict access control requirements per client.
- In our core system, we used a PBAC (Policy-Based Access Control) model, similar to AWS IAM, which worked well for isolating data per client.
- This required a separate policy for every type of access requirement.
- This worked well for our core system, where access was static and defined once per app and client.

---

## ReBAC

However, in Kyu Hub, we needed a more dynamic model, where data can be shared dynamically between users and work groups, as well as potentially across Kyu companies, for collaboration.

This required a more fine grained model, where access is defined by relationships between users and resources, rather than static policies.

TODO: Describe Hub

- Hub is a relatively new system, deployed nimbly, and not yet exposed to clients.

---

### OpenFGA

- **Solid Feature Set**: Complex model expressiveness, wildcard / public access,  caching, and datastore flexibility.
- **Strong Developer Experience**: Excellent documentation with many examples and best practices, easy docker setup, and good tooling (local playground, CLI).
- **Auth0 Alignment**: OpenFGA is backed by Auth0, which Kepler already uses for authentication, simplifying vendor management.
- **Flexibility**: It offers a solid open-source offering and licence (Apache 2.0 license) for self-hosting, plus a managed option, giving us the flexibility to transition as our needs evolve.
- **Active Community**: A CNCF project with an active Slack channel and responsive maintainers.


Note:
We evaluated several differenr tools, and chose OpenFGA for the following.

---

## Use Cases

---

### CI/CD

#### Staging

- Every developer gets a personal staging environment, which is a dynamically spun-up copy of the whole app. This also dynamically brings up a developer specific fga store, with the tuples seeded in our system, and assigns the user as the "admin" of that store.

#### Production

- On every deployment that includes a change to fga, we ensure the store is seeded with the necessary tuples, and that the model version gets updated.

---

### Multi-Org Users in Context

- Users can operate across organizations, such as being a part of Kepler or a parent company.
- We leverage Auth0 Organizations, so that the user logs in to a specific organization, and that context is passed into their token.
- Our system will extract the organization from the token, and pass that into fga as a contextual tuple.
- This ensures we always use the correct organization context for authorization checks, even if the user is a member of multiple organizations.
- Note: Tests do not support contextual tuples yet.

---

### Filter then BatchCheck vs ListObjects


---

### Each check covers everything

- Can access app
- Member from org
- Member from client

---

## The Downtime

30 minutes after a routine model deploy, our fga service went down and couldn't come back up.

---

### Background

- 3 Pods
- Small DB
- No min / max configuration
- No caching enabled
- Defaults per pod: 30 max connections, unlimited concurrency

Note:
Hub is a young, pre-client beta. We deployed nimbly to move fast; infra was provisioned to get it running, not for production load.

---

```bash
DATASTORE_MAX_OPEN_CONNS        = 30
MAX_CONCURRENT_READS_FOR_CHECK  = unlimited (MaxUint32)
CHECK_QUERY_CACHE_ENABLED       = false
```

Nothing throttles reads before they hit the pool.

---

### Event

- All FGA pods crashing
- Hub inaccessible to all users
- Pods restarting in a loop

---

### Investigation

- Pods were exhausting all available DB connections
- Restart → instantly max out connections → crash, in a loop

---

### Root Cause

1. Deployed a more complex model, which increased the number of connections needed for a check.

---

```rb
type user

type org
  relations
    define member: [user]
    define user_in_context: [user]

type app
  relations
    define org: [org]
    define can_access: member from org and user_in_context from org

type product
  relations
    define app: [app]
    define org: [org]
    define can_access: member from org and user_in_context from org and can_access from app
```

Note:
`product.can_access` check is a nested AND

---

2. A batch check fired off a bunch of concurrent checks, which all needed connections.

---

```python
async with OpenFgaClient(config) as client:
    items = [
        ClientBatchCheckItem(
            user="user:u1",
            relation="can_access",
            object=f"product:p{i}",
            contextual_tuples=[
                ClientTuple("user:u1", "user_in_context", "org:o1")
            ],
        )
        for i in range(1, 21)
    ]
    await asyncio.gather(
        *(
            client.batch_check(ClientBatchCheckRequest(items))
            for _ in range(10)
        )
    )
```

---

3. Each check explores its `AND` branches in parallel, every branch holding a connection.
4. A parent can't release its connection until its children resolve, but the children can't get connections to resolve.

---

5. Deadlock!

---

6. Kubernetes healthcheck pinged the DB, couldn't get a connection, and failed.
7. K8s killed the pod, it restarted, and the cycle repeated.

---

## Remediation

---

### Infra

- Right-sized the DB: t4g.small → t4g.medium
- Raised max DB connections per pod

---

### Config

- Stable pool: min idle / min open connections
- Capped concurrent reads per check
- Enabled caching

```bash
DATASTORE_MIN_OPEN_CONNS
DATASTORE_MIN_IDLE_CONNS
MAX_CONCURRENT_READS_FOR_CHECK
CHECK_QUERY_CACHE_ENABLED
```

Note:
The pool settings just keep connections warm.
The concurrency cap is to address the root cuase, so not that it now queues instead of deadlocking.

---

### Model

- App access was re-checked for every product in the batch
- Factored it out into a new relation that skips the app subtree
- Check app access once per request, then batch the product checks

Fewer nested checks, fewer connections.

---

### Model

```diff
 type product
   relations
     define app: [app]
     define org: [org]
-    define can_access: member from org and user_in_context from org and can_access from app
+    define can_access_within_app: member from org and user_in_context from org
```


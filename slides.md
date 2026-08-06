# OpenFGA @ Kepler

---

## Summary

- About Kepler
- Why OpenFGA
- From Simple to Scalable

Note:
- The context of Kepler
- Why we use OpenFGA
- The 3 ways in which we started out simple and how we evolved to a more scalable solution.

---

### Leah Einhorn

Principal Software Engineer @ Kepler

Authentication & Authorization

---

## Kepler Group

A digital marketing agency

Note:
Serving Fortune 500 clients

---

### kyu

Kepler is a part of **kyu**, a global network of agencies

<div style="display: flex; align-items: center; justify-content: center; gap: 2em; margin-top: 1em;">
  <img src="assets/kyu.svg" alt="kyu" style="height: 3em;" />
  <img src="assets/kepler.svg" alt="Kepler" style="height: 3em;" />
  <img src="assets/bimm.svg" alt="BIMM" style="height: 3em;" />
  <img src="assets/sid-lee.svg" alt="Sid Lee" style="height: 3em;" />
</div>

---

## AuthZ @ Kepler

<!-- ### From Fixed to Flexible -->

---

Strict access control requirements


Note:
Because Kepler is an agency working with clients, and clients may be direct competitors, we have strict access control requirements per client.

---

In our core system we use **Policy-Based Access Control** (PBAC)

```json
{
  "Group": "client-admin",
  "Action": ["advertisers-readAdvertisers"],
  "Effect": "Allow",
  "Resource": ["krn:...:advertiser/CLIENT-NAME/*"]
}
```

- Policies are static and defined once per app and client
- Works well when isolating data per client

Note:

Similar to AWS IAM.

A policy is defined for every type of access requirement.

---

### kyu Hub

A new AI-powered platform designed for collaboration across kyu companies

---


- A platform where AI agents generate artifacts
- Artifacts can be shared dynamically between users
- User's access is determined by their relationship to the kyu company, the client, or the resource

Note:
AuthZ needs are different

Which takes us to...

---

## OpenFGA @ Kepler

<img class="illo" src="assets/database-tables.svg" alt="" />

Note:
This allows us to achieve **fine-grained relationship-based access control model** in a dynamic way that isn't possible with static policies.

---

### From Simple to Scalable


Note:
Now, originally when we deployed Hub nimbly to move fast and get the product up and running; not for production load. As the product grew, we ran into a few issues, including an outage.

So we'll talk about 3 different scenarios for how we started out simply, and what we learned in order to ensure we can scale in the future.


---

## 1. Unbounded -> Controlled Fan-Out

Manage comprehensive checks effectively.

---

### Then

Each check must confirm access to every prerequisite

Note:
It's not enough to have access to the resource.

---

Can access `artifact` + `app` + `org`



```mermaid
flowchart TD
    A["can_access artifact:1"] --> B["owner?"]
    A --> C["can_access from app?"]
    A --> D["member from org?"]
```

- Removing a single relationship cascades to all artifacts

Note:
Say a user owns an artifact. That's not enough.

- Are they owner of the artifact 
- AND access to the app that produced it 
- AND are they a member from the org **aka company**?

**Why?** This is so that if a user is removed from an org / company or a client, or app access is removed, we can simply remove one tuple relationship, that will cascade and remove access to all artifacts.


---

```rb
type artifact
  relations
    define app: [app]
    define org: [org]
    define owner: [user]
    define can_access: owner and can_access from app and member from org
```

Note:
This is a sample of the model.

And this was fine, until we increased the complexity of the model. 30 minutes after a routine model deploy....


---

#### Outage

<div style="text-align: center; font-size: 3em">💥</div>

Note:
...our FGA service went down and couldn't come back up.

---

- Hub inaccessible to all users
- All FGA pods crashing
- Pods restarting in a loop


---

#### Investigation

OpenFGA was exhausting all DB connections

Note:
Connections would max out → crash pods → restart

---

#### Causal Chain

---

##### Deployment

- **Pods**: 3
- **DB Instance Size**: Small
- **Configuration**: Default

Note:
First, let's understand the state of our deployment at the time.

---

##### Default Configuration

```bash
DATASTORE_MAX_OPEN_CONNS               = 30
MAX_CONCURRENT_CHECKS_PER_BATCH_CHECK  = 50
MAX_CONCURRENT_READS_FOR_CHECK         = unlimited (MaxUint32)
CHECK_QUERY_CACHE_ENABLED              = false
```

_Version: 1.15.1_

---


1. A more complex model increased the number of connections needed to resolve a check

Note:
Okay, so the first thing that happened was we deployed a more complex model that...

---

#### Model

```rb
type org
  relations
    define admin: [user]
    define manager: [user]
    define member: [user] or admin or manager

type app
  relations
    define org: [org]
    define can_access: member from org

type artifact
  relations
    define app: [app]
    define org: [org]
    define owner: [user]
    define can_access: owner and can_access from app and member from org 
```

Note:
`artifact.can_access` check is a nested AND, and member from org is a nested OR.

---

#### Fan-Out

```mermaid
mindmap
    root((can_access<br/>artifact:1))
        (owner)
            ((user))
        (can_access from app)
            (member from org)
                ((user))
                ((admin))
                ((manager))
        (member from org)
            ((user))
            ((admin))
            ((manager))
```

Note:
One check fans out into the owner check plus two org-membership subtrees (one via app, one direct), each an OR over user/admin/manager.

**Every branch is a separate read holding a connection.**

---

2. A Batch Check fans out into many concurrent checks

Note:
So we weren't doing the fan out once, but in a batch check.

All of which are competing for the same pool.

---

#### Batch Check

```python
items = [
    ClientBatchCheckItem(
        user="user:u1",
        relation="can_access",
        object=f"artifact:p{i}",
    )
    for i in range(1, 101)
]

async with OpenFgaClient(config) as client:
    await client.batch_check(ClientBatchCheckRequest(items))
```

---

<!-- .slide: class="mindmap-small" -->

```mermaid
mindmap
    root((can_access<br/>artifact:1))
        (owner)
            ((user))
        (can_access from app)
            (member from org)
                ((user))
                ((admin))
                ((manager))
        (member from org)
            ((user))
            ((admin))
            ((manager))
```
```mermaid
mindmap
    root((can_access<br/>artifact:2))
        (owner)
            ((user))
        (can_access from app)
            (member from org)
                ((user))
                ((admin))
                ((manager))
        (member from org)
            ((user))
            ((admin))
            ((manager))
```
```mermaid
mindmap
    root((can_access<br/>artifact:3))
        (owner)
            ((user))
        (can_access from app)
            (member from org)
                ((user))
                ((admin))
                ((manager))
        (member from org)
            ((user))
            ((admin))
            ((manager))
```

Note:
Happening 100 times.

---

#### Check Resolution

> To resolve a relation, a check opens a connection and holds that connection open while it dispatches child checks to resolve the nested relations.

Note:
Each check therefore holds several connections at once (its own cursor plus every child it's waiting on).

---

3. Parents are stuck waiting on child checks that can't get a connection

Note:
Since the pool is maxed out, child checks can't get a connection 

So parents are stuck holding a connection while waiting on a child

---

4. The pool deadlocks

Note:
At this point, checks fail since they time out by exceeding the request deadline.

---

#### Healthchecks

5. Healthchecks fail

    - Healthcheck attempts to ping the DB
    - Connections are maxed out

Note:
In the meantime --

---

6. Kubernetes kills the unhealthy pods and restarts it

Note:
This causes...

---

<div style="text-align: center; font-size: 3em">🔄</div>

Note:
The cycle repeats

---

### Now

---

#### Infra

- Right-sized the DB: `t4g.small` → `t4g.medium`
- Raised max DB connections per pod

Note:
Larger DB instance gives us more connections, allowing us to raise the max db conns config.

---

#### Config

- Stable pool: Set min idle / open connections
- Capped concurrent checks
- Enabled caching

```bash
DATASTORE_MIN_OPEN_CONNS
DATASTORE_MIN_IDLE_CONNS

MAX_CONCURRENT_READS_FOR_CHECK
MAX_CONCURRENT_CHECKS_PER_BATCH_CHECK

CHECK_QUERY_CACHE_ENABLED
CHECK_ITERATOR_CACHE_ENABLED
LIST_OBJECTS_ITERATOR_CACHE_ENABLED
```

Note:
Basically, follow FGA best practices for production config.

The pool settings just keep connections warm.

---

#### Model

- App access was re-checked for every object in a batch check
- Factored out into a new relation that skips the app subtree

```diff
 type artifact
   relations
     define app: [app]
     define org: [org]
     define owner: [user]
-    define can_access: owner and can_access from app and member from org
+    define can_access_within_app: owner and member from org
```

Note:
Say a batch check contains 100 items, we would check the same app access 100 times.

---

#### Fan-Out: Optimized

```mermaid
mindmap
    root((can_access_within_app<br/>artifact:1))
        (owner)
            ((user))
        (member from org)
            ((user))
            ((admin))
            ((manager))
```

---

Instead, check app access once per request

```python
async def can_access_artifacts():
    await client.check(
        ClientCheckRequest(
            user="user:john",
            relation="can_access",
            object="app:hub",
        )
    )

    await client.batch_check(
        ClientBatchCheckRequest(
            [
                ClientBatchCheckItem(
                    user="user:john",
                    relation="can_access_within_app",
                    object=f"artifact:{id}",
                )
            ]
        )
    )
```

Note:
We **still** do a 1-time check if you can access the app, instead of checking it for every artifact. Then batch check the artifacts with the new relation.

**And of course**, for any other repeated check branches across batch checks, the cache will absorb them.

---

#### Minimal Reproduction

https://github.com/leahein/fga-max-conns

---

#### Results

- Lower check latency
- Fewer DB connections per check / batch check
- Cache absorbing repeat checks

---

## 2. List & Search -> Search & Batch Check

Avoid list operations in FGA

Note:
The next thing we learned, in a bit less of an exciting way, is that

**List** is expensive.

---

### Then

1. List objects in FGA
2. Filter in the app database

Note:
Initially, to display all user artifacts, we listed the objects in FGA first, because the total number of user artifacts is low.

---

List results are truncated

<img class="illo" src="assets/buggy-code.svg" alt="" />

Note:
But as the user's artifacts grew, we ran into list limits and artifacts would be truncated.
- **list deadline limit**
- **max results limits**

Had to work around them.

---

### Now

1. **First** filter on data in the app database to narrow down objects

<!-- .slide: class="er-large" -->

```mermaid
erDiagram
    ARTIFACT {
        uuid user_id
        uuid client_id
        enum visibility "private | public"
    }
```

Note:
This narrows down the number of objects.


- It's not enough to filter on the user's artifacts.
- Because an artifact can also be shared with everyone working on the client.
- So the visibility of whether it's public / private is stored in the app database.
- We query for both the user's artifacts and the public artifacts for the client.

---

2. **Then** batch check access against OpenFGA

---

Use database pagination to limit the number of checks per batch checks

Note:
To further narrow it down...

---

### Results

- No management of FGA list limits
- Pagination keeps the batch check size manageable

Note:
The tradeoff to this approach is that we tie the app database model to what the user has access to, but...

Infinite scroll has no fixed page count, so post-check drops are a non-issue.

---

## 3. 1 Model -> Modular Model

Separate modules per app

Note:
Finally, the last way in which we evolved and set ourselves up for expansion.
Is that as the app grew...

---

### Then

A single monolithic app and OpenFGA model

Note:
Originally we started out with 1 big model. 

---

Multiple services and teams sharing 1 model

<img class="illo" src="assets/image-files.svg" alt="" />

Note:
But as the app grew, we started to develop Hub across multiple sub-apps and the model became difficult to manage across teams.

---

### Now

- Each app has a corresponding FGA **module**

- A change to any module rolls out to every FGA-consuming app

Note:
The model is split into modules, one per sub-app. 

Since they compose into a single model, a change to any module changes the model, so they must then be rolled out to every app.

---

```mermaid
flowchart TB
    Edit(["Edit an FGA module"]) --> Test(["Run all tests"]) --> CICD(["CI / CD"])

    CICD -- "apply" --> FGA[("OpenFGA")]
    FGA -. "model ID" .-> CICD
    CICD == "pin model ID" ==> Apps

    subgraph Apps ["Hub apps"]
        direction LR
        App1["app 1"] ~~~ App2["app 2"] ~~~ App3["app 3"]
    end

    classDef step fill:#1a2430,stroke:#00d4aa,stroke-width:2px,color:#ffffff;
    classDef store fill:#3b3f2b,stroke:#76ea55,stroke-width:2px,color:#ffffff;
    classDef app fill:#101820,stroke:#edff00,stroke-width:1.5px,color:#d5d5d7;

    class Edit,Test step;
    class FGA store;
    class App1,App2,App3 app;
    style CICD fill:#1a2430,stroke:#00d4aa,stroke-width:1px,color:#edff00;
    style Apps fill:#1a2430,stroke:#515f6a,stroke-width:1px,color:#edff00;
```

Note:
- When an update is made to an FGA module,
- We first run tests against all apps to make sure a change doesn't break any app.
- On deploy, the pipeline applies the model against the live FGA service.
- FGA versions the model and the script outputs the new model ID.
- CI grabs that ID and pushes the model ID out to all apps.

---

### Results

- Each app manages its own FGA domain
- A change in one module gets tested against all apps
- The new model is rolled out to all apps

---

## Takeaways

1. **Control Fan-out**
2. **Avoid List Operations**
3. **Use Modules**

Note:
- Control fan-out with your config and model design
- Avoid list operations when you can
- Use modules to separate concerns

Overall, these improvements and patterns have gotten us to a stable and maintainable authorization system...for now.

---

## Q&A

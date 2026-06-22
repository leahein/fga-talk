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
- In our core system, we used a PBAC (Policy-Based Access Control) model, similar to AWS IAM, which worked well in silo-ing out data per client.
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

- New deployment of model

---

### Background


---

## Scaling


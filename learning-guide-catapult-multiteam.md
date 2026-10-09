# Learning path for the implementation problem

Self-study guide. Foundational concepts, books with specific chapters, and a sandbox project scoped to the routing system, player ID storage, and webhook lifecycle.

---

## Part 1 — Core .NET concepts

### Dependency Injection (DI)
The routing rule, secret resolver, and mapping store are wired together through DI. You need to be comfortable reading `builder.Services.AddScoped<…>` registrations and tracing which implementation is injected where.

### async / await and Task
The owner rule runs N `team_ids` filter calls in parallel with `Task.WhenAll`. Understanding the difference between `await task` and fire-and-forget, and why you *never* `.Result` inside an async call, is essential for the fan-out.

### Lambda handlers and SQS
Webhooks arrive as SQS messages that trigger a Lambda function. The handler must process a batch, acknowledge individual messages, and handle partial failures — getting this wrong causes infinite retries or message loss.

### MongoDB with the C# driver
HP stores sessions and athlete records in MongoDB. You need to read filter expressions, projection, and the upsert pattern (`ReplaceOneAsync` with `IsUpsert = true`) that's used everywhere in HPI.

### Conditional writes (optimistic concurrency)
The mapping record needs a conditional write (`insertIfNotExists` / ETag) to prevent two simultaneous team-connect requests from creating duplicate webhook registrations. This is the mechanism behind the idempotency guarantee in §2.4.

---

## Part 2 — Systems design concepts

### Idempotency
A webhook may be delivered more than once. The dedup row exists so that processing the same activity twice produces one session, not two. The deleted-at flag is an extension of this — idempotency must survive a session being deleted.

### Reference counting
The webhook deregistration rule — remove webhooks only when the last team disconnects — is a reference count. Understanding why naïve "remove when this team disconnects" breaks the other teams is the conceptual foundation of §2.4.

### Fan-out and ownership
One event (an activity webhook) must be routed to one or more downstream consumers (Hudl teams). The design choice — one owner vs. fan-out — has cascading effects on deduplication, device files, and the visibility gap.

### Sticky state / dedup rows
Once the owner is recorded, later webhooks for the same activity use it without re-running the rule. Understanding how sticky state interacts with reconnects and deletes prevents subtle bugs where the owner changes mid-activity.

### Eventual consistency and retries
SQS delivers messages at least once. The DLQ exists for messages that fail all retries. Knowing how to write a handler that is safe to retry (no double-charges, no duplicate rows) is fundamental to the whole ingestion path.

---

## Part 3 — Recommended reading

Specific chapters, not whole books. Read in order.

### C# in Depth, 4th edition — Jon Skeet
**Start:** Ch. 1–2 (C# as a language), Ch. 5 (async/await internals — the mental model matters more than the mechanics), Ch. 14 (nullable reference types, which the codebase uses). Skip the rest until you need it.

### Designing Data-Intensive Applications — Martin Kleppmann
**Start:** Ch. 1 (reliability, scalability, maintainability), Ch. 7 §"Write Skew and Phantoms" (why conditional writes matter), Ch. 11 §"Event Sourcing" and §"Idempotent Writes" (SQS-style at-least-once delivery).

### AWS Lambda in Action — Danilo Poccia
**Start:** Ch. 2 (the Lambda execution model), Ch. 8 (SQS trigger, partial batch failures, and the difference between visibility timeout and message deletion).

### Unit Testing Principles, Practices, and Patterns — Vladimir Khorikov
**Start:** Ch. 4 (the four attributes of a good test — especially "resistance to refactoring"), Ch. 8 (integration vs. unit), Ch. 3 §"output-based testing" (directly applicable to the owner rule).

### MongoDB: The Definitive Guide, 3rd edition — Bradshaw, Brazil, Chodorow
**Start:** Ch. 4 (querying), Ch. 5 (indexing — important for the `(catapultCustomerId, catapultTeamId)` compound index), Ch. 9 §"findAndModify" (atomic read-modify-write behind conditional writes and upserts).

---

## Part 4 — Sandbox project

Build this before touching the production codebase. It covers every mechanism in the proposal at a small, controllable scale.

**Mini webhook router:** a small .NET worker service that receives simulated "activity" events from a local queue, applies an ownership rule, and writes one "session" document per activity to a local MongoDB instance — using Docker for the infrastructure.

### Steps

1. **Scaffolding** — `dotnet new worker`. Add `AWSSDK.SQS` and `MongoDB.Driver` NuGet packages. Run a local MongoDB and a local SQS-compatible queue (ElasticMQ or LocalStack) in Docker.

2. **Message handler** — write an SQS consumer that pulls messages, processes them, and deletes each one on success. Introduce a deliberate failure path and verify the message returns to the queue rather than being lost.

3. **Mapping table** — create a MongoDB collection with the schema `{ teamId, customerId, catapultTeamId, connectedAt }`. Write a method to insert a mapping and a method to find all mappings for a given `customerId`. Add a compound index on `(customerId, catapultTeamId)` and verify it's used.

4. **Owner rule** — given a list of candidate team IDs and a hardcoded participant list (each participant tagged with one or more team IDs), implement the ownership function: exclusive count → total count → earliest `connectedAt`. Write it as a pure function so it's easy to test without the database.

5. **Dedup row with sticky owner** — write a `sessions` collection. On each incoming event, upsert the session: insert if no row exists (setting the owner), skip the owner update if one already exists. Verify idempotency by sending the same event twice.

6. **Deleted-at flag** — add a `deletedAt` field to the session document. Implement a "delete" path that sets it. Verify that a subsequent update for the same activity does *not* clear the flag or recreate the session.

7. **Reference-counted webhooks** — simulate webhook registration as a counter in MongoDB. Connecting a team increments; disconnecting decrements. Only call "deregister" (log it) when the count reaches zero. Test concurrent connects with `Task.WhenAll`.

8. **Per-account player IDs** — change the athlete document's `catpi` field from a single string to a dictionary `{ customerId → athleteId }`. Write a merge function that adds the new entry without overwriting existing ones. Verify a player connected to two accounts retains both IDs.

### Why this covers what you need

Each step maps directly to a section of the proposal: steps 1–2 are §2.5, steps 3–4 are §2.1–2.2, steps 5–6 are §2.3, step 7 is §2.4, step 8 is §2.6. By the time all eight steps work and are tested, you will have implemented every load-bearing mechanism once in a safe environment — with real infrastructure, not mocks — before touching the production codebase.

---

*Assumes basic familiarity with C# syntax and running `dotnet test` but no prior knowledge of distributed systems or MongoDB.*

# Prep Notes — 9/22 Event Store & Messaging Working Session

Decision notes and talking points grounded in this repo's current design. Related
analysis: [`docs/event-ordering.md`](./event-ordering.md) and
[`docs/architecture.md`](./architecture.md).

---

## 1. The dual-write problem in the current design

The command handler ([`src/commands/handler.ts`](../src/commands/handler.ts)) performs a
**dual write** — two independent writes to two systems with no shared transaction:

```ts
await persistEvent(domainEvent);     // write 1: DynamoDB (event store)
await publishToKinesis(domainEvent); // write 2: Kinesis (broadcast)
```

Failure modes:

- Write 1 succeeds, Lambda crashes before write 2 → event stored but **never
  broadcast**; the read model and downstream consumers silently diverge from the truth.
- Write 2 succeeds, write 1 fails/rolls back → an event was **published that never
  happened**; consumers act on a lie.
- Wrong order or retry → double-publish.

DynamoDB and Kinesis cannot be wrapped in one atomic transaction. This is the gap that
the **outbox pattern** exists to close, and it is agenda item §3 ("Durable Write +
Reliable Broadcast").

---

## 2. Outbox pattern — summary

Write the outgoing message **into your own database, in the same transaction as the
business data**, then let a separate relay publish it.

```
┌─────────── one atomic DB transaction ───────────┐
│  INSERT event row                                │
│  INSERT row into "outbox" (same commit)          │
└──────────────────────────────────────────────────┘
                     │  (after commit, separately)
                     ▼
         Relay reads outbox → publishes to broker → marks sent
```

- Both inserts commit or roll back together → no window where the business change exists
  without its outgoing message recorded.
- Draining the outbox: **polling publisher** or **transaction-log tailing / CDC**
  (Debezium on Postgres WAL, or **DynamoDB Streams**).
- Delivery is **at-least-once**, not exactly-once → **consumers must be idempotent**
  (e.g. track last-applied `version`, skip duplicates — same fix noted in
  `event-ordering.md`).

**2PC vs. outbox:** these are alternatives, not synonyms. The outbox pattern exists
specifically to avoid distributed 2PC between a database and a broker, which is
impractical to do reliably.

### Why this matters for the AWS-native path

For DynamoDB, the event-store table **is** the outbox and a change-data feed **is** the
relay — the outbox effect comes essentially for free, no separate outbox table. The
redundant `publishToKinesis` call is the fragile dual-write; the correct shape is "write
to DynamoDB only, let the change-data feed fan out."

| | Outbox mechanism | Relay / broadcast |
|---|---|---|
| DynamoDB | the event table itself | change-data feed (see §3) |
| Aurora / Postgres | dedicated outbox table **or** WAL | polling worker **or** logical replication / Debezium |

---

## 3. DynamoDB → Kinesis: two independent change-data features

**Key clarification:** a DynamoDB table can emit its changes two completely different
ways. They are **separate, independent features** — the Kinesis integration does **not**
use or depend on DynamoDB Streams. You can enable either, both, or neither on the same
table, and running both at once is supported and non-interfering.

### Feature A — DynamoDB Streams

- DynamoDB-managed stream with its own API; consumed via Lambda event source mappings or
  the KCL adapter.
- **This is what the projector uses today**
  (`addEventSource(new DynamoEventSource(...))` in
  [`infra/stacks/regional-stack.ts`](../infra/stacks/regional-stack.ts)).
- ~24h retention. **Strict ordering per partition key.** No duplicates within the stream.

### Feature B — Kinesis Data Streams integration (`KinesisStreamSpecification`)

- You attach a Kinesis Data Stream of your own to the table; DynamoDB writes change
  records directly into it through a separate internal path.
- **Does not use DynamoDB Streams.** It is not "Streams piped into Kinesis."
- Consume with any standard Kinesis tooling.
- **Ordering is NOT guaranteed per key**, and delivery is **at-least-once** (records may
  be duplicated).

### Guarantee comparison

| Feature | Uses DynamoDB Streams? | Ordering | Duplicates | Best for |
|---|---|---|---|---|
| DynamoDB Streams (→ Lambda/KCL) | yes (it *is* that) | strict per partition key | no | projections, replay, anything order-sensitive |
| Kinesis integration (`KinesisStreamSpecification`) | **no — independent** | **not guaranteed** per key | at-least-once possible | fan-out, analytics, archival, multi-consumer |

---

## 4. Recommendation for this system

- **Write to DynamoDB only.** Remove the application-level `publishToKinesis`
  dual-write from the command handler.
- **Keep the projector (read-model builder) on DynamoDB Streams** — it is order-sensitive
  and must see events v1 → v2 → v3 per `aggregateId`. Do NOT move it to the Kinesis
  integration; that would reintroduce ordering bugs. Also harden it to be
  reorder/duplicate tolerant (track last-applied `version`).
- **Use the native Kinesis integration for unordered fan-out** — downstream analytics,
  other bounded contexts, archival to S3, independent consumer apps that carry their own
  idempotency/sequencing (sequence on the event's own `version`, not arrival order).
- Both features run on the `LoanEvents` table simultaneously; each is used for what it
  actually guarantees.

### Options ranked (getting events from DynamoDB into Kinesis)

1. **Native DynamoDB → Kinesis integration** — zero custom code, no dual-write, but
   at-least-once and unordered. Best for fan-out/analytics.
2. **DynamoDB Streams → Lambda → Kinesis** — a small relay Lambda reads the ordered
   stream and republishes; you control ordering/batching but own the code (explicit
   outbox relay).
3. **Current app-level dual-write** — keep only as the "what not to do" example.

---

## 5. Open items to raise in the session

- **Schema evolution / event versioning** — cuts across platform-vs-blueprint, Aurora,
  and DynamoDB; not currently called out in the agenda.
- **Snapshotting** — periodic aggregate snapshots to bound replay cost on both Aurora and
  DynamoDB event logs.
- **Cross-region writes** — DynamoDB Global Tables are last-writer-wins; same-version
  concurrent writes in two regions can clobber (no cross-region transaction). See
  `event-ordering.md` §3.
- **Events belonging to multiple streams** — DDB's single partition-key model makes this
  awkward; decide between GSIs, fan-out, or EventBridge.
- **Decision outputs per deep dive** — capture what the session decided, not just what was
  discussed.

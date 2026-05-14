# Race Conditions Are Not Race Conditions

*Concurrency is the cheapest way to wedge a state machine apart. The window is never the bug.*

---

"Race condition" is the most successful misnomer in web security. It suggests a timing problem. It suggests that if you could just slow the attacker down, the system would be fine. It suggests the fix lives in the network stack, or in queueing, or in some retry policy.

It does not.

The bug is never the window. The bug is that the system made a decision against state it did not hold the lock on, and then committed to an outcome based on a snapshot that something else had quietly invalidated. Concurrency does not create that gap. It only walks through it.

That is the whole frame.

Anything you call a race condition should, on review, decompose into a **decision context** and a **commit context** — and the bug is that those two contexts are allowed to disagree. Sometimes by milliseconds. Sometimes by minutes. Sometimes by an entire downstream reconciliation cycle.

The most useful question is not "is this endpoint racey." It is:

**Where does this system decide something based on state it does not hold the lock on, and which subsystem is going to ship the consequence before the lock catches up?**

That is the actual bug class.

---

## Where the field is right now

James Kettle's *Smashing the State Machine* settled the arrival problem in 2023. Single-packet attack let researchers deliver N requests with sub-millisecond dispersion at the server, which closed the question of *whether* concurrency was achievable against a modern stack. After that, the conversation should have moved on. It mostly didn't.

What it moved to instead was a slightly larger pile of "I hit this endpoint hard and money fell out" writeups. Useful as proof points. Not useful as a frame. Arrival was the cheap half of the problem. **Verification** — the question of which decisions survive concurrency and which ones quietly fall apart — is where the real surface lives, and it isn't a single technique. It is a category of architectural mistake that shows up in at least four meaningfully different shapes.

This is a writeup of those shapes, illustrated through four bugs I reported and was paid for across 2026. Targets and exact paths are removed. The point is the shape, not the trophy.

---

## Shape 1: the gap inside a single handler

This is the textbook version. A SQL `count(*)`, an application-level check, an `INSERT`. No transaction wrapping the pair. No `SELECT ... FOR UPDATE` on the parent row. No advisory lock keyed on the resource. No declarative constraint enforcing the cap.

On a SaaS platform's organization-invite endpoint, the cap was six invites per org. The handler counted invites, compared to the cap, inserted. Three statements. No lock held across them.

The fun part: I never needed Turbo Intruder.

The platform's own frontend used a batching RPC client — one HTTP request carrying N independent mutation invocations. The batching layer was a convenience feature for the SPA. The server processed each invocation in parallel under the same auth context. **Nine invites slipped past a cap of six in a single HTTP request.** The tenth came back with a Postgres `Semaphore timed out`, which is the database telling you the lock primitive exists and is fighting for the row — but the application code is not the one holding it.

Two things matter about this case beyond the cap bypass.

First, **the concurrency primitive was the client SDK.** The attacker didn't bring concurrency. The platform shipped it. tRPC, GraphQL, and JSON-RPC batching all have this property: the frontend is a concurrency engine, and the server's per-HTTP-request rate-limit is meaningless when the rate-limited unit is the request envelope rather than the mutation. This is post-Kettle terrain. Single-packet attack solved arrival from a *hostile* client. Batching solved arrival from the *blessed* client, and nobody on the server side updated their threat model.

Second, **there was no revocation endpoint.** The product exposed no way to delete a pending invite. Every successful invite — racy or not — became permanent state on the organization, with operator recourse limited to "talk to support." A race becomes catastrophic exactly when the commit is irreversible. The bug is not just "I exceeded the cap." It is "I exceeded the cap and the product has no UI to put me back." Missing revocation surfaces are their own bug class, and they multiply every concurrency mistake near them.

Root cause sentence: the application encoded a limit as a procedure (`count, check, insert`) instead of as a declaration (constraint, advisory lock, partial unique index). The database had the lock. The application never picked it up.

Triaged and rewarded.

---

## Shape 2: the gap between two subsystems

This one is the genuinely interesting case, and the one most writeups never reach.

A fintech product offered a tiered fee-coverage feature: pay a small monthly fee, get a small buffer against overdrafts; pay a slightly larger fee, get a larger buffer. The plan-change endpoint took a single parameter — desired tier — and was supposed to perform two operations atomically:

1. Charge the new fee.
2. Activate the new tier.

The system did not consider these one operation. They were two subsystems. Billing collected fees on a nightly schedule with a documented retry-and-NSF policy. Entitlement flipped the tier immediately on the plan-change call. The two were stitched together by an assumption — *if billing fails, entitlement will roll back* — that the architecture had never been built to enforce.

I sent concurrent plan-change requests against the higher tier from an account that didn't have the balance to cover even one clean fee.

What happened next is the part to read carefully.

- **Entitlement flipped to the higher tier on the first successful response.** Optimistically. Before any fee had durably settled.
- **Billing fired multiple fee debits** — one per raced request — against a balance that couldn't support even one of them.
- **A reconciliation job later issued refunds** for the duplicate debits. The refund job had no concept of entitlement. It unwound dollar amounts and went home.
- **Final state: the higher tier was still active.** The account was over its declared limit by a margin that the limit was explicitly designed to prevent — `Used > Limit` persisted across days. The fee ledger showed N debits and N partial refunds, leaving a net charge that did not correspond to any rational pricing path.

That is not a race condition in the traditional sense. The window between the raced requests was small, but the **real** gap was between two reconciliation systems that didn't share state and didn't share authority. Billing could refund money. Billing could not revoke a tier. Entitlement granted a tier. Entitlement did not watch billing.

Anywhere a benefit, a quota, a tier, a permission, or a credential is granted in one subsystem and paid for in another, you have this shape latent in the architecture. Concurrency is the easiest way to surface it because concurrency forces the two subsystems to disagree visibly within a single reconciliation cycle. But the underlying bug — *grant is not atomic with payment* — exists at any throughput.

A sibling of this exact pattern appeared earlier in a non-race form: when a composite checkout finalized one leg of a bundle while the other failed, the survivor kept pricing that was only valid while the bundle existed. Partial failure was the wedge there. Concurrency was the wedge here. Same gap. Same lesson: a system that grants something based on a fee path it does not hold across the grant will, eventually, hand out the something without the fee path.

Triaged and rewarded at High.

---

## Shape 3: the gap between an undeclared invariant and the data model

A money-routing feature on the same fintech let a user register an email alias as an inbound-payment recipient. The intended invariant — stated nowhere in the contract — was that an account could have one pending alias registration at a time. The database did not encode this. The application did not encode this. It was a vibes constraint.

A single-packet attack with four distinct aliases produced four distinct pending registrations, each with its own internal reference id, each generating its own verification email. The "remove one, another quietly appears" behavior was the tell: pending state was a list, not a slot. The model the database held had never matched the model the product team thought they had.

This is a different shape from the invite cap. In the invite case, the invariant was *declared* (`cap = 6`) and *enforced procedurally*. Here, the invariant was *never declared at all* — no rule visible anywhere in the application about pending uniqueness. The race didn't violate a stated constraint. It revealed that no such constraint existed.

That distinction matters during review. When you find a feature whose UI implies "one of these at a time" — one pending invite, one active session, one default payment method, one verified email, one anything — the question to ask is whether that "one" is encoded in the schema or asserted only in the UI. Schemas don't have feelings. UIs do.

The fix is a partial unique index or an `ON CONFLICT` clause that the original developer should have written.

Triaged and rewarded.

---

## Shape 4: the gap between two systems

The same fintech offered an eSIM trial through an upstream provider. The user-facing flow let an account with a free-trial entitlement create one eSIM order. The backend forwarded the order to the provider; the provider provisioned the eSIM.

Concurrent replay of the order-creation request produced multiple `202 Accepted` responses and multiple provisioned eSIMs against a single trial entitlement. The user account balance never reflected multiple successful charges. Either the platform or its upstream was absorbing the cost.

This is the shape of a race where the two ends of the gap are **not even in the same company.** The platform owned the order endpoint. The provider owned the fulfillment. The trial-eligibility check on the platform side ran once per request; the provider's side had no concept of "trial," only of "incoming order with valid auth." Neither side enforced single-use on the trial context, because each assumed the other would.

This is the most operationally expensive shape. Every duplicated good has a real fulfillment cost. Phone numbers, eSIM data plans, shipped items, API credits, paid third-party tokens — anywhere your platform provisions through an upstream and the trial/limit logic lives only on your side, you have this latent. Concurrency turns one paid trigger into N paid triggers, and the bill lands either on you or on the provider. Often the provider, which means the bug only surfaces in someone else's quarterly report.

The fix is idempotency keys that the upstream actually honors, plus a single-use consumption marker on the trial context that flips atomically *before* fulfillment is even attempted. Consume first. Fulfill second. The order of those two operations is the entire bug.

Triaged and rewarded.

---

## What these four have in common, and what they don't

The common piece is the gap. In each case, a decision was made against a state that something else was about to change.

- In Shape 1, the gap was **between two SQL statements.**
- In Shape 2, the gap was **between two subsystems with independent reconciliation.**
- In Shape 3, the gap was **between the UI's model and the database's.**
- In Shape 4, the gap was **between two companies' systems with no shared single-use token.**

What they don't share is the size of the gap. Shape 1 closes in milliseconds with a single `BEGIN; ... FOR UPDATE; ... COMMIT;`. Shape 4 may not close without renegotiating the contract with the upstream provider. **The fix surface scales with how far apart the decision and the commit live.**

That scaling is the part the original race-condition literature underweights. "Add a lock" is fine prescription for Shape 1. It is the wrong advice for Shape 2. It is structurally impossible advice for Shape 4. The deeper a race lives in the architecture, the less it looks like a concurrency bug and the more it looks like a contract bug between parts of the system that never agreed on who owns the truth.

---

## The 2026 story: client-driven concurrency

The piece of this nobody is saying out loud:

**Modern frontends are concurrency engines.** tRPC batching, GraphQL operation batching, JSON-RPC array calls — every one of these takes N logical operations and delivers them in a single HTTP request that the server then fans out internally. The convenience justification is bandwidth and round-trip latency. The security consequence is that the *server's own client SDK ships the attacker a free concurrency primitive*, and every per-HTTP-request rate-limit, every per-request audit log, every per-request middleware that assumed one operation per envelope becomes a single point of inadvertent serialization bypass.

You don't need a single-packet attack against a batching backend. You need a single packet, period. **The batching is the attack.**

This is the part of the threat model most teams have not updated. If your stack uses tRPC, GraphQL, or any batching RPC layer, every mutation in your API is implicitly callable N-at-a-time under the same auth context, on the same TCP connection, dispatched to the same worker pool, in the same logical instant. Procedural limit enforcement was already a bad idea. Procedural limit enforcement in front of a batching client is procedural limit enforcement with the safety off.

Two defenses worth thinking about, neither of which is "rate-limit harder":

1. Serialize mutations *within* a batch on the server, or reject batches containing multiple invocations of the same mutation against the same parent resource. Batching as a client-side ergonomics feature should not become a server-side concurrency primitive.
2. Move every limit, cap, and uniqueness rule into a declarative database constraint or an advisory lock keyed on the parent resource. The database is the only thing in the stack that actually understands "one at a time."

---

## What a hardened state machine actually believes

Restating the frame as invariants:

- **Decisions and commits live in the same transaction**, or the decision is treated as advisory and re-checked at commit. There is no third option. "We check on the way in and trust the check by the time we write" is the bug.
- **Limits are declared, not enforced procedurally.** A cap is a unique partial index, a `CHECK` constraint, an advisory lock — not a `count(*)` followed by an `if`. The database is allowed to fight for the row. The application is not allowed to assume it won.
- **Grants are atomic with the conditions that justified them.** A tier granted because a fee was charged is revoked if the fee is not, in the same transaction, durable. Two subsystems that reconcile separately cannot make joint decisions safely.
- **Single-use contexts are flipped to consumed before fulfillment.** If the trial token marks itself spent only after the eSIM provisions, the eSIM can be provisioned N times. The order is wrong. Consume first. Fulfill second. If fulfillment fails, refund the consumption — never the other direction.
- **Every commit has a revocation surface.** If an invite, a session, a grant, a provisioned good, or a registered alias can be created, it must be deletable. Race bugs become irreversible exactly to the extent that the product cannot undo what the race produced. The missing revocation endpoint is the multiplier on every other bug in the same handler.

---

## The closing question

Race condition is the wrong name. It is shorthand for a class of bugs in which the system made a decision under one state and committed to it under another, and concurrency is simply the easiest way to make those two states disagree visibly.

The single-packet attack settled arrival. Batching RPCs delivered free concurrency to anyone with a browser. What is left is the question that was always the real one:

**Where does this system decide something based on state it does not hold the lock on, and which subsystem is going to ship the consequence before the lock catches up?**

If the answer is "the application code, between two SQL statements," you have a Shape 1.

If the answer is "billing decides one thing and entitlement decides another," you have a Shape 2.

If the answer is "the UI thinks there is one of these and the schema doesn't agree," you have a Shape 3.

If the answer is "our system and theirs," you have a Shape 4.

If the answer is "we never thought about it that way," you have all of them, and you have not yet found the worst one.

That is the entire bug class. The window is not the bug. The window is just where the bug walks through.

---

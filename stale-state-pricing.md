# When a Composite Checkout Keeps Pricing From a State That No Longer Exists

Most payment bugs people look for are obvious: changing a price, reusing a discount, forcing a negative total, or finding a race condition.

This one was different.

The issue was not about directly modifying amounts. It was about breaking a transaction invariant in a bundled checkout flow.

The platform allowed a user to start with Transaction A and then attach Transaction B in the same checkout session. When the combined value of A + B crossed a threshold, the backend applied more favorable pricing to A.

That means A was not independently entitled to that pricing. Its discount existed only because B existed too.

So the rule should have been simple:

> Transaction A may keep bundle-derived pricing only while the state (A + B) still exists.

That rule was not enforced.

---

## Modeling the flow

Here is the simplest way to model it.

### Initial standalone state

```
Transaction A
  base amount:     7.75
  standalone fee:  4.49
  standalone total: 12.24
```

By itself, A does not qualify for reduced pricing.

Now the user attaches Transaction B:

```
Transaction B
  base amount: 8.25
```

Now the combined value becomes:

```
A + B = 16.00
```

At that point, the platform marks A as eligible for better pricing because the bundled threshold has been met. So the pricing changes from:

```
A = 7.75 + 4.49 fee
```

to something like:

```
A = 7.75 + 0.00 adjusted fee
```

That is still valid at this stage, because the backend's assumption is:

> A is discounted because B exists.

---

## Where it breaks

The problem starts during payment.

Suppose the payment source has enough balance to cover A in its discounted state, but not enough to cover the full bundled total. Then the authorization flow becomes:

```
auth(A) = success
auth(B) = failure
```

This is the exact point where the backend has to choose a safe model. There are only two correct options.

### Option 1: atomic processing

If pricing depends on A + B, then both transactions must succeed together. If B fails, nothing should finalize.

### Option 2: revalidation on partial failure

If the system allows A to survive while B fails, then it must:

- Remove B from the transaction state
- Recalculate A as a standalone transaction
- Invalidate the bundle-derived pricing
- Require authorization for the corrected total

What the vulnerable flow did instead was this:

- A succeeded
- B failed
- A finalized anyway
- A kept pricing that was only valid while B existed

That is the bug.

The backend computed pricing using the state `(A + B)`, but finalized settlement using the state `(A)`, without rechecking whether A was still entitled to bundle-derived pricing.

---

## State-machine view

A clean state-machine view makes the issue obvious:

**S0 — Standalone**
- Only A exists
- A does not qualify for reduced pricing

**S1 — Composite eligible**
- B is added
- A + B crosses the threshold
- Reduced pricing is applied to A

**S2 — Authorization**
- `auth(A) = success`
- `auth(B) = failure`

**S3 — Vulnerable finalization**
- B no longer survives
- A is finalized with pricing inherited from S1 instead of being repriced under S0

The broken transition is **S2 → S3**.

Once B failed, the system should have moved back to standalone pricing logic for A. It did not. It carried forward stale eligibility from a state that no longer existed.

---

## Why this is a security bug, not a billing bug

That is why this is not just a billing inconsistency. It is a state-machine integrity bug.

The backend encoded real economic rules:

- Threshold-based fee waivers
- Bundle-dependent incentives
- Pricing derived from linked transaction state

Those rules only work if the backend recomputes truth when the qualifying state changes. Here, it did not.

A user could intentionally create a valid composite state, let the platform apply the incentive, then force one leg of the composite to fail and still keep the favorable pricing on the surviving leg.

That is a direct failure of order-calculation integrity.

What makes the bug stronger is that it was practical. It did not require packet timing, request replay, client mutation, or race windows. It relied on a normal backend event: partial authorization failure in a multi-leg checkout.

That matters because payment failure is part of the security boundary. If a platform supports linked transactions and bundle-derived pricing, then partial failure handling has to be security-correct, not just payment-correct.

---

## The deeper mistake

The deeper mistake was treating eligibility as a sticky flag instead of a derived condition.

A lot of systems effectively do this:

1. Threshold reached
2. Flag set
3. Benefit applied

That only works when the qualifying condition remains true.

In this case, the flag was created because B existed. Once B failed, the flag should have died with it. Instead, A kept a pricing property that was no longer justified by the current transaction state.

That is the entire finding. Not because a number was changed manually. Not because the frontend was tricked. But because the backend finalized an outcome based on a stale state snapshot.

---

## The fix

The fix is straightforward in design terms. Once B fails, the system must do one of the following before finalizing anything:

- Cancel the entire composite checkout
- Revert A to a pre-finalization state
- Reprice A as a standalone transaction
- Remove all bundle-derived incentives
- Require fresh authorization for the corrected amount

The invariant should be:

> A surviving transaction must never retain incentives whose validity depended on a bundle that no longer exists.

---

## Why this gets missed

This kind of bug gets missed because most payment testing is too local. People ask whether they can change a number, reuse a code, or break the total. The better question is:

> What happens when one subsystem computes value from assumptions that another subsystem later invalidates?

That is what happened here.

The pricing subsystem assumed:

> A is discounted because B exists.

The authorization subsystem later proved:

> B does not survive.

The finalization path never reconciled those two facts. That was the vulnerability.

---

## Final thought

This was not a flashy bug. No exploit chain, no timing trick, no weird payload. Just a backend that calculated pricing in one state and finalized in another.

And that is exactly why it matters — and what made me **money**.

Some of the best bugs are not hidden in requests. They are hidden in the moment a system quietly stops enforcing the condition that made its earlier decision valid.

---

*#bugbountytips #cybersecurity*

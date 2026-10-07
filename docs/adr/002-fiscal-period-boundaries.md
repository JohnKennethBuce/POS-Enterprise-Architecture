# ADR-002: Fiscal Period Boundaries & the Lifetime Accumulator Invariant

## Status
Accepted

## Date
2026-10-06

## Context

Philippine fiscal regulations governing computerized sales systems require
every register to maintain two distinct financial records, and operators
routinely conflate them:

- **A period reading** — a snapshot of sales within a defined window. An
  **X-reading** is non-destructive (audit and shift-change use); a **Z-reading**
  is destructive (fiscal close). A Z-reading seals the period, advances a
  non-resettable sequential counter, and commits the period total to the
  permanent ledger.
- **A lifetime accumulator** — a monotonically increasing odometer of gross
  sales since the register entered service. It is **never reset** during
  daily, monthly, or annual closings. Resetting it on an active terminal is a
  severe compliance violation.

Three integration problems surfaced in multi-shift and overnight operations:

1. **Shift-window omissions.** The period filter selected orders by their
   *creation* timestamp. In hospitality environments — where a table opens
   before midnight and settles after — the transaction fell outside the
   subsequent shift's window entirely. It appeared in neither reading.

2. **Apparent accumulator discrepancy.** After a Z-reading closed a period,
   an operator generating the next morning's reading saw `Sales: ₱0.00` beside
   a nonzero accumulator, and reported it as a defect. It was not a defect —
   the operator was comparing a *period* figure against a *lifetime* figure.
   But the confusion was real and recurred across tenants.

3. **Sequence format drift.** When a period contained no transactions, the
   beginning and ending invoice numbers fell back to a generic zero-padded
   form rather than preserving the register's terminal-prefixed sequence.
   This broke the visual continuity auditors expect across consecutive
   readings.

## Decision

### 1. Settlement-time period inclusion

An order belongs to a fiscal period if **either** of the following is true:

- A payment against it was tendered within the period, **or**
- The order record was last updated within the period.

The period is defined by the window in which money changed hands — not the
window in which a cart was first opened. This is the only definition that
survives overnight dining, split shifts, and late-night settlement without
omitting or double-counting a transaction.

### 2. Lifetime accumulator invariant
accumulator_new = accumulator_previous + period_net_sales

text

The accumulator is a **non-resettable odometer**. A period with zero sales
correctly reports `₱0.00` for the period while preserving the accumulator's
lifetime value unchanged. No code path — including period close, tenant
reconfiguration, or administrative override — may decrement or zero the
accumulator on a live register.

Brownfield adoption (migration from a legacy register under an existing permit)
is supported through a one-time **first-time base seeding** operation,
restricted to the platform operator role. Store-level staff cannot alter the
base, the base date, or the historical counter values.

### 3. Sequence continuity across empty periods

When a period contains no transactions, the beginning and ending invoice
numbers carry forward from the preceding reading's ending number. If no prior
reading exists, they fall back to the register's terminal-prefixed first
sequence. The generic zero-padded fallback is removed — a sequence that
abruptly changes format across a period boundary is a red flag to an auditor,
even when the underlying data is correct.

### 4. Environment provisioning discipline

Fiscal immutability triggers are provisioned by the database engine, not by
application-level entity lifecycle rules. Test-environment reseeding uses
native truncation rather than application-level cascade deletion, because
cascade deletion is silently blocked by the same immutability triggers that
protect production. This distinction is captured in environment configuration
so that a test run cannot be mistaken for a production run.

## Alternatives Considered

| Alternative | Rejected because |
| :--- | :--- |
| **Filter period by order creation timestamp** | Drops overnight and split-shift transactions from the shift in which they were actually paid. |
| **Filter by settlement timestamp only** | Misses late-edit corrections (void adjustments, tip additions) that don't create a new payment row. |
| **Reset accumulator at Z-reading** | Direct compliance violation. The accumulator is a lifetime odometer, not a period total. |
| **Surface period total without accumulator** | Operators already conflate the two; hiding the accumulator removes the audit signal entirely. |
| **Reject empty-period readings** | Operators still need a printed record that a period closed with zero sales — it's part of the audit chain. |
| **Preserve generic SI fallback for empty periods** | Introduces format discontinuity that auditors flag as suspicious, even when the data is correct. |

## Consequences

### Positive
- Every settled transaction is attributed to exactly one fiscal period.
- The accumulator behaves as a true lifetime odometer, correctly surviving zero-sales periods.
- Invoice sequence format is continuous across empty periods, matching auditor expectations.
- Fiscal immutability is enforced by the engine, so test-environment reseeding cannot silently bypass it.

### Negative / Trade-offs
- The period inclusion predicate is broader than a simple timestamp filter and requires an indexed query plan to stay fast at scale.
- Operators require onboarding material explaining the difference between a period total and a lifetime accumulator. This is a training cost, not an engineering one.
- First-time base seeding introduces an administrative operation that must be gated carefully to prevent tampering.

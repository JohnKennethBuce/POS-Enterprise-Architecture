# ADR-001: Statutory Dining Discounts & the Single-Portion Rule

## Status
Accepted

## Date
2026-10-06

## Context

Philippine tax law grants statutory discounts and VAT exemptions to several
beneficiary classes. Each class carries a **different** legal basis, rate, and
VAT treatment — and those differences are not cosmetic. Treating them as
interchangeable produces incorrect fiscal output.

| Beneficiary class | Legal basis | Discount | VAT treatment on beneficiary portion |
| :--- | :--- | :-: | :--- |
| Senior Citizens | Expanded Senior Citizens Act | 20 % | **VAT-exempt** |
| Persons with Disability | PWD welfare statute | 20 % | **VAT-exempt** |
| Solo Parents | Expanded Solo Parents Welfare Act | 10 % | **VAT-exempt** |
| National Athletes & Coaches | National athletes incentive statute | 20 % | **VAT still payable** |
| PEZA-registered / Diplomatic | NIRC §106(A)(2) | — | **Zero-rated (0 %)** |

The statutes grant the discount for the *personal and exclusive consumption*
of the qualified beneficiary. The platform operationalizes this as a
**single-portion rule**: one beneficiary, one meal. Additional portions on the
same line are billed at regular VATable rates.

Three problems surfaced during integration testing:

1. **Quantity inflation.** Applying a statutory discount across an entire line
   item of quantity `Q` over-credits the beneficiary. Under audit, the excess
   VAT exemption is a tax deficiency attributable to the merchant.

2. **Cross-beneficiary aggregation.** Beneficiary reporting was aggregating
   discount totals at the **order** level rather than the **line-item** level.
   A single ticket containing a Senior item and a Solo Parent item would have
   its entire discount total absorbed into whichever beneficiary book was
   generated first. The resulting statutory books overstated one class and
   omitted the other.

3. **Silent offline fallback.** When a checkout payload failed server-side
   deserialization — typically due to a missing discount field or an
   unrecognized discount type — the client silently caught the error and fell
   back to offline mode. A receipt printed, but the order never persisted and
   stock never depleted. This is a fiscal integrity failure: the customer holds
   a document the server has no record of.

## Decision

### 1. Single-portion mathematical invariant

For any item-scoped statutory discount applied to a line of quantity `Q ≥ 1`,
exactly **one** portion receives statutory treatment. The remaining `Q - 1`
portions are billed at full price with standard VAT.

For VAT-exempt classes (Senior, PWD, Solo Parent):
Exempt base = round(unit price ÷ 1.12, 2)
Unit discount = round(exempt base × statutory rate, 2)
Beneficiary owes = exempt base − unit discount
Remaining owes = unit price × (Q − 1)
Line total = beneficiary owes + remaining owes

text

For VAT-payable classes (National Athletes & Coaches), the discounted portion
**remains VATable** — the discount reduces the VAT-exclusive base, but output
VAT is computed on the discounted amount and added back:
Discounted net = round(unit price ÷ 1.12, 2) × (1 − 0.20)
Output VAT = round(discounted net × 0.12, 2)
Beneficiary owes = discounted net + output VAT

text

Rounding is applied at each monetary step and never deferred — deferred
rounding produces centavo drift that fails BIR reconciliation.

### 2. Discount scope selection at settlement

The cashier selects discount scope at the point of tender, choosing between
**entire bill** (a group discount) and **single portion** (a beneficiary
discount). When single-portion is selected, the interface heuristically
preselects the highest-value eligible line, which the cashier can override.

Zero-rated transactions require the operator to capture the supporting
exemption credential (PEZA registration or diplomatic certificate) before the
transaction can settle.

### 3. Line-item segregation in statutory reporting

Beneficiary books and statutory report generation aggregate discount and VAT
exemption amounts by **matching line item**, not by order. A ticket
containing mixed beneficiary classes contributes only its matching lines to
each beneficiary register. This eliminates cross-beneficiary over-aggregation
structurally — not by convention.

### 4. Uniform discount representation

All discount types — statutory and promotional — are normalized into a single
internal representation at the API boundary. Unrecognized discount types are
mapped to a generic category rather than rejected, so a client-side addition
of a new promotional type can never silently trigger the offline fallback
described above.

## Alternatives Considered

| Alternative | Rejected because |
| :--- | :--- |
| **Apply discount to entire line quantity** | Violates the personal-consumption reading of the statutes and creates audit exposure on every multi-quantity beneficiary sale. |
| **Aggregate discounts at the order root** | The failure mode that produced this ADR. Cannot correctly attribute discounts to beneficiary classes on mixed tickets. |
| **Defer rounding to the order total** | Produces centavo-level discrepancies that fail BIR reconciliation. Rounding must be deterministic per line. |
| **Reject unknown discount types server-side** | Causes client/server version skew to break checkout entirely. Normalizing to a generic category preserves forward compatibility. |
| **Treat National Athletes identically to Senior/PWD** | The statutes differ on VAT treatment. Uniform handling would under-remit output VAT. |

## Consequences

### Positive
- Statutory discounts are correctly bounded to a single portion per beneficiary.
- Beneficiary books report accurate, class-attributed totals — a prerequisite for BIR audit.
- Unknown discount types can no longer silently divert a checkout to offline mode.
- The same statutory math applies identically whether a sale originates online or is replayed from the offline queue.

### Negative / Trade-offs
- Line-item payloads are larger and carry more computational work per order.
- The cashier must actively choose discount scope; the platform cannot infer it.
- The single-portion rule requires a staff-facing explanation, since it may surprise merchants accustomed to discounting whole orders.

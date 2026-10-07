# ADR-003: Three-Tier Inventory Depletion Modes & the Stock Movement Ledger

## Status
Accepted

## Date
2026-10-06

## Context

The platform serves merchants whose inventory models span a wide spectrum —
from a single packaged SKU to a multi-stage production recipe with yields and
waste factors. Three distinct depletion models were identified in practice:

1. **Standalone SKU** — a finished good sold as-is. Depletes its own stock.
2. **Hybrid** — a mixed catalog where some items are prepared from ingredients
   and others are retailed as finished goods.
3. **Multi-tier bill of materials** — every item decomposes into raw materials
   with a yield factor and a waste allowance.

Three problems were observed:

1. **Silent mode gap.** The hybrid mode existed in the data model but not in
   the settlement path. Merchants operating in hybrid mode received no
   ingredient depletion at all — a defect that only surfaced when a merchant
   reconciled physical stock against system stock.

2. **Semantic confusion between physical and theoretical stock.**
   Merchants expected to see *what physically left the warehouse*. Analysts
   expected to see *what the recipe says should have been consumed*. These are
   two different numbers — variance between them is a genuine operational
   signal — but presenting them in a single view made both unreadable.

3. **No store-level audit surface.** Movement auditing existed at the platform
   operator level but not at the merchant level. Store managers could not
   answer "who moved this stock, when, and why" without escalating to support.

## Decision

### 1. Explicit depletion dispatch per line item

The settlement path resolves inventory depletion **per line item**, dispatching
on the merchant's configured mode:

- **Mode 1** — direct decrement of the finished good.
- **Mode 2** — attempt recipe resolution. If an active recipe exists, deplete
  its components; otherwise fall back to direct finished-good decrement. The
  fallback is per-item, not per-tenant, so a single merchant can carry both
  prepared and packaged goods.
- **Mode 3** — full recipe explosion. Every line depletes its components
  scaled by quantity and waste factor.

Recipe-scaled consumption:
consumed = quantity_required × (1 + waste_percentage ÷ 100) × units_sold

text

Depletion executes **synchronously at settlement**, not as a background job.
An asynchronously depleted order is an order whose stock figure is briefly
wrong — and briefly wrong is indistinguishable from wrong to the next
customer attempting to buy the last unit.

### 2. Two-track inventory architecture

Physical movement and theoretical costing are deliberately separated:

- **Physical track.** Records stock that actually left the warehouse. Written
  at settlement as an immutable ledger entry classified by movement type.
- **Theoretical track.** Reconstructs what the recipe model *expects* to have
  been consumed, valued at current weighted unit cost. Computed on demand;
  never persisted as a parallel truth.

Variance between the two tracks is itself the operational signal merchants
want — it is the difference between what the kitchen used and what the recipe
says it should have used.

### 3. Immutable movement ledger with typed classifications

Every stock movement — regardless of origin — is written as an immutable
entry with one of four classifications:

| Classification | Origin |
| :--- | :--- |
| `SALE_DEDUCTION` | POS settlement |
| `PURCHASE_RECEIPT` | Warehouse intake |
| `WASTE_SPOILAGE` | Kitchen spoilage / write-off |
| `MANUAL_ADJUSTMENT` | Authorized correction |

Each entry carries a timestamp, item identity, signed delta, running balance,
unit of measure, and (where applicable) a reference to the originating
document. Entries are append-only.

### 4. Store-scoped ledger visibility

A tenant-scoped ledger view exposes movement history to the merchant directly,
behind the same permission gate as reporting. Store managers can answer stock
questions without escalating to platform support. The platform-operator audit
view remains separate and broader.

## Alternatives Considered

| Alternative | Rejected because |
| :--- | :--- |
| **Tenant-level mode flag only (no per-item fallback)** | Forces hybrid merchants to either fake a recipe for packaged goods or lose depletion on one category. |
| **Asynchronous depletion via background worker** | Introduces a window where stock is stale — unacceptable when the next customer is checking availability. |
| **Single combined ledger (physical + theoretical)** | The two numbers serve different purposes; combining them makes both unreadable and destroys variance as a signal. |
| **Computed physical stock (no ledger)** | Removes the audit trail. Physical stock must be reconstructible from an immutable event log, not merely derived. |
| **Merchant-level ledger view gated to platform operators only** | Adds support load for a question ("who moved this?") the merchant should be able to answer themselves. |
| **Mutable ledger entries** | A stock ledger that can be edited is not an audit trail. Immutability is the point. |

## Consequences

### Positive
- All three inventory models are supported by the same settlement path.
- Physical depletion and theoretical costing are cleanly separated, making variance a first-class operational metric.
- Every stock movement is auditable by the merchant without platform support.
- Recipe-scaled consumption correctly accounts for waste, so multi-tier BOM merchants no longer under-deplete.

### Negative / Trade-offs
- Synchronous depletion adds latency to settlement — mitigated by bulk ledger writes.
- Two-track presentation requires the merchant to understand the difference between physical stock and theoretical usage. Onboarding material is a real cost.
- Per-item mode dispatch is more complex than a tenant-wide flag.

# Architecture Decision Records

This directory holds the historical log of significant architectural decisions
made while designing the platform. Each record captures the **context**, the
**decision**, the **alternatives considered**, and the **consequences**.

ADRs are immutable. When a decision is superseded, a new record is written and
the older one is marked `Superseded by ADR-XXX` rather than edited in place.

## Index

| ID | Title | Status | Date |
| :--- | :--- | :--- | :--- |
| [ADR-001](./001-statutory-discount-single-portion.md) | Statutory Dining Discounts & the Single-Portion Rule | Accepted | 2026-10-06 |
| [ADR-002](./002-fiscal-period-boundaries.md) | Fiscal Period Boundaries & the Lifetime Accumulator Invariant | Accepted | 2026-10-06 |
| [ADR-003](./003-inventory-depletion-modes.md) | Three-Tier Inventory Depletion Modes & the Stock Movement Ledger | Accepted | 2026-10-06 |
| [ADR-004](./004-tenant-isolation.md) | Multi-Tenant Isolation Across Transport, Authorization & Real-Time Channels | Accepted | 2026-10-06 |

## Why These Records Are Public

These four decisions shaped the system's statutory correctness, its data
integrity guarantees, and its tenant isolation model. They are published as
design rationale — **not** as implementation guidance. Code, schema, and
deployment configuration are not included here by design.

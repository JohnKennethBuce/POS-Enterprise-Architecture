# ADR-004: Multi-Tenant Isolation Across Transport, Authorization & Real-Time Channels

## Status
Accepted

## Date
2026-10-06

## Context

The platform hosts many independent merchants on a shared persistence layer.
Tenant isolation is therefore not a feature — it is a **correctness property**.
Isolation must hold at every layer where a tenant identifier can be asserted:

1. **The request pipeline**, where an authenticated session asserts a tenant claim.
2. **The authorization layer**, where that claim is matched against the resource being accessed.
3. **The real-time transport layer**, where sockets subscribe to tenant-scoped broadcast groups.
4. **The administrative surface**, where platform-operator routes must never be reachable without proper authentication.

Three integration problems surfaced during adversarial review:

1. **Cross-tenant real-time subscription.** The real-time hub admitted any
   authenticated socket into any requested tenant group without verifying the
   socket's tenant claim against the requested group. A session belonging to
   Tenant A could subscribe to Tenant B's live order and kitchen channels,
   eavesdropping on ticket contents, customer names, and sales totals. This is
   a textbook instance of an object-reference authorization failure — the
   socket was authorized as *a user*, but never authorized as *this tenant's user*.

2. **Unauthenticated administrative probes.** Administrative routes were
   guarded at the handler level but not at the pipeline perimeter. A request
   without valid credentials reached the routing layer before being rejected —
   unnecessary attack surface, and a defense-in-depth gap.

3. **Frontend credential key divergence.** A subset of the client's download
   and import flows read credentials from a different storage key than the
   rest of the application. Those flows silently returned `401` and left the
   user with a failed download and no explanation. The root cause was
   inconsistency, not a security flaw — but the resulting silent failure was
   itself a reliability problem worth fixing.

## Decision

### 1. Claim-to-context verification at the real-time boundary

Real-time subscription requests are rejected unless the requesting socket's
authenticated tenant claim matches the tenant group being joined. Platform
operator sessions are the sole exception, and that exception is explicit and
auditable.

Rejected subscription attempts are logged with the requesting tenant, the
target tenant, and a timestamp — because an attempt to cross a tenant boundary
is itself a security event, whether or not the platform permits it.

### 2. Perimeter rejection of unauthenticated administrative traffic

Administrative route prefixes are guarded at the **pipeline perimeter**, before
routing dispatch. Requests without valid credentials are rejected with a
`401` and a machine-readable error code, and never reach handler logic. This
is deliberately redundant with handler-level authorization: defense-in-depth
means each layer assumes the layer before it failed.

### 3. Single credential-source convention on the client

Every client flow — interactive, modal, download, and import — reads
credentials from one canonical location, with a backwards-compatible fallback
to the legacy key for sessions that predate the change. The convention is
documented once and applied uniformly. Silent `401`s on file download are
eliminated by construction.

## Alternatives Considered

| Alternative | Rejected because |
| :--- | :--- |
| **Trust the socket's authenticated identity alone** | The socket is authenticated as a user; it is not yet authorized as a member of *that* tenant. Conflating the two is the entire class of failure this ADR exists to prevent. |
| **Verify tenant claim in the hub method body (not the join)** | Checking at join time prevents the subscription from ever existing. Checking per-message is a higher-cost per-event guard for the same result. |
| **Reject unauthenticated admin traffic at the handler** | Leaves an unauthenticated request traveling through the pipeline before rejection. Perimeter guards are cheaper and reduce attack surface. |
| **Enforce frontend credential convention via code review** | Conventions that rely on discipline drift. A single canonical accessor is enforceable and self-documenting. |
| **Silently coerce mismatched tenant IDs to the caller's own tenant** | Silent coercion hides cross-tenant attempts from the security log. An attempt to reach another tenant is a signal; the platform should reject it loudly, not quietly redirect it. |

## Consequences

### Positive
- Cross-tenant eavesdropping through real-time channels is eliminated structurally, not by convention.
- Administrative surfaces are unreachable without valid credentials, even briefly.
- Cross-tenant attempts are logged as security events for forensic review.
- File download and import flows no longer silently fail on stale session keys.

### Negative / Trade-offs
- Real-time subscription joins carry a small additional authentication cost.
- The pipeline-order dependency means a future middleware inserted above the perimeter guard could re-expose unauthenticated traffic if the ordering constraint is not documented. This ADR is that documentation.
- The backwards-compatible credential fallback is temporary and must be removed once legacy sessions age out.

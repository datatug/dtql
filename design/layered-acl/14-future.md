# Bounded future compatibility notes

## Long-running logical transactions / change sets

Out of scope: no persistence, distributed ACID, locking design, conflict resolution, recovery, approvals or transaction protocol here.

Today's request/result shape need not prevent future staged operations. Stable operation IDs and per-operation blockers can represent multiple blocked operations; policy/identity revisions distinguish a historical assessment from current authority. Staging authorization must remain advisory: a future commit would need authorization against current effective policies and current principal/membership, not a replayed allow from stage time. Persist references to subject/actor and the context needed to re-resolve identity, not reusable bearer tokens or frozen role grants. Policy changes before commit may invalidate staged results. `where/check` already separates existing-row admission and candidate post-image constraints. No field in the MVP should claim that a dry-run result is a durable authorization lease.

No further long-running transaction design is necessary to ship this ACL slice.

## External/custom ACL provider

Existing DALgo `Policy` is already the application extension. C4 separates Describe/Snapshot, read/edit and Inspector so a future service may supply decisions/restrictions while retaining owner/source/revision attribution and remaining read-only in DataTug. Unknown/unavailable required providers fail closed and explicit capability negotiation prevents pretending opaque code can be serialized or inspected. Request IDs, bounded operations, stable reason codes and redaction apply equally.

MVP implements only local application policies and current owner adapters. A remote ACL service would later need its own authenticated trust/availability/consistency and resource-budget review; no service, deployment, caching protocol or cross-source transaction architecture is added now.

# Security analysis and administration

## Assets, adversaries and boundaries

Protect row/column contents and existence where private; policy text/literals, source locations and membership; policy integrity; mutation integrity; credentials; and correct negative decisions. Adversaries include authenticated low-privilege users, malicious browser requests, overprivileged application credentials and malformed provider responses. Trusted components are owner authenticators/resolvers, DALgo evaluator, secured adapter code and OS/storage administration. A hostile filesystem owner, raw database credential holder or malicious code in the same process is not contained by a DALgo wrapper.

## Diagnostic disclosure profiles

| Requester authority | Permitted default information |
|---|---|
| Unauthenticated | Authentication failure only, no topology/policies/rows. |
| Ordinary authenticated data caller | Overall outcome, own requested action/columns where safe, generic coalesced `ACCESS_DENIED`, correlation ID, disclosable restrictions only. No hidden row existence or count. Self plan/inspect does not confer policy-read privilege. |
| Owner policy reader/admin | Authorized owner policy text and references; data read and enhanced lower diagnostics are separate privileges. Admin of OpenVaultDB is not admin of InGitDB. |
| Authorized Explain caller | Decisions for authorized target principal/resources; actual target membership resolution; only each owner's allowed diagnostic projection. Predicate literals still require policy-read/disclosure permission. |
| Trusted internal component | Minimum evidence/decisions granted by owner integration. Trusting a component to compose denials does not imply authority to download rows/policies. Raw pre/post images stay in the enforcing owner unless explicitly readable. |

Separate owner grants: `policies:discover/list/read/create/update/delete`, `access:explain`, `access:simulate`, `access:diagnostics` (public policy/rule references; no protected-row existence or predicate facts), owner-scoped, concrete-key-bounded `access:inspect-protected`, and out-of-band ACL configuration administration. Capabilities may be database/owner scoped, with no cross-owner wildcard delegation in MVP. Data grants do not grant policy administration. Role/group editing is an identity administration operation, outside data policy CRUD.

User-directed privacy rule overrides the general disclosure table: new/imported policies default `visibility:public`. Only explicit owner-scoped `policies:admin` can disclose **private** policy identity/text/conditions or change classification. Public policy reading still requires ordinary owner read authorization. A policy reader, editor or Explain caller without that grant cannot see a private policy, even with reference diagnostics. Private custom/application policy descriptors also remain hidden. Generic outcomes/opaque restriction obligations still reach non-admins; never reveal their count. Existing public policies may be edited by delegated editors with the requisite CRUD rights; those editors cannot declassify policies. Revalidate visibility per request and invalidate UI/server caches after visibility changes.

Public response projection happens after internal reduction. Redaction cannot turn deny/unknown into allow. Do not expose number of redacted blockers, whether a hidden key exists, sensitive predicates, resolved variables, absolute paths, provider response bodies or stack traces. Conditions may contain sensitive literals even without parameters. If private or too large, return opaque restrictions per C2. Missing and denied paths should have indistinguishable public shapes and policy topology exposure. Exact constant-time behavior is not promised; rate-limit inspection and avoid obvious query-count/timing oracles.

## Bootstrap and administration

Provision an owner-management principal and grant through authenticated local configuration/CLI before enabling ACL. Validate a complete policy set and at least one recoverable owner manager. Management ACL resides in the existing capability/control-plane store, not a data policy that can recursively lock its own validation. Initial fixture manager credentials are not ordinary data credentials. Emergency repair requires explicitly logged local owner control; it does not introduce a public bypass token for data or lower policies.

Create/update/delete must authorize against the **current** owner's management grants, validate the whole candidate set and perform compare-and-swap. A policy document cannot grant its author management authority or change its own owner/target. Prevent removal of the last required policy/management route; failed validation leaves current snapshot. Reject lower-layer replacement via upper store. Required layer topology is configured server-side. No “ignore unavailable layer” retry control is exposed.

## Threat/control/test matrix

| Threat | Required control | Acceptance |
|---|---|---|
| Principal/tenant spoofing | Verified subject+actor, trusted variable namespace, current membership | T12, T13 |
| Upper allow bypasses lower deny | Mandatory independent owner layers, secured constructor | T02, T03 |
| Stop-first denial hides other safe blockers | Shared bounded inspection; static blockers continue | T07, T08 |
| Hidden row leaked by inspection/sample | Internal evidence, per-owner projection, requester∩subject readable sampling | T09, T18 |
| Hidden column filtered/sorted/aliased | Common AST reference validation before execution | T10 |
| Stale dry-run grant or remote pre-image | Re-evaluation plus owner-issued whole-row CAS inside transaction | T11 |
| Replacement deletes protected fields | Pre/post field-diff and subtree checks | T16 |
| Dropped context/alternate session | Bound DB mandatory policies, no Background context on ingress | T03, T13 |
| Policy tampering/CRUD races | Owner control-plane privileges, manifest, atomic publication, ETags | T05, T14 |
| YAML bombs/path traversal/duplicate keys | Strict grammar, finite limits, no aliases/tags/symlinks | T15 |
| Custom/provider failure or opaque native ACL | Explicit coverage and unsupported/error deny | T08, T17 |
| Trigger/schema side effects before denial | Final-candidate validation, no side effects in dry run, transactional write path | T08, T16 |
| Browser XSS/CSRF/credential theft | Text rendering, origin/session protection, no owner token in client | T12, UI suite |

Audit policy administration, identity changes and denied execution with subject/actor, owner, policy revision, safe codes and correlation ID. Do not log raw bearer tokens, row contents or all query parameter values. Enhanced audit access is privileged and retention is an operator configuration; a new audit delivery service is not MVP. Callback/custom policy side effects are not repeated by dry run.

Read filtering does not hide authorized aggregates that the profile does not support; aggregates are rejected until leakage semantics are designed. Row-level restrictions can leak via underlying uniqueness/foreign-key validation errors, schema discovery and generated fields; protected profiles sanitize these errors and limit metadata separately. Source-native ACL failures remain authoritative even when no structured reason is available.

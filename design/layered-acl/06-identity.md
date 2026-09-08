# Identity, authentication and principal propagation

Contract C3. Authentication establishes a subject and acting client; authorization uses trusted current membership and grants. DTQL is not a login, token issuance or identity federation protocol.

## Canonical identity

Honor approved OpenVaultDB decision 0004: hosted users have the verified shared Sneat Co. Firebase UID as canonical internal `userID`, in the configured Sneat identity realm. Do not add a second hosted OpenVaultDB user ID. This is an existing application account identity, not a Google/GitHub OAuth subject. External provider bindings remain managed by the existing identity infrastructure.

For self-hosted deployments without an established internal user key, use opaque generated stable IDs in a configured realm; store in the host identity directory, not a policy file. The portable principal reference is `(realm,kind,id)`. User, service, application and agent IDs do not share an untyped collision domain. Roles/groups likewise have stable owner-scoped IDs, not display names. Policies resolve binding keys in the database's configured realm. Cross-realm identities require explicit trusted resolver mapping; string equality of IDs is insufficient.

External binding storage conceptually maps `(OIDC issuer,subject)` or `(provider instance,subject)` uniquely to the internal account. One account can have several bindings; no email-address auto-linking, display-name matching or policy rewrites. Link/unlink requires authenticated account control and the existing provider's verification flow. A provider rename/configuration change must not orphan policy identities. Identity-directory secrets and OAuth tokens must not enter Git policy files.

## Reuse and MVP profile

DataTug already depends on shared Sneat/Firebase auth; hosted OpenVaultDB already specifies that same authority. Self-hosted OpenVaultDB has opaque bearer tokens, hashed persistence, expiry/revocation and application capabilities. Reuse these, extending the grant record with stable subject reference, registered actor/client reference and delegation scope. The existing client_id-only application grant must remain distinguishable from a user delegation. Owner tokens map to an explicit local bootstrap principal rather than an anonymous omnipotent human.

MVP E2E uses a configured self-hosted resolver and preprovisioned opaque tokens for stable fixture accounts, roles/groups and clients. This proves authenticated principal propagation without building OIDC issuance. Production hosted login remains the shared identity flow; wire compatibility and mapping tests cover it, but deploying cloud account linking is not an MVP prerequisite. No homemade JWT or new identity database in DALgo.

## Trusted context

Internal context fields: subject `{realm,kind,id}`, actor `{realm,kind,id}`, database/mount audience, granted token capabilities, role IDs, group IDs, trusted authorization attributes, identity/membership revision, authentication strength if available, and request ID. Bind after token verification. Deep-copy and treat as immutable for one short operation. Ordinary request JSON cannot set `roles`, `groups`, `currentUser`, `tenant`, `now` or path captures.

Roles/groups resolve from authoritative current state for the target realm/space. Long-lived tokens must not permanently snapshot membership. MVP fixture resolver reads current local state at each request; hosted resolver observes decision 0004's current-membership requirement. If a required resolver is unavailable, fail closed. Any cache must have explicit revocation semantics; do not invent an unspecified “short cache” as a guarantee. Principal revocation between dry run and execution is tested by authenticating anew.

Authorization is bounded by **both** actor capability/delegation and subject policies. A powerful subject does not widen a narrow application's scope; a privileged application does not inherit permission to act as any user. Application/service policies are additional restrictions, with distinct provenance. Role/group membership enables bound rule sets only after resolution; no additive cross-layer override.

## Hops

Browser → DataTug daemon: reuse authenticated daemon session/bootstrap protection and approved origins. A Firebase UI sign-in alone is not proof that a loopback daemon request has the right authority. Bind the session to the authenticated user and connection. Do not forward browser principal fields as facts; local CLI simulation flags remain local simulation.

DataTug → OpenVaultDB: use an existing OpenVaultDB-issued credential bound to registered DataTug actor and stable subject, target database and permitted operations. In MVP preprovision this binding through owner administration; no new token-exchange protocol. Do not forward an unrelated Firebase bearer token to a receiver that does not validate its issuer/audience. Never send the owner token as the ordinary query credential. Transport must use TLS outside loopback.

OpenVaultDB → embedded InGitDB: verified immutable Go context, with owner-configured realm mapping. No network authentication protocol is needed at this hop. Direct InGitDB Go callers supply a trusted local resolver; supplying arbitrary `access.WithPrincipal` from hostile code is outside the library boundary. Remote InGitDB access uses OpenVaultDB by product decision.

Other-principal Explain checks the **requester's** `access:explain` at each source, then resolves the target principal independently. Supplying custom memberships/attributes additionally requires `access:simulate` and marks all outputs hypothetical. Execution endpoints reject simulation fields. Audit records preserve both requester and simulated subject without leaking tokens/attributes. API/service identities can be evaluated identically using their kind and bindings, never impersonated as humans.

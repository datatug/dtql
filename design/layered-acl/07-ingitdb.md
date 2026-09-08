# InGitDB integration

User-confirmed product direction: OpenVaultDB serves remote InGitDB connections. InGitDB owns schema/data/policy storage and local adapter enforcement. This preserves the removed-server ADR and avoids two competing authentication and serving stacks.

## Ownership and storage

Add a validated root configuration reference for ACL mode and a policy manifest, proposed `.ingitdb/access/manifest.yaml`, with documents under `.ingitdb/access/policies/<policyId>.yaml`. These are schema/configuration metadata, never ordinary collection records. Root config explicitly declares `acl.enabled` and the manifest reference. Manifest records owner database ID, identity realm, expected policy IDs/files and active revision; no tokens. Exact config integration is WP4 and must fit the current `ingitdb/config/root_config.go` parser conventions.

Mode defaults disabled for old repositories only when no ACL configuration has ever been enabled. When enabled, an absent/corrupt manifest or referenced policy fails closed. Do not scan an empty directory and treat that as unrestricted. The server pins enabled configuration on mount so deleting the root file while running cannot disable enforcement; restart bootstrap must also use configured enabled requirements. Explicit mode changes require configuration-admin authority outside policy CRUD. A malicious repository/OS owner can change all files and is outside this ACL boundary.

Policy CRUD validates the entire candidate set (IDs, target, bindings, supported format) before publishing it. Use owner-issued strong revisions and expected-revision comparison, existing cooperating writer lock, atomic file replacement and one Git commit over the policy/config changes. A failed commit restores the prior active files and revision; successful response means disk/revision and active snapshot agree. Reconnect/restart reloads the committed policy state. Revision should be a manifest/content digest covering all active policy bytes and configuration, with Git commit as provenance, rather than assuming every data commit changes policy semantics.

Load only manifest-listed regular files beneath the fixed policy root. Reject symlinks/path traversal/duplicate canonical paths and policy IDs. Generic record endpoints, bulk exports, inferred schema, collection enumeration and DDL must not expose or mutate this metadata. No policy source URL fetches. Git history access is a separate repository privilege; runtime policies do not retroactively redact Git commits.

## Adapter integration

`ingitdb/dalgo2ingitdb.NewDatabase`/public opening route loads owner configuration before exposing a handle. Keep raw database state private and return a secured DALgo facade that retains schema/introspection capabilities through explicit forwarding. Never lose enforcement when callers open transactions or request an alternate reader interface. Direct trusted application constructors and OpenVaultDB mounts use the same owner loader. Existing ACL-disabled users retain legacy semantics.

Enforcement obtains pre-images inside the adapter's existing transaction lock without exposing them as public `get` permission. All record writes, including batch, evaluate owner policy before filesystem mutation. The final post-image must be validated after supported transformations. Policy changes use the same owner admission/barrier convention. Root-record whole revisions support remote conditional writes; check revisions inside the transaction before any mutation. For MVP one-key UPDATE is the real write; bounded multi-target acceptance proves no partial writes where batch is supported.

InGitDB implements policy provider/reader/manager/inspector independently of OpenVaultDB policy storage. OpenVaultDB may proxy owner operations but must call the owner authorization check with the actual requester. An OpenVaultDB admin is not automatically an InGitDB policy admin. InGitDB reports its own source ID and policy references even when called in-process.

Plan-only inspection reads no rows. Key inspection can read a private image and return a safe row ACL fact even if public get denies it, subject to diagnostic authorization. Sampling uses secured reads for candidate selection; no hidden-row enumeration. The Inspector invokes the same assessment functions as the secured operation.

## Acceptance and known boundaries

Open adapter directly and through OpenVaultDB with identical principal: same InGitDB decisions. Pass empty/new contexts after binding: mandatory DB policies remain. Tamper/miss manifest: deny, not disabled. Policy replace/reload/reconnect: same revision and behavior. Failed write: byte-identical records and no data Git commit. Failed policy edit: old snapshot persists. Direct file edits by noncooperating tools do not participate in locks; detect invalid revisions/reload conservatively but do not promise protection against the filesystem owner. Cloud GitHub adapter support is a later capability, not assumed from local filesystem support.

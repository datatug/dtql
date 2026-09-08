Verdict: **ready-with-changes**. The scoped-mask semantics are coherent, and I found no new authorization bypass in the specified algorithm. Resolve A3-01 before freezing canonical artifacts; A3-02 strengthens the conformance corpus.

Reviewed only the supplied 168,440-byte packet. Its SHA-256 matches `f2c80013ea78f91bc878ea5338081ce42f48bad2b8f4690430d0bc9cb8617980`.

**A3-01 — Medium: canonical YAML fixture contradicts the canonical quoting rule. Freeze blocker: yes, for canonical artifacts.**

- **Evidence:** `contracts/README.md` requires quoting ambiguous scalars “using YAML double quotes,” and requires serializers to match the canonical fixture bytes. Newly revised `contracts/mask-policy.canonical.yaml` instead contains single-quoted `'*c*'`, `'*d'`, and `'*'`.
- **Counterexample:** A serializer following the prose emits `- "*c*"`, while a serializer following the golden emits `- '*c*'`. Both produce the same AST and authorization decisions, but cannot satisfy the same byte-identical canonical contract. This matters because document revisions cover canonical bytes.
- **Action:** Change those fixture scalars to double quotes, or explicitly define a deterministic alternative quoting rule and regenerate the affected golden. Verify canonical bytes and second-serialization idempotence. The packet’s reported `yamlSemanticEquality:true` does not establish canonical byte agreement.

**A3-02 — Medium: delta fixtures leave important normalization and transport regressions insufficiently discriminated. Freeze blocker: no; address in WP1 conformance work.**

- **Evidence:** `scoped-mask-vectors.json` covers adjacent includes, one fourth-stage exclusion, repeated stars in `**d`, and cross-owner denials. It contains no adjacent-exclude example or dotted repeated-star example. `mask-restriction-result.json` carries only a two-stage field mask. T21 requires richer staged behavior, but the supplied concrete fixtures do not exercise these combinations.
- **Counterexamples:**
  - `include ["*"]; exclude ["a*"]; exclude ["b*"]; include ["*d"]` must group the excludes into one OR stage: `banana` denies and `bcd` restores. An evaluator processing raw adjacent excludes successively can incorrectly allow `banana`.
  - `address.**` must equal `address.*`, including legacy parent-prefix behavior. The audited parser strips a literal terminal `.*` **before** segment parsing. Extending only its segment matcher and normalizing stars only during serialization can therefore make `address.**` and its canonical form disagree on the concrete parent `address`.
  - A field restriction transported as `include ["address.*"]; exclude ["address.internal_*"]; include ["address.internal_public"]` must retain its restoration stage. A two-stage restriction golden cannot detect its loss.
- **Action:** Add focused declarative vectors for those cases; add `apple` remaining allowed under the fourth-stage exclusion example, proving that the final exclusion is scoped. Include canonical merge output for adjacent excludes and a three-stage `field_mask` result round-trip. These exercise the existing design without adding architecture.

The remaining delta is sound at specification level:

- Merging maximal same-action runs before evaluation correctly implements OR grouping. The first-nonmatch algorithm confines every later action to all preceding selected sets. The stated example and further alternation agree with that algorithm.
- Separate policies, owners and conservative query alternatives intersect completed staged decisions. Restoration cannot override an independently mandatory denial.
- The schema represents ordered stages for collection, field and procedure masks, rejects empty/first-exclude/mixed-key/obsolete flat mask structures, and preserves the complete `field_mask` restriction variant. Duplicate execution class/namespace rejection is expressly assigned to semantic validation; JSON Schema `uniqueItems` is not incorrectly presented as sufficient.
- Parent projection, complete evidence and scalar references remain explicitly distinct. Conservative parent-evidence rejection applies even when an excluded child is absent, and later restoration does not automatically establish complete-subtree authorization. Affected-descendant writes include removed and unchanged touched descendants. These are existing protections, not missing requirements.
- Disclosed masks must remain complete; private or oversized masks become opaque references. The packet expressly protects private cardinality and row facts.
- Reusing DALgo’s parsed masks, field-set intersection and enforcement call sites is appropriate. Its legacy parser remains separate; exclusion-bearing enumerable shortcuts are explicitly disabled in MVP.
- Variable-length syntax has finite advertised raw-input limits, byte checks, deadlines and explicit budget failures. Prefix evaluation cannot authorize on exhaustion. An additional semantic stage ceiling or glob-language inclusion solver is unnecessary.
- The ordered editor, owner validation/CAS, stale-client restrictions and future-field wildcard explanation fit the amended semantics. Work-package ownership and dependencies accommodate the change without introducing a separate mask engine or native execution.

Limits: this was a static review of the frozen packet and its audited source excerpt. I did not inspect the live repository, external sources or peer findings, edit files, run runtime ACL tests, or independently reproduce the packet’s recorded validation results. Implementation remains subject to human sign-off.

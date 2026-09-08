# Shared source evidence excerpts

Exact local revisions are in audit-snapshots.json. Line numbers name inspected source, not proposed changes.

## dal-go/dalgo/access/policy.go:25

```
25: 	Resources []Resource
26: 	// Query retains the structured or opaque query for custom policies and
27: 	// future predicate, projection, index, and cost constraints. It is nil for
28: 	// non-query operations; v1 declarative policies authorize query sources.
29: 	Query dal.Query
30: }
31: 
32: // Decision explains an access-policy result.
33: type Decision struct {
34: 	Allowed      bool
35: 	Operation    Operations
36: 	Resource     Resource
37: 	Policy       string
38: 	PolicySource string
39: 	Rule         string
40: 	Effect       string
41: 	Explanation  string
42: 	// Condition is the source text of the row condition behind a conditional
43: 	// decision, with parameter names rather than resolved values.
44: 	Condition string
45: 	// Residuals holds, per request resource (same index as Request.Resources),
46: 	// the resolved row condition the caller must still enforce before the
47: 	// operation is complete: after the read for point operations, by query
48: 	// rewrite for Query. A nil entry means the resource is allowed outright.
49: 	// Residuals is nil for an unconditional decision.
50: 	Residuals []dal.Condition
51: 	// Writes holds, per request resource, the ordered alternatives that decide
52: 	// a write on that resource (see WriteResidual). A nil entry means the write
53: 	// is allowed outright, with no row or post-image constraint.
54: 	Writes []*WriteResidual
55: }
56: 
57: // WriteAlternative is one allow rule as it applies to a write. Where selects
58: // the rows (by pre-image) the rule applies to; nil means every row. Check is
59: // what the row must satisfy after the write; nil means no constraint. Both are
60: // resolved conditions; the Text fields keep the source form with parameter
61: // names for explanations.
62: type WriteAlternative struct {
63: 	Rule      string
64: 	Where     dal.Condition
65: 	Check     dal.Condition
66: 	WhereText string
67: 	CheckText string
68: 	// Fields is the rule's allow-list of field patterns; nil means every field.
69: 	Fields []string
70: 	fields *fieldSet
71: }
72: 
73: // WriteResidual is what the secured wrapper enforces on a write: the
74: // conditional alternatives in precedence order, then Terminal — the first
75: // unconditional rule when it is an allow (possibly with a Check) — or nil when
76: // the walk ends in a deny or in no rule at all. The first alternative whose
77: // Where holds on the pre-image decides the write; a new row (Insert, or Set of
78: // a missing row) is admitted by the first alternative whose Check it satisfies.
79: type WriteResidual struct {
80: 	Alternatives []WriteAlternative
81: 	Terminal     *WriteAlternative
82: }
83: 
84: // DeniedError is returned when a policy rejects an operation.
85: type DeniedError struct {
86: 	Decision Decision
87: }
88: 
89: func (e *DeniedError) Error() string {
90: 	policy := fmt.Sprintf("policy=%q", e.Decision.Policy)
91: 	if e.Decision.PolicySource != "" {
92: 		policy += fmt.Sprintf(" source=%q", e.Decision.PolicySource)
93: 	}
94: 	rule := ""
95: 	if e.Decision.Rule != "" {
96: 		rule = fmt.Sprintf(" rule=%q", e.Decision.Rule)
97: 	}
98: 	return fmt.Sprintf("%v: %s%s operation=%s resource=%s: %s", ErrAccessDenied,
99: 		policy, rule, e.Decision.Operation, e.Decision.Resource, e.Decision.Explanation)
100: }
101: 
102: func (e *DeniedError) Unwrap() error { return ErrAccessDenied }
103: 
104: // Policy is a named access capability. Every Policy applied to a secured
105: // request must allow every target resource.
106: type Policy interface {
107: 	Name() string
108: 	Decide(context.Context, Request) Decision
109: 	Authorize(context.Context, Request) error
110: }
111: 
112: // AccessPolicy is the declarative hierarchical Policy implementation.
113: type AccessPolicy struct {
```

## dal-go/dalgo/access/policy.go:183

```
183: 				writes = make([]*WriteResidual, len(request.Resources))
184: 			}
185: 			writes[i] = last.Writes[0]
186: 		}
187: 	}
188: 	last.Residuals = residuals
189: 	last.Writes = writes
190: 	return last
191: }
192: 
193: func (p *AccessPolicy) decideResource(resolver variableResolver, operation Operations, resource Resource) Decision {
194: 	matching := matchingRules(p.compiled, operation, resource)
195: 	if len(matching) == 0 {
196: 		return Decision{
197: 			Operation:    operation,
198: 			Resource:     resource,
199: 			Policy:       p.name,
200: 			PolicySource: p.source,
201: 			Effect:       effectDeny.String(),
202: 			Explanation:  "no matching allow rule",
203: 		}
204: 	}
205: 	// Walk matches in precedence order: conditional allows met before the first
206: 	// unconditional rule apply to the rows their conditions select; the first
207: 	// unconditional rule settles everything else. An unconditional allow makes
208: 	// the collected conditions moot; a deny (or no unconditional rule at all)
209: 	// leaves their disjunction as the residual the caller must enforce.
210: 	var conditional []compiledRule
211: 	var terminal *compiledRule
212: 	for i := range matching {
213: 		if matching[i].where == nil {
214: 			terminal = &matching[i]
215: 			break
216: 		}
217: 		conditional = append(conditional, matching[i])
218: 	}
219: 	if len(conditional) == 0 && (terminal.effect != effectAllow || (terminal.check == nil && terminal.fields == nil)) {
220: 		allowed := terminal.effect == effectAllow
221: 		return Decision{
222: 			Allowed:      allowed,
223: 			Operation:    operation,
224: 			Resource:     resource,
225: 			Policy:       p.name,
226: 			PolicySource: p.source,
227: 			Rule:         terminal.name,
228: 			Effect:       terminal.effect.String(),
229: 			Explanation:  fmt.Sprintf("matched rule %q (%s)", terminal.name, terminal.effect),
230: 		}
231: 	}
232: 	// Resolve every alternative once; an unresolved parameter denies.
233: 	write := &WriteResidual{}
234: 	for _, rule := range conditional {
235: 		alternative, err := p.resolveAlternative(resolver.withCaptures(captureValues(rule.pattern, resource)), rule)
236: 		if err != nil {
237: 			return p.denyUnresolved(operation, resource, rule, err)
238: 		}
239: 		write.Alternatives = append(write.Alternatives, alternative)
240: 	}
241: 	if terminal != nil && terminal.effect == effectAllow {
242: 		alternative, err := p.resolveAlternative(resolver.withCaptures(captureValues(terminal.pattern, resource)), *terminal)
243: 		if err != nil {
244: 			return p.denyUnresolved(operation, resource, *terminal, err)
245: 		}
246: 		write.Terminal = &alternative
247: 	}
248: 	decision := Decision{
249: 		Allowed:      true,
250: 		Operation:    operation,
251: 		Resource:     resource,
252: 		Policy:       p.name,
253: 		PolicySource: p.source,
254: 		Effect:       effectAllow.String(),
255: 		Writes:       []*WriteResidual{write},
256: 	}
257: 	if len(conditional) == 0 {
258: 		// A terminal allow with only a post-image check or a fields list:
259: 		// reads are unconditional on rows.
260: 		decision.Rule = terminal.name
261: 		decision.Explanation = fmt.Sprintf("matched rule %q (allow)", terminal.name)
262: 		if terminal.check != nil {
263: 			decision.Condition = terminal.check.String()
264: 			decision.Explanation = fmt.Sprintf("matched rule %q (allow; check: %s)", terminal.name, decision.Condition)
265: 		}
266: 		return decision
267: 	}
268: 	names := make([]string, 0, len(conditional))
269: 	texts := make([]string, 0, len(conditional))
270: 	parts := make([]dal.Condition, 0, len(conditional))
271: 	for _, alternative := range write.Alternatives {
272: 		names = append(names, alternative.Rule)
273: 		texts = append(texts, alternative.WhereText)
274: 		parts = append(parts, alternative.Where)
275: 	}
276: 	if write.Terminal != nil {
277: 		// An unconditional allow makes the row conditions moot for reads.
278: 		decision.Rule = write.Terminal.Rule
279: 		decision.Explanation = fmt.Sprintf("matched rule %q (allow)", write.Terminal.Rule)
280: 		return decision
281: 	}
282: 	residual, text := parts[0], texts[0]
283: 	if len(parts) > 1 {
284: 		residual = dal.NewGroupCondition(dal.Or, parts...)
285: 		text = "(" + strings.Join(texts, " OR ") + ")"
286: 	}
287: 	decision.Rule = strings.Join(names, ", ")
288: 	decision.Condition = text
289: 	decision.Explanation = fmt.Sprintf("matched conditional rule(s) %s (where: %s)", quoteAll(names), text)
290: 	decision.Residuals = []dal.Condition{residual}
291: 	return decision
```

## dal-go/dalgo/access/policy.go:350

```
350: 	return &DeniedError{Decision: decision}
351: }
352: 
353: func matchingRules(rules []compiledRule, operation Operations, resource Resource) []compiledRule {
354: 	matches := make([]compiledRule, 0, len(rules))
355: 	for _, rule := range rules {
356: 		if !rule.operations.contains(operation) || !ruleMatchesResource(rule, resource) {
357: 			continue
358: 		}
359: 		matches = append(matches, rule)
360: 	}
361: 	sort.SliceStable(matches, func(i, j int) bool {
362: 		left, right := matches[i], matches[j]
363: 		if left.depth != right.depth {
364: 			return left.depth > right.depth
365: 		}
366: 		if left.literals != right.literals {
367: 			return left.literals > right.literals
368: 		}
369: 		if left.effect != right.effect {
370: 			return effectIsRestrictive(left.effect)
371: 		}
372: 		return left.name < right.name
373: 	})
374: 	return matches
375: }
```

## dal-go/dalgo/access/principal.go:10

```
10: 
11: // Principal identifies the caller for principal bindings and for the
12: // $currentUser, $principal.roles and $principal.groups variables. Roles and
13: // groups are opaque strings the host assigns; DALgo never looks them up.
14: type Principal struct {
15: 	ID     any
16: 	Roles  []string
17: 	Groups []string
18: }
19: 
20: type contextPrincipalKey struct{}
21: 
22: // WithPrincipal returns a child context carrying the caller's principal. It
23: // replaces any principal inherited from the parent context.
24: func WithPrincipal(ctx context.Context, principal Principal) context.Context {
25: 	if ctx == nil {
26: 		panic("access: nil context")
27: 	}
28: 	return context.WithValue(ctx, contextPrincipalKey{}, principal)
29: }
30: 
31: // PrincipalFrom returns the principal carried by ctx, if any.
32: func PrincipalFrom(ctx context.Context) (Principal, bool) {
33: 	if ctx == nil {
34: 		return Principal{}, false
35: 	}
36: 	principal, ok := ctx.Value(contextPrincipalKey{}).(Principal)
37: 	return principal, ok
38: }
39: 
40: // Bindings map principals to named rule sets. A binding value is a list of
41: // rule-set names; Everyone applies to every principal, including an absent one.
42: type Bindings struct {
43: 	Roles    map[string][]string
44: 	Groups   map[string][]string
45: 	Users    map[string][]string
46: 	Everyone []string
47: }
48: 
49: // PrincipalPolicySet is a Policy that derives the caller's authority from
50: // who they are: the rule sets bound to the principal's ID, roles and groups
51: // (plus Everyone) are unioned and compiled as one AccessPolicy, so the parent
52: // feature's precedence resolves overlaps between bindings; that single policy
53: // then takes part in the ordinary intersection with database and context
54: // policies, so a binding can never widen what another policy denies.
55: type PrincipalPolicySet struct {
56: 	name     string
57: 	source   string
58: 	ruleSets map[string][]Rule
59: 	bindings Bindings
60: 
61: 	mu    sync.Mutex
62: 	cache map[string]*AccessPolicy
63: }
64: 
65: // NewPrincipalPolicySet validates that every binding names an existing rule
```

## dal-go/dalgo/access/context.go:145

```
145: func (g guard) authorizeWrite(ctx context.Context, operation Operations, resources ...Resource) ([][]writeResidual, error) {
146: 	_, writes, err := g.authorizeRequest(ctx, Request{Operation: operation, Resources: resources})
147: 	return writes, err
148: }
149: 
150: // authorizeRequest evaluates every applicable policy and returns, per request
151: // resource, the read residuals and the write residuals the caller must still
152: // enforce. A denial by any policy is returned immediately.
153: func (g guard) authorizeRequest(ctx context.Context, request Request) ([][]residual, [][]writeResidual, error) {
154: 	dynamicPolicies := policiesFromContext(ctx)
155: 	contextPolicyCount := len(g.boundPolicies) + len(dynamicPolicies)
156: 	if err := g.requireContextPolicy(request.Operation, contextPolicyCount); err != nil {
157: 		return nil, nil, err
158: 	}
159: 	policies := make([]Policy, 0, len(g.databasePolicies)+contextPolicyCount)
160: 	policies = append(policies, g.databasePolicies...)
161: 	policies = append(policies, g.boundPolicies...)
162: 	policies = append(policies, dynamicPolicies...)
163: 	var residuals [][]residual
164: 	var writes [][]writeResidual
165: 	for _, policy := range policies {
166: 		decision := policy.Decide(ctx, request)
167: 		if !decision.Allowed {
168: 			return nil, nil, &DeniedError{Decision: decision}
169: 		}
170: 		for i, condition := range decision.Residuals {
171: 			if condition == nil || i >= len(request.Resources) {
172: 				continue
173: 			}
174: 			if residuals == nil {
175: 				residuals = make([][]residual, len(request.Resources))
176: 			}
177: 			residuals[i] = append(residuals[i], residual{
178: 				policy:       decision.Policy,
179: 				policySource: decision.PolicySource,
180: 				rule:         decision.Rule,
181: 				text:         decision.Condition,
182: 				resource:     request.Resources[i],
183: 				condition:    condition,
184: 			})
185: 		}
186: 		for i, write := range decision.Writes {
187: 			if write == nil || i >= len(request.Resources) {
188: 				continue
189: 			}
190: 			if writes == nil {
191: 				writes = make([][]writeResidual, len(request.Resources))
192: 			}
193: 			writes[i] = append(writes[i], writeResidual{
194: 				policy:       decision.Policy,
195: 				policySource: decision.PolicySource,
196: 				resource:     request.Resources[i],
197: 				residual:     write,
198: 			})
199: 		}
200: 	}
201: 	return residuals, writes, nil
202: }
203: 
204: func (g guard) checkContext(ctx context.Context) error {
205: 	return g.requireContextPolicy(0, len(g.boundPolicies)+len(policiesFromContext(ctx)))
206: }
207: 
208: func (g guard) requireContextPolicy(operation Operations, contextPolicyCount int) error {
209: 	if !g.requireContext || contextPolicyCount > 0 {
210: 		return nil
211: 	}
212: 	return &DeniedError{Decision: Decision{
213: 		Operation:   operation,
214: 		Policy:      "context",
215: 		Effect:      effectDeny.String(),
216: 		Explanation: "a context-bound access policy is required",
217: 	}}
218: }
```

## dal-go/dalgo/access/write.go:55

```
55: func (s securedWriteSession) enforceWrites(ctx context.Context, operation Operations, writes [][]writeResidual, targets []writeTarget) error {
56: 	if len(writes) == 0 {
57: 		return nil
58: 	}
59: 	for i, target := range targets {
60: 		perResource := writes[i]
61: 		if len(perResource) == 0 {
62: 			continue
63: 		}
64: 		images, err := s.writeImages(ctx, operation, target, perResource[0])
65: 		if err != nil {
66: 			return err
67: 		}
68: 		for _, w := range perResource {
69: 			if err := evaluateWrite(operation, images, w); err != nil {
70: 				return err
71: 			}
72: 		}
73: 	}
74: 	return nil
75: }
76: 
77: // writeImages reads the pre-image (inside the caller's transaction) and
78: // computes the post-image for one target.
79: func (s securedWriteSession) writeImages(ctx context.Context, operation Operations, target writeTarget, w writeResidual) (writeImages, error) {
80: 	var images writeImages
81: 	if operation != Insert {
82: 		reader, ok := s.session.(dal.ReadSession)
83: 		if !ok {
84: 			return images, w.deny(operation, "", "", "a conditional write needs a session that can read the row's pre-image")
85: 		}
86: 		shadow := record.NewRecordWithData(target.key, &map[string]any{})
87: 		if err := reader.Get(ctx, shadow); err != nil {
88: 			return images, err
89: 		}
90: 		if shadow.Exists() {
91: 			images.exists = true
92: 			images.pre = *shadow.Data().(*map[string]any)
93: 		}
94: 	}
95: 	images.updates = target.updates
96: 	switch operation {
97: 	case Insert, Set:
98: 		post, err := condeval.ToMap(target.data)
99: 		if err != nil {
100: 			return images, w.deny(operation, "", "", fmt.Sprintf("row data could not be evaluated: %v", err))
101: 		}
102: 		images.post = post
103: 	case Update:
104: 		if images.exists {
105: 			post := condeval.CloneMap(images.pre)
106: 			if err := condeval.ApplyUpdates(post, target.updates); err != nil {
107: 				return images, w.deny(operation, "", "", fmt.Sprintf("post-image could not be computed: %v", err))
108: 			}
109: 			images.post = post
110: 		}
111: 	}
112: 	return images, nil
113: }
114: 
115: // evaluateWrite applies one policy's write residual. The first alternative
116: // whose Where holds on the pre-image decides an Update, Set of an existing row
117: // or Delete; a new row is admitted by the first alternative whose Check it
118: // satisfies; the Terminal allow applies when no alternative does.
119: func evaluateWrite(operation Operations, images writeImages, w writeResidual) error {
120: 	r := w.residual
121: 	isNewRow := operation == Insert || (operation == Set && !images.exists)
122: 	if isNewRow {
123: 		for _, alternative := range r.Alternatives {
124: 			check, checkText := alternativeCheck(alternative)
125: 			ok, err := condeval.Match(images.post, check)
126: 			if err != nil {
127: 				return w.deny(operation, alternative.Rule, checkText, fmt.Sprintf("post-image check could not be evaluated for rule %q: %v", alternative.Rule, err))
128: 			}
129: 			if ok {
130: 				return checkFields(operation, images, w, alternative)
131: 			}
132: 		}
133: 		return terminalAdmits(operation, images, w, "no rule admits the new row")
134: 	}
135: 	if !images.exists {
136: 		// Nothing to protect: the adapter reports the missing row unless an
137: 		// unconditional allow applies.
138: 		if r.Terminal != nil {
139: 			return nil
140: 		}
141: 		return w.deny(operation, "", "", "row does not exist or is outside every conditional rule")
142: 	}
143: 	for _, alternative := range r.Alternatives {
144: 		ok, err := condeval.Match(images.pre, alternative.Where)
145: 		if err != nil {
146: 			return w.deny(operation, alternative.Rule, alternative.WhereText, fmt.Sprintf("row condition could not be evaluated for rule %q: %v", alternative.Rule, err))
147: 		}
148: 		if !ok {
149: 			continue
150: 		}
151: 		if operation == Delete {
152: 			return nil
153: 		}
154: 		if err := checkFields(operation, images, w, alternative); err != nil {
155: 			return err
156: 		}
157: 		check, checkText := alternativeCheck(alternative)
158: 		ok, err = condeval.Match(images.post, check)
159: 		if err != nil {
160: 			return w.deny(operation, alternative.Rule, checkText, fmt.Sprintf("post-image check could not be evaluated for rule %q: %v", alternative.Rule, err))
161: 		}
162: 		if !ok {
163: 			return w.deny(operation, alternative.Rule, checkText, fmt.Sprintf("post-image check not satisfied for rule %q (check: %s)", alternative.Rule, checkText))
164: 		}
165: 		return nil
166: 	}
167: 	return terminalAdmits(operation, images, w, "row condition not satisfied for any rule")
168: }
169: 
170: // alternativeCheck returns the post-image condition of an alternative: its
171: // Check, or its Where when no Check is declared.
172: func alternativeCheck(alternative WriteAlternative) (dal.Condition, string) {
173: 	if alternative.Check != nil {
174: 		return alternative.Check, alternative.CheckText
175: 	}
176: 	return alternative.Where, alternative.WhereText
177: }
178: 
179: // terminalAdmits applies the Terminal allow, if any, to a write no
180: // alternative decided: a Delete needs no check; other writes must satisfy the
181: // terminal's Check when it has one.
182: func terminalAdmits(operation Operations, images writeImages, w writeResidual, failure string) error {
183: 	terminal := w.residual.Terminal
184: 	if terminal == nil {
185: 		return w.deny(operation, alternativeNames(w.residual), alternativeTexts(w.residual), failure)
186: 	}
187: 	if operation == Delete {
188: 		return nil
189: 	}
190: 	if err := checkFields(operation, images, w, *terminal); err != nil {
191: 		return err
192: 	}
193: 	if terminal.Check == nil {
194: 		return nil
195: 	}
196: 	ok, err := condeval.Match(images.post, terminal.Check)
197: 	if err != nil {
198: 		return w.deny(operation, terminal.Rule, terminal.CheckText, fmt.Sprintf("post-image check could not be evaluated for rule %q: %v", terminal.Rule, err))
199: 	}
200: 	if !ok {
201: 		return w.deny(operation, terminal.Rule, terminal.CheckText, fmt.Sprintf("post-image check not satisfied for rule %q (check: %s)", terminal.Rule, terminal.CheckText))
202: 	}
203: 	return nil
204: }
205: 
206: // checkFields refuses a write that sets (Insert, Set) or touches (Update) a
207: // field outside the deciding alternative's allow-list.
208: func checkFields(operation Operations, images writeImages, w writeResidual, alternative WriteAlternative) error {
209: 	if alternative.fields == nil {
210: 		return nil
211: 	}
212: 	sets := fieldSets{alternative.fields}
213: 	var refused []string
214: 	if operation == Update {
215: 		refused = sets.disallowedUpdates(images.updates)
216: 	} else {
217: 		refused = sets.disallowedPaths(images.post)
218: 	}
219: 	if len(refused) == 0 {
220: 		return nil
```

## dal-go/dalgo/access/condition.go:185

```
185: 	}
186: 	for i := 1; i < len(residuals); i++ {
187: 		for _, r := range residuals[i] {
188: 			return nil, r.deny(Query, fmt.Sprintf("conditional rule %q applies to a joined source; conditional rules on joins are not supported in this version", r.rule))
189: 		}
190: 	}
191: 	base := residuals[0]
192: 	if len(base) == 0 {
193: 		return query, nil
194: 	}
195: 	structured, ok := query.(dal.StructuredQuery)
196: 	if !ok {
197: 		return nil, base[0].deny(Query, "row conditions require a structured query")
198: 	}
199: 	conditions := make([]dal.Condition, len(base))
200: 	for i, r := range base {
201: 		conditions[i] = r.condition
202: 	}
203: 	condition := conditions[0]
204: 	if len(conditions) > 1 {
205: 		condition = dal.NewGroupCondition(dal.And, conditions...)
206: 	}
207: 	if existing := structured.Where(); existing != nil {
208: 		condition = dal.NewGroupCondition(dal.And, existing, condition)
209: 	}
210: 	return dal.WithWhere(structured, condition), nil
211: }
212: 
213: // existsThroughRead upgrades an existence check to a read so a residual can be
214: // evaluated. The record is read into a private map, never into caller memory.
215: func existsThroughRead(ctx context.Context, session dal.ReadSession, key *record.Key, residuals []residual) (bool, error) {
216: 	shadow := record.NewRecordWithData(key, &map[string]any{})
217: 	if err := session.Get(ctx, shadow); err != nil {
218: 		return false, err
219: 	}
220: 	if !shadow.Exists() {
221: 		return false, nil
222: 	}
223: 	if err := checkRecord(Exists, shadow, residuals); err != nil {
224: 		return false, err
225: 	}
226: 	return true, nil
227: }
```

## datatug/datatug-cli/pkg/accesspolicies/run.go:35

```
35: 	// Lines explains, per policy, what applied to the query; nil when
36: 	// unrestricted.
37: 	Lines []Line
38: 	// Query is the query as executed, with parameters substituted.
39: 	Query dal.Query
40: }
41: 
42: // Run executes query over session as a policy-secured application would for
43: // the principal: it puts the principal and variables on the context,
44: // substitutes the query's own parameters, refuses aliases and references to
45: // fields the caller may not see, explains the policies' decisions and opens
46: // the reader through access.SecureReadSession. Denials wrap
47: // access.ErrAccessDenied; refused queries wrap ErrInvalidQuery.
48: func Run(ctx context.Context, session dal.ReadSession, query dal.Query, o Options) (Result, error) {
49: 	if len(o.Policies) == 0 && !o.Unrestricted {
50: 		return Result{}, ErrNoPolicies
51: 	}
52: 	bindings := map[string]any{}
53: 	for name, value := range o.Variables {
54: 		bindings[name] = value
55: 	}
56: 	if o.Principal != nil {
57: 		if o.Principal.ID != nil {
58: 			bindings["currentUser"] = o.Principal.ID
59: 		}
60: 		ctx = access.WithPrincipal(ctx, *o.Principal)
61: 	}
62: 	if len(o.Variables) > 0 {
63: 		ctx = access.WithVariables(ctx, o.Variables)
64: 	}
65: 	query, err := substituteParams(query, o.Variables)
66: 	if err != nil {
67: 		return Result{}, err
68: 	}
69: 	result := Result{Query: query}
70: 	if len(o.Policies) == 0 {
71: 		result.Reader, err = session.ExecuteQueryToRecordsReader(ctx, query)
72: 		return result, err
73: 	}
74: 	result.Lines = Explain(ctx, o.Policies, query, bindings)
75: 	if structured, ok := query.(dal.StructuredQuery); ok && restricted(result.Lines) {
76: 		if aliases := aliasedColumns(structured); len(aliases) > 0 {
77: 			return result, fmt.Errorf("%w: column aliases are not supported under field-restricted policies: %s", ErrInvalidQuery, strings.Join(aliases, ", "))
78: 		}
79: 		hidden, err := hiddenReferences(structured, result.Lines)
80: 		if err != nil {
81: 			return result, fmt.Errorf("%w: cannot verify the query against field-restricted policies: %v", access.ErrAccessDenied, err)
82: 		}
83: 		if len(hidden) > 0 {
84: 			return result, fmt.Errorf("%w: the query selects, filters or sorts on fields the policies hide: %s", access.ErrAccessDenied, strings.Join(hidden, ", "))
85: 		}
86: 	}
87: 	secured := access.SecureReadSession(session, Policies(o.Policies)...)
88: 	result.Reader, err = secured.ExecuteQueryToRecordsReader(ctx, query)
89: 	return result, err
90: }
91: 
92: // substituteParams resolves the query document's own `param` nodes from the
93: // caller's variables. Only the right-hand side of a `where` comparison may
94: // hold a parameter; one anywhere else, or one no variable resolves, is an
95: // ErrInvalidQuery.
96: func substituteParams(query dal.Query, variables map[string]any) (dal.Query, error) {
```

## datatug/datatug-cli/pkg/accesspolicies/explain.go:88

```
88: // Explain asks every loaded policy what it decides for the query's base
89: // collection and returns one Line per policy, in load order. bindings are the
90: // variables the caller supplied; only those are echoed.
91: func Explain(ctx context.Context, loaded []Loaded, query dal.Query, bindings map[string]any) []Line {
92: 	resource := BaseResource(query)
93: 	request := access.Request{Operation: access.Query, Resources: []access.Resource{resource}, Query: query}
94: 	lines := make([]Line, 0, len(loaded))
95: 	for _, item := range loaded {
96: 		lines = append(lines, lineFor(item, item.Policy.Decide(ctx, request), resource, bindings))
97: 	}
98: 	return lines
99: }
100: 
101: func lineFor(item Loaded, decision access.Decision, resource access.Resource, bindings map[string]any) Line {
102: 	line := Line{Policy: item.Policy.Name(), Source: item.Source, Resource: resource.String(), Rule: decision.Rule, Allowed: decision.Allowed, Explanation: decision.Explanation}
103: 	if _, via, ok := strings.Cut(decision.Explanation, " via "); ok {
104: 		line.Via = via
105: 	}
106: 	if !decision.Allowed {
107: 		return line
108: 	}
109: 	if len(decision.Residuals) > 0 && decision.Residuals[0] != nil && decision.Condition != "" {
110: 		line.Condition = decision.Condition
111: 		line.Bindings = bindingsFor(decision.Condition, bindings)
112: 	}
113: 	if len(decision.Writes) > 0 && decision.Writes[0] != nil {
114: 		line.FieldLists = fieldLists(decision.Writes[0])
115: 	}
116: 	return line
117: }
118: 
119: // bindingsFor lists the caller-supplied values of the variables a condition
120: // references, sorted by name.
121: func bindingsFor(condition string, bindings map[string]any) []string {
122: 	seen := map[string]bool{}
123: 	var parts []string
124: 	for _, match := range variableReference.FindAllStringSubmatch(condition, -1) {
125: 		name := match[1]
126: 		value, ok := bindings[name]
127: 		if !ok || seen[name] {
128: 			continue
129: 		}
```

## openvaultdb/openvaultdb-go/pkg/auth/middleware.go:10

```
10: // Config is the server-side auth configuration. A nil *Config means auth is
11: // disabled (the local-dev default documented in the threat model).
12: type Config struct {
13: 	// OwnerToken grants full access to everything, including admin endpoints.
14: 	OwnerToken string
15: 	// Store holds application grants and pending authorization codes.
16: 	Store *Store
17: }
18: 
19: type contextKey struct{}
20: 
21: // FromRequest returns the authenticated principal attached by Middleware,
22: // or nil when the request is unauthenticated (public path or auth disabled).
23: func FromRequest(r *http.Request) *Principal {
24: 	p, _ := r.Context().Value(contextKey{}).(*Principal)
25: 	return p
26: }
27: 
28: // publicPath reports whether the path is reachable without a token: the
29: // discovery document and the connect flow itself.
30: func publicPath(path string) bool {
31: 	switch path {
32: 	case "/.well-known/openvaultdb", "/authorize", "/token":
33: 		return true
34: 	}
35: 	return false
36: }
37: 
38: // Middleware implements the spec's Layer-1 token validation: it
39: // authenticates every request (401 on missing/invalid tokens except on
40: // public paths) and attaches the Principal for the handlers' Layer-2
41: // capability enforcement.
42: func (cfg *Config) Middleware(next http.Handler) http.Handler {
43: 	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
44: 		if publicPath(r.URL.Path) {
45: 			next.ServeHTTP(w, r)
46: 			return
47: 		}
48: 		token := BearerToken(r)
49: 		if token == "" {
50: 			unauthorized(w, "missing bearer token")
51: 			return
52: 		}
53: 		if subtle.ConstantTimeCompare([]byte(token), []byte(cfg.OwnerToken)) == 1 {
54: 			next.ServeHTTP(w, r.WithContext(
55: 				context.WithValue(r.Context(), contextKey{}, &Principal{Owner: true})))
56: 			return
57: 		}
58: 		if g := cfg.Store.Lookup(token); g != nil {
59: 			next.ServeHTTP(w, r.WithContext(
60: 				context.WithValue(r.Context(), contextKey{}, &Principal{Grant: g})))
61: 			return
62: 		}
63: 		unauthorized(w, "invalid or expired token")
64: 	})
65: }
66: 
67: // BearerToken extracts the bearer token from the Authorization header.
68: func BearerToken(r *http.Request) string {
```

## openvaultdb/openvaultdb-go/pkg/server/httperr.go:1

```
1: package server
2: 
3: import (
4: 	"errors"
5: 	"net/http"
6: 
7: 	"github.com/openvaultdb/openvaultdb-go/pkg/core"
8: 
9: 	"github.com/openvaultdb/openvaultdb-go/pkg/schema"
10: )
11: 
12: type errorBody struct {
13: 	Error errorDetail `json:"error"`
14: }
15: 
16: type errorDetail struct {
17: 	Code    string `json:"code"`
18: 	Message string `json:"message"`
19: }
20: 
21: func writeError(w http.ResponseWriter, status int, code, message string) {
22: 	writeJSON(w, status, errorBody{Error: errorDetail{Code: code, Message: message}})
23: }
24: 
25: // writeMappedError converts core/engine/schema errors into the API error
26: // shapes documented in docs/api.md.
27: func writeMappedError(w http.ResponseWriter, err error) {
28: 	var validationErr *schema.ValidationError
29: 	switch {
30: 	case errors.Is(err, core.ErrNotFound), errors.Is(err, core.ErrUpdateOfMissingRecord):
31: 		writeError(w, http.StatusNotFound, "not_found", err.Error())
32: 	case errors.Is(err, core.ErrAlreadyExists):
33: 		writeError(w, http.StatusConflict, "already_exists", err.Error())
34: 	case errors.As(err, &validationErr):
35: 		writeError(w, http.StatusUnprocessableEntity, "schema_validation", err.Error())
36: 	default:
37: 		writeError(w, http.StatusInternalServerError, "internal", err.Error())
38: 	}
39: }
```

## ingitdb/dalgo2ingitdb/database.go:129

```
129: // worker with a read-write transaction. Writes are journaled in-memory before
130: // their first filesystem mutation. If the worker or the optional Git commit
131: // fails, the touched files are restored to their exact pre-transaction state.
132: // This is deliberately a single-writer transaction: it makes a failed
133: // multi-record Synchestra state transition recoverable without pretending that
134: // a Git worktree supports concurrent multi-writer transactions.
135: func (db *Database) RunReadwriteTransaction(ctx context.Context, f dal.RWTxWorker, options ...dal.TransactionOption) error {
136: 	// This lock covers definition load, all record mutations, rollback, and the
137: 	// optional Git ref update. It serialises cooperating Database writers for
138: 	// the complete transaction; Git/editor processes that ignore advisory locks
139: 	// remain outside this adapter's explicit single-writer contract.
140: 	lockPath, err := transactionLockPath(ctx, db.projectPath)
141: 	if err != nil {
142: 		return err
143: 	}
144: 	return withExclusiveLock(lockPath, func() error {
145: 		return db.runReadwriteTransaction(ctx, f, options...)
146: 	})
147: }
148: 
149: func (db *Database) withTransactionReadLock(ctx context.Context, fn func() error) error {
150: 	lockPath, err := transactionLockPath(ctx, db.projectPath)
151: 	if err != nil {
152: 		return err
153: 	}
154: 	return withSharedLock(lockPath, fn)
155: }
156: 
```

## openvaultdb/openvaultdb/spec/decisions/0004-sneat-co-identity-and-space-principals.md:27

```
27: an application or extension as if it were the user or Space would erase the
28: explicit client and capability boundaries required by OpenVaultDB.
29: 
30: ## Decision
31: 
32: The hosted OpenVaultDB Cloud service MUST authenticate humans through the shared
33: Sneat Co. Firebase project `sneat-eur3-1`. The verified Firebase UID is the
34: canonical Sneat Co. `userID` and MUST be used directly as the hosted
35: OpenVaultDB user principal; the service MUST NOT create a second translated
36: OpenVaultDB account ID.
37: 
38: A Sneat Co. Space is a first-class collaborative grant subject identified by
39: its `spaceID`. Its identity and grants MUST NOT depend on whether it is
40: presented by `sneat.work`, `sneat.app`, or an extension mini-app.
41: 
42: Products, extension mini-apps, services, and CLIs are registered clients. A
43: client acts as itself and, where approved, on behalf of an authenticated user
44: or Space. Merely running inside a product or Space MUST NOT confer OpenVaultDB
45: access; the client and requested capabilities remain explicit in every grant.
46: 
47: A contact without a linked Sneat Co. `userID` MAY be the target of a pending
48: invitation, but MUST NOT become an active authenticated grant subject until the
49: contact is securely claimed and linked to a user.
50: 
51: Long-lived device tokens MUST identify the authenticated `userID`, registered
52: client, and approved token scopes. They MUST NOT snapshot Space membership or
53: roles. Resource authorization MUST evaluate the current Space membership and
54: OpenVaultDB grant when the resource is used, failing closed when membership
55: cannot be established.
56: 
57: This hosted identity profile does not constrain self-hosted or third-party
58: OpenVaultDB providers, which MAY use their own identity and group resolvers.
59: 
60: ## Rationale
61: 
62: The shared user ID makes OpenVaultDB collaboration compose directly with the
63: existing Sneat Co. contact and Space membership model. A user removed from a
64: Space can lose access without rewriting a copied list of per-user grants, and a
65: contact can retain its team-local identity while an invitation is pending.
```

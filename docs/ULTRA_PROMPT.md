# VEIL — Ultra Prompt for Agentic Implementation

Use this prompt with Codex, Claude Code, or another coding agent when extending VEIL. Treat it as an execution contract, not inspirational prose.

---

## ROLE

You are the principal engineer, privacy architect, product designer, adversarial reviewer, and verification lead for **VEIL — Research Privacy Gateway**.

VEIL is a **pre-egress privacy control plane** for qualitative research. It exists so researchers can reduce disclosure risk locally before they intentionally copy a reviewed, transformed payload into a non-local AI model.

The product is NOT an AI chat client, NOT an anonymity certificate, NOT an IRB decision engine, and NOT a legal-compliance oracle.

Your north-star invariant is:

> **Identifiable raw research data must never cross the application trust boundary through application code.**

Everything else is subordinate to that invariant.

---

## 0. OBJECTIVE

Build and continuously improve a polished, production-minded local-first research privacy gateway that:

1. ingests qualitative research locally;
2. finds direct identifiers, quasi-identifiers, and reviewer-defined sensitive spans;
3. makes every transformation inspectable and reversible only inside the local trust zone;
4. estimates transparent residual re-identification risk without pretending the score is a probability;
5. blocks unsafe release states;
6. forces human contextual and institutional review;
7. exposes the exact outbound payload before any manual egress;
8. keeps raw data, mappings, watchlists, and review state off remote systems;
9. provides auditable, testable privacy semantics;
10. preserves enough semantic utility for qualitative analysis.

Success is not “the detector found some PII.” Success is a coherent privacy workflow whose state transitions, failure modes, invariants, UX, and evidence can be inspected.

---

## 1. ONTOLOGY FIRST

Before changing code, reason in terms of these entities.

### Entities

- `SourceDocument`
  - raw text or locally extracted research material;
  - highest sensitivity;
  - memory only.

- `Finding`
  - a source span classified as direct identifier, quasi-identifier, watchlist match, or reviewer-marked span;
  - fields: id, type, class, start, end, confidence, source, default action.

- `PolicyDecision`
  - one explicit action per finding: `tokenize | generalize | remove | keep`.

- `Transformation`
  - source span + action + replacement.

- `MappingEntry`
  - original value ↔ stable pseudonymous token;
  - sensitive;
  - memory only unless encrypted for deliberate export.

- `RiskAssessment`
  - score, gate, retained-direct count, retained-quasi count, uncertainty, causal contributions.

- `Attestation`
  - reviewer acknowledgement of contextual review, authorization, and exact-payload review.

- `HandoffPacket`
  - exact transformed content approved for manual copy.

- `AuditRecord`
  - non-identifying evidence: source hash, detector version, finding counts, action counts, risk result, timestamp;
  - MUST NOT contain original identifier values.

### Relationships

```text
SourceDocument 1 ──* Finding
Finding        1 ──1 PolicyDecision
Finding        1 ──0..1 Transformation
Transformation * ──0..1 MappingEntry
SourceDocument 1 ──1 RiskAssessment
RiskAssessment 1 ──* Attestation
RiskAssessment 1 ──0..1 HandoffPacket
SourceDocument 1 ──* AuditRecord
```

### Control boundaries

```text
LOCAL TRUST ZONE
────────────────────────────────────────────────────────────────
Source → Detect → Review → Transform → Risk → Attest → Preview
                                                     │
                                                     │ explicit user action
─────────────────────────────────────────────────────┼──────────
                                                     ▼
EXTERNAL / OUT OF VEIL TRUST ZONE              Clipboard → chosen AI
```

No automatic model call is permitted in the MVP architecture.

---

## 2. NON-NEGOTIABLE INVARIANTS

Never weaken these merely to ship a feature faster.

### Privacy invariants

1. No raw source, finding value, watchlist value, mapping, or unreviewed transformed text may leave the browser through application code.
2. Runtime code must contain no provider SDK, analytics, telemetry, remote font, CDN script, beacon, websocket, or automatic API request.
3. Production CSP must retain `connect-src 'none'` unless the privacy architecture is deliberately redesigned and reviewed.
4. Raw research state must not be stored in cookies, Local Storage, Session Storage, IndexedDB, Cache Storage, or service-worker caches.
5. Reversible mappings are sensitive and must never be exported in plaintext.
6. Audit records must not reproduce original identifier values.

### State invariants

7. Any source mutation invalidates the scan, risk result, attestations, and handoff state.
8. Any watchlist/manual-span/policy mutation invalidates downstream approval.
9. A direct or reviewer-marked identifier set to `keep` hard-blocks release.
10. Low detector confidence cannot silently become release-ready.
11. The exact outbound payload must be inspectable before copy.
12. Copying is explicit; VEIL does not initiate transmission.

### Claim invariants

13. Never say “anonymous,” “safe,” “compliant,” “IRB approved,” or “cannot be re-identified” as a product conclusion.
14. Use **residual risk**, **pseudonymized**, **generalized**, **suppressed**, and **manual review required**.
15. A low numeric score is a workflow signal, not proof of anonymity or a calibrated probability.

---

## 3. STATE MACHINE

Implement behavior as an explicit state machine even if the UI uses derived state.

```text
EMPTY
  │ ingest
  ▼
LOADED
  │ local scan
  ▼
SCANNED
  │ policy application
  ▼
TRANSFORMED
  │ residual-risk assessment
  ├──────────────▶ BLOCKED
  ├──────────────▶ REVIEW_REQUIRED
  └──────────────▶ LOW_RESIDUAL
                         │ 3 human attestations
                         ▼
                       READY
                         │ exact preview
                         ▼
                      RELEASED
```

Invalidation transitions:

```text
source change       → LOADED
watchlist change    → LOADED
manual span change  → LOADED
finding action      → TRANSFORMED + attestations reset
risk-policy change  → TRANSFORMED + attestations reset
```

There must be no stale-release path.

---

## 4. DETECTION ARCHITECTURE

### MVP detector stack

Keep the current deterministic engine as an auditable baseline:

- Taiwan national ID + checksum validation;
- email;
- Taiwan phone;
- URL;
- social handle;
- IPv4;
- student / employee identifier;
- medical-record identifier;
- precise Taiwan street address;
- context-bound person names;
- exact ages;
- exact dates;
- granular Taiwan locations;
- organizations / institutions;
- roles;
- session-only user watchlist;
- reviewer-marked spans.

### Overlap resolution

Overlapping detections must be deterministic.

Order candidates by:

1. semantic priority;
2. confidence;
3. longer span;
4. earlier position.

Manual findings and watchlist terms outrank generic heuristics.

### Future semantic detector

If adding local NER:

- run entirely inside the local trust zone;
- prefer locally hosted WASM/ONNX assets;
- pin model artifacts with checksums;
- maintain deterministic rules in parallel;
- expose disagreement between detectors;
- never replace human review with confidence theater.

Do not add a remote LLM “PII detector” unless the architecture changes to an institutionally approved private deployment and the trust boundary is rewritten first.

---

## 5. TRANSFORMATION SEMANTICS

Every finding gets one explicit action.

### `tokenize`

Use stable within-document pseudonyms.

```text
林美晴 → [PARTICIPANT_001]
林美晴 → [PARTICIPANT_001]
```

Referential consistency is mandatory.

### `generalize`

Preserve analytic signal while reducing precision.

```text
27 歲          → [AGE_20_29]
2025-09-14     → [2025_H2]
內湖區         → [LOCATION_NORTH_TAIWAN]
某研究所       → [UNIVERSITY_OR_RESEARCH_INSTITUTION]
博士班二年級   → [RESEARCH_STUDENT]
```

### `remove`

Suppress the complete finding with a typed placeholder.

### `keep`

Retain verbatim. Direct/manual identifiers retained verbatim must hard-block release.

---

## 6. RESIDUAL-RISK ENGINE

Keep the model transparent and causal.

Per finding:

```text
contribution = base_weight(type)
             × action_multiplier(action)
             × uncertainty_multiplier(confidence)
```

Add explicit interaction penalties for combinations of retained quasi-identifiers.

Minimum output:

```ts
type RiskAssessment = {
  score: number;                 // 0..100 workflow heuristic
  gate: 'BLOCKED' | 'REVIEW_REQUIRED' | 'LOW_RESIDUAL';
  retainedDirect: number;
  retainedQuasi: number;
  unresolvedLowConfidence: number;
  contributions: Array<{
    type: string;
    action: string;
    points: number;
    reason: string;
  }>;
};
```

### Current release policy

```text
retained direct/manual > 0  => BLOCKED
score > 20                   => REVIEW_REQUIRED
unresolved uncertainty > 0   => REVIEW_REQUIRED
otherwise                    => LOW_RESIDUAL
```

Then require all human attestations before `READY`.

Never optimize an eval by simply lowering weights or thresholds.

---

## 7. HUMAN REVIEW GATE

The reviewer must explicitly attest to all three dimensions:

### Context
“I reviewed every detected span and narrative clues the detector may miss, including rare events, tiny teams, unusual careers, health histories, and public-linkage clues.”

### Authorization
“The applicable IRB, participant consent, NDA, data-use agreement, institutional policy, and law permit the intended external processing.”

### Exact payload
“I inspected the exact transformed payload and judge that it contains no information I consider identifying or prohibited.”

These are state predicates, not passive warnings.

---

## 8. UX / VISUAL DESIGN BAR

The site should feel like a high-end research operations terminal, not an admin dashboard template.

### Visual language

- near-black graphite base;
- dense 1 px technical borders;
- signal lime for locally verified/ready state;
- cyan for system/runtime state;
- amber for review/egress boundary;
- red only for block/failure;
- native system sans + precise monospace UI labels;
- no remote fonts;
- no stock illustrations;
- no fake security badges;
- no gradient-heavy SaaS cliché.

### Information architecture

The first screen must answer, without reading documentation:

1. What problem does VEIL solve?
2. Where is the trust boundary?
3. Does anything leave automatically?
4. What does the operator do next?
5. What blocks release?

The workbench must simultaneously expose:

```text
A / SOURCE        raw local input
B / FINDINGS      masked identifier ledger
C / TRANSFORM     exact de-identified output
D / RISK          causal residual-risk ledger
E / RELEASE       human gate + exact handoff
```

### Interaction details

- sensitive values masked by default;
- deliberate reveal per finding and reveal-all control;
- drag/drop and paste;
- keyboard actions;
- action changes update preview/risk immediately;
- source/policy changes visibly revoke prior approval;
- release button disabled with an explicit causal reason;
- modal shows exact outbound payload;
- responsive down to ~390 px without horizontal page overflow;
- honor `prefers-reduced-motion`;
- keyboard focus must always be visible.

---

## 9. AGENTIC ENGINEERING LOOP

Operate as four logically separate roles.

### PLANNER

Before editing:

```text
Objective
Impacted entities
State transitions
Invariants at risk
Threats introduced
Acceptance tests
Rollback/failure strategy
```

Do not start implementation until the privacy semantics are clear.

### EXECUTOR

Implement the smallest coherent change.

Rules:

- pure functions for privacy semantics;
- deterministic transformations;
- browser-native APIs before dependencies;
- minimal mutation surface;
- explicit state invalidation;
- no hidden network path;
- no copy that overclaims protection.

### CRITIC

Attack the implementation.

Ask:

- Can raw content escape through any runtime path?
- Can a direct identifier survive while the gate says ready?
- Can stale attestations survive a source edit?
- Can an overlapping detector hide a more severe finding?
- Does an audit export leak the raw value?
- Can a reversible mapping be exported unencrypted?
- Does UI wording imply anonymity/compliance?
- Does a low score create a false sense of certainty?
- Does mobile layout hide a blocker?
- Did a new dependency create supply-chain or egress risk?

Classify failures:

```text
S1 = raw-data disclosure / release-gate bypass
S2 = privacy-semantic defect / misleading guarantee
S3 = workflow/UX defect affecting safe review
S4 = cosmetic / non-safety defect
```

No unresolved S1/S2 is shippable.

### VERIFIER

Always run:

```bash
npm run verify
```

Then verify the affected state transitions and adversarial scenario manually or in automated tests.

A change is complete only when:

- syntax passes;
- unit tests pass;
- zero-egress audit passes;
- privacy semantics have regression coverage;
- critic has no unresolved S1/S2;
- documentation matches behavior.

---

## 10. EVALUATION HARNESS

Build evaluation before scaling sophistication.

### Dataset families

Use synthetic/adversarial data only in the repository.

Create corpora for:

- Traditional Chinese interviews;
- mixed Traditional Chinese / English;
- healthcare/social-science scenarios;
- phone/email/ID formatting variants;
- Unicode obfuscation;
- punctuation and line-break splits;
- rare institution + role + age + location combinations;
- tiny teams;
- distinctive life events;
- false-positive traps;
- overlapping identifiers;
- repeated aliases;
- tables represented as plain text.

### Detector metrics

Track per type:

- recall;
- precision;
- false negatives weighted by severity;
- false positives per 1,000 words;
- reviewer actions per 1,000 words;
- detection disagreement;
- latency.

### Transformation metrics

Track:

- stable-token consistency;
- source-span correctness;
- semantic utility preservation;
- leaked direct identifiers;
- stale decision invalidation.

### Gate metrics

Highest priority metric:

> **False-negative release rate: unsafe synthetic cases incorrectly reaching READY.**

This must dominate cosmetic score calibration.

---

## 11. ADVERSARIAL EVALS

Every release candidate should attempt these attacks:

1. Unicode-obfuscated email and phone values.
2. Identifier broken across punctuation/newlines.
3. `name@example.test` overlapping a URL.
4. Taiwan ID with valid vs invalid checksum.
5. Manual span overlapping an automated finding.
6. Watchlist term overlapping an organization finding.
7. Repeated name requiring one stable token.
8. Direct identifier changed to `keep` after attestations.
9. Source edited after gate becomes ready.
10. Watchlist edited after gate becomes ready.
11. Attempt to export mapping with a weak/incorrect password.
12. Search audit JSON for original values.
13. Runtime source scan for network/persistence APIs.
14. Mobile layout with long identifiers and long organization names.
15. Rare quasi-identifier combination that appears individually harmless.

---

## 12. FAILURE RECOVERY

Design every subsystem with a fallback.

| Failure | Required recovery |
|---|---|
| detector misses term | manual selection + session watchlist |
| detector overlap conflict | deterministic priority resolution |
| ambiguous finding | reviewer action + review-required gate |
| direct value kept | hard block |
| stale approval | automatic invalidation |
| crypto export failure | no plaintext fallback |
| clipboard permission failure | select exact payload for manual copy |
| unsupported file | explain supported local text formats; never upload for conversion |
| new model unavailable | irrelevant to core flow; VEIL stays provider-independent |
| institutional permission unclear | do not authorize release; require external policy review |

---

## 13. ROADMAP EXECUTION

### P0 — Privacy contract

- ontology;
- trust boundary;
- state machine;
- threat model;
- release predicate;
- claim policy.

**Done when:** semantics are documented and testable.

### P1 — Deterministic local MVP

- text ingestion;
- deterministic identifier detector;
- watchlist/manual spans;
- transform ledger;
- stable pseudonyms;
- residual-risk score;
- attestations;
- exact handoff preview;
- audit export;
- encrypted mapping export;
- zero-egress CI.

**Done when:** `npm run verify` is green and release bypass tests fail closed.

### P2 — Local semantic NER

- browser-local WASM/ONNX;
- zh-TW + English;
- deterministic rules retained as ensemble member;
- detector disagreement UI;
- local artifact checksums.

Target direct-identifier recall ≥ 0.995 on curated synthetic/adversarial evaluation data before production claims.

### P3 — Corpus-level uniqueness

- cross-document token consistency;
- equivalence classes;
- rare-combination warnings;
- k-anonymity diagnostics where structurally meaningful;
- synthetic auxiliary linkage attacks;
- local-only corpus index.

Never present k-anonymity alone as proof of anonymity.

### P4 — Research formats

- DOCX local extraction;
- PDF local text extraction with metadata/image warnings;
- CSV schema-aware policies;
- transcript/subtitle formats;
- batch review queue;
- reviewer roles.

### P5 — Institutional deployment

- self-hosted static bundle;
- SSO/RBAC around application access;
- signed organization policy packs;
- controlled update provenance;
- managed endpoint guidance;
- optional separately authorized encrypted mapping vault.

Raw source still must not cross the application boundary simply because SSO exists.

### P6 — Evaluation-driven improvement loop

```text
failure corpus
   ↓
planner → hypothesis
   ↓
executor → minimal patch
   ↓
critic → privacy + UX attack
   ↓
verifier → fixed evals + regressions
   ├─ fail → repair loop
   └─ pass → candidate release
```

Self-improvement may optimize implementation, detectors, and UX. It may NOT silently rewrite the normative invariants or release policy.

---

## 14. DEFINITION OF DONE

A feature is done only if all relevant conditions hold:

### Correctness
- deterministic behavior where specified;
- overlap handling is stable;
- transformations preserve indexes correctly;
- state invalidation is complete.

### Privacy
- no new application egress;
- no raw persistent storage;
- no audit leakage;
- no plaintext mapping export;
- no release-gate bypass.

### Security
- CSP remains restrictive;
- no unnecessary dependency;
- crypto remains browser-native and authenticated;
- security assumptions documented.

### UX
- blocker reason visible;
- finding values masked by default;
- exact payload inspectable;
- keyboard usable;
- responsive;
- no misleading confidence theater.

### Verification
- `npm run verify` green;
- relevant adversarial tests added;
- documentation updated;
- no S1/S2 critic finding unresolved.

---

## 15. OUTPUT FORMAT FOR EVERY AGENT RUN

Return a compact engineering record:

```markdown
## Objective

## Ontology impact
- entities:
- relationships:
- state transitions:

## Invariants checked

## Implementation
- files changed:
- behavior changed:

## Threat / failure analysis
- S1:
- S2:
- S3:

## Verification
- commands:
- tests added:
- result:

## Remaining limitations

## Next highest-leverage step
```

If verification fails, do not present the work as complete. Enter a repair loop and fix the failure or clearly report the unresolved blocker.

---

## FINAL PRODUCT PRINCIPLE

VEIL should make the safe workflow the obvious workflow:

```text
RAW RESEARCH
    ↓
LOCAL DETECTION
    ↓
HUMAN REVIEW
    ↓
DE-IDENTIFICATION
    ↓
RESIDUAL-RISK CHECK
    ↓
INSTITUTIONAL + CONTEXT ATTESTATION
    ↓
EXACT PAYLOAD PREVIEW
    ↓
MANUAL EXTERNAL USE
```

Do not optimize for the illusion that one click “anonymizes” research. Optimize for **visible trust boundaries, explicit transformations, fail-closed release control, and evidence-driven improvement**.

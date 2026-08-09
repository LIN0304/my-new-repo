# VEIL — Research Privacy Gateway

```text
┌──────────────────────────────────────────────────────────────────────┐
│  RAW RESEARCH  →  LOCAL DETECTION  →  TRANSFORM  →  VERIFY  │ GATE │
│       never leaves the tab                     manual release only   │
└──────────────────────────────────────────────────────────────────────┘
```

**VEIL** is a dependency-free, local-first de-identification workspace for qualitative research. It detects direct and quasi-identifiers in interview transcripts, lets a human reviewer choose a transformation policy for every finding, computes a conservative residual-risk score, and produces an inspectable handoff packet only after the release gate passes.

The application performs **no automatic cloud call**, ships **no analytics**, uses **no persistent browser storage**, and declares `connect-src 'none'` in its Content Security Policy. Raw source text, watchlists, and identifier mappings stay inside the current browser tab unless the operator deliberately downloads an encrypted mapping file.

> VEIL reduces disclosure risk; it does not prove that a dataset is anonymous, provide legal advice, or override an IRB protocol, consent form, NDA, data-use agreement, institutional policy, or applicable law.

![VEIL desktop operator console](./docs/previews/desktop-hero.webp)

## What the MVP does

- Reads `.txt`, `.md`, `.csv`, and `.json` files locally with `File.text()`; maximum 5 MB.
- Detects Taiwan national IDs with checksum validation, email addresses, phone numbers, URLs, social handles, IPv4 addresses, student or employee IDs, medical record IDs, Taiwan street addresses, contextual person names, exact dates, exact ages, locations, organizations, roles, reviewer-marked spans, and session-only watchlist terms.
- Resolves overlapping findings deterministically by priority, confidence, and span length.
- Supports per-finding actions: **tokenize**, **generalize**, **remove**, or **keep** where permitted.
- Preserves referential consistency through stable local tokens such as `[PARTICIPANT_001]`.
- Generalizes ages, dates, Taiwan regions, organizations, and roles with deterministic rules.
- Scores residual risk and returns one of three gates: `BLOCKED`, `REVIEW REQUIRED`, or `LOW RESIDUAL`.
- Requires three explicit human attestations before release.
- Shows the exact cloud handoff payload before the operator manually copies it.
- Exports a non-identifying audit report containing hashes, counts, policies, and risk results—but never original identifier values.
- Optionally exports the reversible mapping encrypted with AES-256-GCM and PBKDF2-SHA256 using browser-native Web Crypto.

## Trust boundary

```text
LOCAL TRUST ZONE                                                  EXTERNAL

Source file / paste
        │
        ▼
┌──────────────────────┐
│ deterministic scan   │  rules + watchlist + manual spans
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ transformation plan  │  tokenize / generalize / remove / keep
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ residual-risk engine │  blocked / review / ready
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ human attestations   │  source edit invalidates this state
└──────────┬───────────┘
           ▼
┌──────────────────────┐              ┌───────────────────────────┐
│ exact packet preview │ ── copy ───▶ │ user-selected cloud model │
└──────────────────────┘              └───────────────────────────┘

No code path automatically crosses the boundary.
```

## Run locally

Requirements: Node.js 20 or newer. Runtime dependencies: **zero**.

```bash
npm run verify
npm run dev
```

Open `http://127.0.0.1:4173`. The server binds to loopback by default.

The app can also be served by any static server. Keep the security headers in [`vercel.json`](./vercel.json), or reproduce them on the chosen host.

## Verification

```bash
npm run check          # JavaScript syntax checks
npm test               # deterministic engine + crypto tests
npm run audit:egress   # rejects runtime network clients and remote URLs
npm run verify         # all of the above
```

The static egress audit checks the runtime entrypoints for remote HTTP(S) URLs, `fetch`, `XMLHttpRequest`, `WebSocket`, `EventSource`, `sendBeacon`, persistent browser storage, and the required `connect-src 'none'` policy.

## State machine

```text
EMPTY
  └─ ingest ─▶ LOADED
                 └─ scan ─▶ SCANNED / TRANSFORMED
                                ├─ high risk ─▶ BLOCKED
                                ├─ uncertainty ─▶ REVIEW
                                └─ low residual + attestations ─▶ READY
                                                                     └─ exact preview ─▶ RELEASED

Any source, watchlist, manual-span, or policy change invalidates downstream release state.
```

## Repository structure

```text
veil-research-gateway/
├── index.html                    # application shell and trust-boundary UI
├── styles.css                    # complete visual system; no remote assets
├── src/
│   ├── app.js                    # session state, UI orchestration, release gate
│   ├── engine.js                 # pure deterministic detection/risk engine
│   └── crypto.js                 # Web Crypto hashing and encrypted export
├── tests/
│   ├── engine.test.mjs
│   └── crypto.test.mjs
├── scripts/
│   ├── serve.mjs                 # loopback-only static development server
│   └── verify-static.mjs         # zero-egress static audit
├── docs/
│   ├── ARCHITECTURE.md
│   ├── PRIVACY_SPEC.md
│   ├── THREAT_MODEL.md
│   ├── ROADMAP.md
│   ├── REFERENCES.md
│   ├── VERIFICATION.md
│   └── ULTRA_PROMPT.md           # execution prompt for coding agents
├── .github/workflows/ci.yml
├── AGENTS.md
├── SECURITY.md
├── CONTRIBUTING.md
├── vercel.json
└── LICENSE
```

## Design system

VEIL uses a dense research-operations console rather than a generic form UI:

- graphite surfaces, terminal typography, signal-lime and cyan state accents;
- a visible trust-boundary map and five-stage workflow rail;
- masked findings by default, with deliberate reveal controls;
- risk represented as both a quantitative gauge and causal contribution ledger;
- no decorative stock imagery, remote fonts, trackers, or third-party UI libraries;
- responsive layouts and keyboard-accessible native controls.

## Security invariants

1. Raw source material never leaves the browser through application code.
2. There is no automatic invocation of a cloud model or provider API.
3. Raw research state is not persisted in Local Storage, Session Storage, IndexedDB, cookies, or a service worker.
4. The reversible mapping exists in memory and is downloadable only after explicit password-based encryption.
5. Every source or policy mutation invalidates the previous risk assessment and release state.
6. Audit exports contain hashes and metadata, not original identifier values.
7. The release action opens an exact payload preview; it does not transmit anything.
8. Detection and transformation behavior is deterministic and testable.
9. Human contextual review is mandatory even when the numeric score is low.
10. The product never claims guaranteed anonymization or automatic compliance.

See [`docs/PRIVACY_SPEC.md`](./docs/PRIVACY_SPEC.md) and [`docs/THREAT_MODEL.md`](./docs/THREAT_MODEL.md) for the normative model. Primary standards and guidance are tracked in [`docs/REFERENCES.md`](./docs/REFERENCES.md); current test and browser evidence is recorded in [`docs/VERIFICATION.md`](./docs/VERIFICATION.md).

## Current limitations

- The detector is a conservative rules-and-context engine, not a multilingual NER model. It can miss indirect identifiers, uncommon names, narrative uniqueness, and domain-specific codes.
- Risk scoring is a transparent heuristic for workflow control—not a statistically calibrated probability of re-identification.
- The MVP evaluates one document at a time. Corpus-level uniqueness, k-anonymity, l-diversity, t-closeness, membership inference, linkage attacks, and external population data are not yet implemented.
- PDF, DOCX, audio, image OCR, and spreadsheet workbooks are intentionally excluded from the first release.
- A compromised browser, extension, operating system, deployment host, clipboard manager, or endpoint security product remains outside the application’s protection boundary.
- Client-side controls can be modified by a technically capable operator; institutional enforcement requires managed infrastructure and policy controls.

## Agent execution prompt

The full ontology-first build and improvement prompt is in [`docs/ULTRA_PROMPT.md`](./docs/ULTRA_PROMPT.md). It specifies planner/executor/critic/verifier roles, invariants, evals, failure-recovery loops, UI quality bars, and a phased production roadmap.

## License

Apache License 2.0. See [`LICENSE`](./LICENSE).

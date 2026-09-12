---
name: bb-hunt
description: >-
  Production bug-bounty hunting workflow — the single entry point for authorized engagements.
  Invoke when the operator provides a scope + handoff + credentials for an authorized program
  (HackerOne / Bugcrowd / vendor / self-hosted VDP / pentest / CTF). Runs the full pipeline
  end-to-end: Mandatory Inputs Gate → List Files + Surface + Asset Inventory → JS/Source-Map
  Decompile → Threat Model → Planner → 34 specialized agents (28 core + 6 extras: invite/
  magic-link, JWT, multi-tenancy, payment-logic, sourcemap/webpack, chain-discovery) →
  7-Question Validation Gate → Report Reviewer → program-ready reports. Session-expiry aware
  (pauses and requests fresh cookies); role-aware (activates admin/readonly/tenant-B agents
  only when accounts are present); WAF/rate-limit adaptive. Uses hackerone-hunter as knowledge,
  live-harness as execution, CocoIndex on large corpora, bb-agents contract + evidence gate.
license: For authorized security testing only.
---

# bb-hunt — the ultimate hunting workflow

**One skill, one workflow, one pipeline.** Not a phase, not a builder — the production
hunting mode for authorized bug-bounty engagements.

## When to invoke

The operator says any of:
- "Hunt this program"
- "Start bb-hunt on <target>"
- "Run the full workflow on <target>"
- Pastes a program handoff / scope block / `CLAUDE_HANDOFF.md`
- Explicitly invokes `/bb-hunt`

If the operator asks about a single vuln class in isolation, invoke that specific agent skill
directly instead (`bb-idor`, `bb-xss`, etc.). `bb-hunt` is for engagements.

## The mindset (adopt before Phase 1)

> HUNT → VALIDATE → PROVE → IMPACT → REPORT
>
> Quality ≫ quantity. One valid high-impact bug beats 100 low-confidence alerts.
>
> Never assume authorization. Never invent findings. Never touch out-of-scope assets.

## Phase 0 — Mandatory Inputs Gate

**Halt** and request every missing item from `inputs.md` before any active testing. Passive
inspection of user-supplied data (`CLAUDE_HANDOFF.md`, recon tree, APK) is allowed while
waiting.

Required:
1. Root domain(s) + wildcard rules (`*.target.com` yes/no)
2. **Verbatim** in-scope / out-of-scope lists from the program policy
3. Auth model:
   - Self-signup / invite-only / SSO / OAuth / OpenID
   - How to create Account A + Account B (required for BAC/IDOR)
   - Session/cookie/JWT storage, lifetime, rotation
   - Role hierarchy (guest → user → team lead → admin → superadmin …)
   - Invite / magic-link token properties (single-use? expiry? scoped?)
4. Focus areas + payout table
5. Program platform + slug
6. Credentials / cookies / two ready accounts
7. Full `CLAUDE_HANDOFF.md` + recon tree (`BB/<target>/`)
8. Android APK or source in scope? (activates Mobile Agent first)

**Missing input protocol:** state exactly which items are missing, wait. Do not proceed on
"I'll figure it out."

## Phase 1 — LIST FILES + SURFACE + ASSET INVENTORY

Parse the handoff and the entire `BB/<target>/` tree. Produce three deliverables (living
documents; update every subsequent phase):

### A. Prioritized Attack Surface Table

| Priority | Host / Domain | Type | Key Endpoints / JS | Sensitive Data | Notes |

Priority order (non-negotiable):
1. Main dashboard / SPA (`app.*`)
2. Backend API (`api.*` or equivalent)
3. Auth / identity / invite flows
4. Admin / internal panels
5. Other in-scope hosts

### B. Asset & Trust Boundary Inventory

Every object type + ownership model:
- Users, Teams/Organizations, Invites, Cards, Transactions, Files, Wallets, Roles, Permissions,
  Webhooks, API Keys …
- Who owns what (user / team / org / global)
- Cross-tenant boundaries — the money bug lives here
- High-value assets (money movement, PII, admin actions, invite tokens)

### C. Confirmed High-Value Endpoints List

Every functional / sensitive endpoint discovered. **Every later agent must hit every item on
this list**. Not aspirational — enforced by the planner's coverage tracker.

**Also do:**
- Grep all URLs + JS for secrets, JWTs, API keys, internal hosts, signed URLs, emails,
  GraphQL, source maps, `.env`, `.git`, actuators, swagger, etc.
- Direct-fetch (curl + browser) leaked panels, source maps, backup files.
- **If JS corpus or URL list > 50 items → CocoIndex the corpus first, then semantic-search for
  endpoints / sinks / secrets.** Uses the existing CocoIndex flow at
  `~/.claude/skills/bb-agents/knowledge/`.

Gate for Phase 1 → Phase 2: A/B/C exist as files under
`~/.claude/bounty/programs/<slug>/runs/<run_id>/`.

## Phase 2 — JS DECOMPILATION / SOURCE-MAP / WEBPACK

Runs whenever the target ships bundled JS (basically every SPA). Delegates to the
`sourcemap-webpack` agent (`~/.claude/skills/bb-agents/agents/sourcemap-webpack/`).

- Download every JS file, check for `//# sourceMappingURL`.
- Fetch source maps, unpack webpack chunks (`webpack:///` sources).
- Extract: hidden API endpoints, feature flags, internal routes, hardcoded secrets, GraphQL
  schemas, admin paths, dev/staging URLs.
- Feed every new endpoint into the High-Value Endpoints List and Asset Inventory.

## Phase 3 — THREAT MODEL BUILDING

Target-specific. Not a template. Reads the asset inventory + surface table + auth model, then
writes a concise mind-map style threat model:

- **Business purpose** of this exact application (payments? messaging? code hosting? SaaS
  workflow?)
- **Primary actors** and their goals (paying customer, free user, admin, integration partner,
  attacker outside the trust boundary)
- **Critical assets** — what attacker gains by compromising them (money, PII, code, tenant
  data)
- **State machines / workflows** for every important flow (invite → onboarding → payment →
  team management → refund → offboarding). One diagram per workflow.
- **Trust boundaries** — browser↔API, user↔user, tenant↔tenant, service↔service.
- **Highest-ROI attack classes for this target** (different every time — a fintech is not a
  code-hosting platform is not a project-management app).

Ask clarifying questions here if anything is still missing (business model, workflow intent).
The planner's quality depends on this being right.

## Phase 4 — PLANNER

Runs `~/.claude/skills/bb-agents/orchestrator/planner.py` with:
- Signals from the surface model + threat model
- Focus areas from operator
- Coverage penalties from memory
- Exclusions from program rules

Selects from the **34 agents** (28 core + 6 extras).

**Always-on minimum set** (even if threat model is light):
- `info-disclosure`
- `xss`
- `authn-bypass`
- `authz-bypass` + `idor`
- `business-logic`
- `invite-magic-link` (if invites exist in the auth model)

**Execution rules:**
- Max **6 agents in parallel** (matches operator preference; enforced at Workflow level).
- **Read-only agents first** (recon-based, discovery, XSS with canary, IDOR read tests), then
  mutating agents (write tests, race conditions, business-logic state changes). Prevents
  rate-limit / session stomping.
- **WAF / rate-limit adapt** the moment 429 / captcha / CDN challenge appears (slow down,
  rotate headers, rotate accounts, back off). See `session-handling.md`.
- **Every agent must hit every item on the High-Value Endpoints List** and every relevant
  workflow from the state machine — no cherry-picking.
- **If APK/source in scope** → `mobile` agent runs FIRST (decompile → extract endpoints/secrets
  → feed into API agents). If source code is provided → static-analysis pass
  (semgrep/CodeQL-style) before dynamic testing.

## Phase 5 — FINDING PHASE (agent fan-out)

Selected agents fan out (max 6 concurrent). Each emits `AgentOutput` (schema:
`~/.claude/skills/bb-agents/_shared/schemas/agent_output.schema.json`) with candidate findings
carrying full context: exact URL, payload, request/response, why it matters, related assets.

**Prefer chains and high-impact issues.** Agents surface `recommended_chain_tests` when they
observe a primitive another agent can consume.

After the fan-out round completes → **`chain-discovery` agent runs** on the union of confirmed
findings. Chains where every edge is independently demonstrated become chain-scope findings
(XSS→ATO, IDOR→privilege escalation, open-redirect→OAuth theft, invite-token→mass ATO, etc.).

## Phase 6 — VALIDATION (STRICT 7-Question Gate)

Every candidate — individual and chain — runs through
`~/.claude/skills/bb-agents/evidence-report/gate.py`. Seven questions, all must pass:

1. **Is it in scope?**
2. **Reproducible from a fresh session** (unauth or new auth as required)?
3. **Impact real and attacker-attainable** (not theoretical, no "could potentially")?
4. **Soft-404 / info-only / non-exploitable ruled out**?
5. **Affects a high-value asset or crosses a trust boundary**?
6. **Full chain demonstrated end-to-end** (preferably twice)?
7. **Severity justified by program payout guidelines**?

Only true positives proceed. Working curl / browser reproduction required.

**False-Positive Learning Loop** — when a candidate is rejected, record the exact reason via
`bb-memory` — the same class of FP is suppressed on future runs on this target.

## Phase 7 — REPORTING + REPORT REVIEWER

`~/.claude/skills/bb-agents/evidence-report/` produces one report per finding using the exact
structure below. Then the **Report Reviewer** pass (mandatory before delivery — see
`report-review.md`):

- Impact realistic and matches evidence
- Steps minimal and clear (works for triager)
- All tokens/secrets sanitized (redaction gate)
- Severity aligns with program payout guidelines
- No duplicate or overlapping reports

### Report structure (exact)

```
Title: [Clear Impact] at [exact location / endpoint]

Summary
Vulnerable Endpoint / Code Location
Steps to Reproduce      (minimal, numbered, copy-paste ready)
Expected Behavior
Actual Behavior
Impact                  (true ceiling — mass data, financial, ATO, priv-esc)
Remediation
PoCs                    (requests, responses, sanitized tokens, evidence)
```

## Operational discipline

- **Authorized scope only.** Never invent findings.
- **Continuous status updates** after every major phase, plus any questions for the operator.
- **When authenticated testing is required, pause** and request creds / cookies / two accounts.
- **Phase A (unauth) must fully complete** and pass its gate before Phase B (authz/business
  logic).
- **Living High-Value Endpoints List and Asset Inventory** — updated throughout the entire
  engagement.
- **Session expiry** — see `session-handling.md`. Detect via response signature (redirect to
  login, 401 with specific body, missing cookie). On detect: PAUSE, request fresh cookies,
  resume with the same run_id (planner reuses cached state).
- **Role injection mid-run** — if operator adds admin/readonly accounts partway, the surface
  model is refreshed and additional matrix cells activate. Planner re-scores.

## The 34 agents

### 28 core (`~/.claude/skills/bb-agents/agents/`)

`xss`, `xxe`, `csrf`, `idor`, `rce`, `sqli`, `ssrf`, `race-condition`, `subdomain-takeover`,
`open-redirect`, `clickjacking`, `dos`, `oauth`, `account-takeover`, `business-logic`,
`rest-api`, `graphql`, `info-disclosure`, `web-cache`, `ssti`, `file-upload`,
`request-smuggling`, `openid`, `mobile`, `file-reading`, `authz-bypass`, `authn-bypass`, `mfa`

### 6 extras (`~/.claude/skills/bb-agents/agents/`)

- **`invite-magic-link`** — Invite / magic-link token flaws (predictability, reuse, scope
  bleed, mass-invite ATO).
- **`jwt`** — JWT / token manipulation (alg-swap, key confusion, unbounded claims, kid
  injection, jku/x5u abuse).
- **`multi-tenancy`** — Organization/tenant isolation beyond IDOR (org-level admin actions,
  cross-tenant data via UI vs API, tenant-ID injection in body/headers).
- **`payment-logic`** — Payment / financial workflow abuse (negative amounts, currency
  swap, coupon stacking, refund-after-payment, discount-post-validation, MOTO abuse).
- **`sourcemap-webpack`** — Source map + webpack chunk mining (drives Phase 2 output; feeds
  every other agent).
- **`chain-discovery`** — Post-processing agent that composes confirmed primitives into chains
  after each fan-out round.

## Cross-references

- Knowledge & methodology per class: `~/.claude/skills/hackerone-hunter/references/*.md`
- Live-driven capture / replay: `~/.claude/skills/live-harness/`
- Recon pipeline: `~/automate/recon-engine/` (wrapped by `bb-recon`)
- Contract + schemas + planner: `~/.claude/skills/bb-agents/`
- Per-program persistent memory: `~/.claude/bounty/programs/<slug>/`
- Corpus knowledge (14,832 patterns across 28 classes): `KnowledgeAPI().query(vuln_class=...)`
- Top-report browsing (fresh): `https://github.com/reddelexc/hackerone-reports/blob/master/docs/tops_by_bug_type/TOP<CLASS>.md`

## Entry point

```
/bb-hunt

Target:              app.target.com
Wildcard:            *.target.com
Authorization:       h1:target-program
In-scope:            (verbatim)
Out-of-scope:        (verbatim)
Auth model:          self-signup, session cookie 24h, roles: guest/user/admin
Accounts:            acctA (user), acctB (user)  [admin/readonly to be added if available]
Focus:               authz, ato, payment logic
Handoff:             /path/to/CLAUDE_HANDOFF.md
Recon tree:          /home/yash/automate/BB/target.com/
APK:                 none
Budget:              overnight (12h wallclock, 100k requests total, 400/min/host)
```

Files: `inputs.md`, `workflow.md`, `session-handling.md`, `report-review.md` in this skill dir.

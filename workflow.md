# bb-hunt — 12-Phase Workflow

The production runbook. `SKILL.md` is the summary; this is the operational spec every run
walks in order. Phases 1→12 all reference the Universal Validation Lifecycle
(`_shared/validation-lifecycle/`) — the lifecycle runs INSIDE agent tests (Phase 5) but the
workflow phases are the outer envelope.

## The 12 phases

```
0. Scope / authorization gate
1. Passive recon
2. Application model
3. Attack-surface classification
4. High-value agent selection (planner)
5. Initial hypothesis testing (agent fan-out, wave 1)
6. Validation (7Q gate + lifecycle status aggregation)
7. Bypass research (targeted, boundary-aware)
8. Impact analysis (impact graph per confirmed finding)
9. Cross-agent chaining (chain-discovery seeds probes)
10. Independent verification (multi-path per Improvement #3)
11. False-positive elimination (Improvement #9 checklist)
12. Report generation (evidence bundle + reviewer)
```

Phases 5–11 loop; 12 runs once at the end (and again per confirmed chain).

---

## Phase 0 — Scope / authorization gate

Load `inputs.md` checklist. Halt on any missing blocking item. Compile scope regex from
in-scope minus out-of-scope. Start run: `~/.claude/bounty/programs/<slug>/runs/run-<id>/`.
Write `run.json`.

MCP governance loaded from `_shared/mcp-governance/`. `include_writes`, `allow_destructive`
flags stored on the run.

**Halt criteria:** missing target / authorization / in-scope / two accounts / focus / platform
info. Passive analysis of provided artifacts is allowed while waiting.

---

## Phase 1 — Passive recon

Invoke `bb-recon` (wraps `~/automate/recon-engine`). Outputs:
- `subdomains.txt`, `endpoints.txt`, `tech.json`, `flows.jsonl` (operator's authed crawl via
  live-harness), `js_urls.txt`, `params.json`, `secrets.jsonl`.

Passive mode by default (no active scanning). Direct-fetch pass on interesting leaked panels /
source maps / backup files goes through the SAFE ACTIVE tier.

If `wc -l js_urls.txt > 50` OR `wc -l endpoints.txt > 200` → CocoIndex the corpus.

Also runs `sourcemap-webpack` agent inline to unpack `.map` files and webpack chunks — the
discovered endpoints/secrets/schemas broadcast via `bb-memory` to every relevant agent.

---

## Phase 2 — Application model

Invoke `bb-surface-model`. Produces `surface.json` (actors + roles + endpoints + objects +
state machines + trust boundaries + integrations). See `_shared/schemas/…` for shape.

Cross-check against `bb-memory.endpoints_seen` — new endpoints get provenance
(`discovered_via.agent`). Actors from Phase-0 accounts are labeled.

---

## Phase 3 — Attack-surface classification

Living documents built in Phase 1 are now formalized into the three deliverables:
- **A. Prioritized Attack Surface Table** (SPA → API → auth → admin → other)
- **B. Asset & Trust Boundary Inventory** (object types + ownership)
- **C. High-Value Endpoints List** (every functional/sensitive endpoint)

Threat model is written to `runs/<run_id>/threat-model.md`:
- Business purpose
- Actors + goals
- Critical assets
- State machines
- Trust boundaries
- Highest-ROI attack classes FOR THIS target

If threat model can't be completed (business unclear) → PAUSE and ask.

---

## Phase 4 — High-value agent selection (planner)

Runs `orchestrator/planner.py` with:
- Signals collected from Phases 1–3 (see `_shared/severity/priority_signals.md`)
- Focus areas from operator
- Memory coverage penalties
- Program exclusions

Output: `runs/<run_id>/plan.json` — ranked agents with request budgets.

Wave order enforced (read-only first → mutating → post-processing):

| Wave | Agents | Tier requirement |
|---|---|---|
| 1 (read-only)      | info-disclosure, sourcemap-webpack, subdomain-takeover, open-redirect (server-side), clickjacking (frame test), xss (canary only) | PASSIVE + SAFE ACTIVE |
| 2 (read + light mutate) | idor, multi-tenancy, authz-bypass, rest-api, graphql, csrf (PoC only), oauth, openid, jwt, mfa, authn-bypass | + CONTROLLED ACTIVE |
| 3 (mutate + high-cost) | business-logic, payment-logic, invite-magic-link, file-upload, race-condition, ssti, sqli, ssrf, xxe, rce, file-reading, account-takeover, request-smuggling, web-cache | + CONTROLLED ACTIVE |
| 4 (post-processing) | chain-discovery | PASSIVE only |

Max 6 agents in parallel per wave.

---

## Phase 5 — Initial hypothesis testing (agent fan-out, wave 1)

Each selected agent runs its SKILL.md protocol:
1. Retrieve `KnowledgeAPI` patterns for the class + target tech.
2. Consume `bb-memory.broadcasts` intended for the agent (Improvement #6).
3. Generate hypotheses.
4. **For each hypothesis, run the Universal Validation Lifecycle**
   (`_shared/validation-lifecycle/`):
   - Phase 1 OBSERVE — baseline
   - Phase 2 HYPOTHESIZE — state security property (Improvement #1)
   - Phase 3 CONTROL — positive + negative
   - Phase 4 EXPLOIT TEST
5. On `violated` → the agent proceeds through workflow Phases 6–12 for that candidate.
6. On `holds` → recorded to memory as REJECTED (Improvement #7 negative knowledge).

Every AgentOutput carries `recommended_chain_tests` + `potential_chain_edges` for the chain
consumer to consider.

---

## Phase 6 — Validation

For each candidate from Phase 5:
1. Lifecycle Phase 5 INDEPENDENT_VERIFY runs (Improvement #3 multi-path — mandatory for
   classes in `REQUIRES_MULTI_PATH`).
2. `evidence-report/gate.py` runs — the 7-Question Gate + Improvement #9 checklist.
3. If gate passes → status advances toward CONFIRMED; if not → status stays UNCERTAIN and is
   persisted with the specific gap for future runs.

---

## Phase 7 — Bypass research

For candidates the naive test refuted:
1. Invoke `BypassEngine().reason(DefenseContext(...))` — walks the 9 questions
   (Improvement #2) and emits ranked next tests.
2. Agent runs Lifecycle Phase 6 BYPASS with the top-ranked alternate path.
3. If bypass produces `violated` → back to Phase 6.
4. If exhausted → record hypothesis as `UNCERTAIN` with the tried bypasses.

Discipline: after 3 sequential refuted bypass suggestions, back off; the initial hypothesis
may be wrong.

---

## Phase 8 — Impact analysis

For every confirmed finding, Lifecycle Phase 7 IMPACT emits an `ImpactGraph`
(Improvement #4):

```
INPUT_CONTROL → SECURITY_BOUNDARY → PRIVILEGED_OPERATION → ATTACKER_CAPABILITY → BUSINESS_IMPACT
```

The gate refuses to promote to CONFIRMED without a declared `attacker_capability`.

---

## Phase 9 — Cross-agent chaining

`chain-discovery` runs after each wave:
1. Reads all confirmed findings + prior memory.
2. For each finding, reads `potential_chain_edges` (Improvement #5) and consults
   `correlation/patterns.py`.
3. Where a chain precondition holds → seeds a targeted probe back to the appropriate agent
   in the NEXT wave with an elevated priority and a specific hypothesis.
4. When both endpoints of a candidate chain are confirmed AND the downstream finding's
   reproduction demonstrably uses the upstream primitive → build a `Chain` object.

Never fabricates chains. Every edge carries `transition_evidence`.

---

## Phase 10 — Independent verification

For each confirmed finding + confirmed chain, run Lifecycle Phase 5 (INDEPENDENT_VERIFY) if
not already done:
- Class-specific dimension per `_shared/validation-lifecycle/README.md` table
- At least one `verification_dimensions[]` entry with `result: confirmed`

Findings that fail this phase revert to `UNCERTAIN` with a note.

---

## Phase 11 — False-positive elimination

Lifecycle Phase 8 (NEGATIVE_CONTROL) enforces the 10-point checklist (Improvement #9).
Gate re-runs the same checklist as defense-in-depth. Score < 8/10 → `UNCERTAIN`; recorded to
memory so the same signal isn't re-flagged.

---

## Phase 12 — Report generation

`evidence-report/` writes the evidence bundle + `report.md` per the exact structure in
`SKILL.md`. Then the **Report Reviewer** pass (`report-review.md`):
1. Impact realism
2. Steps minimality
3. Redaction / secret sanitization
4. Severity alignment (independent of confidence — Improvement #8)
5. Duplicate / overlap check

Only `verdict: pass` findings land in `runs/<run_id>/delivered/`.

---

## Loop cadence

For overnight/multi-day runs: `/loop` self-paced. Between rounds:
- Orchestrator re-reads memory (including new broadcasts + negative-knowledge REJECTED)
- Coverage-tracker recomputes surface × agent × verdict
- Planner re-scores
- Next wave starts

Ends when:
- Coverage saturated (all agents × HV endpoints have a verdict)
- Budget exhausted
- Operator stops
- Critical chain confirmed AND operator asks to freeze for reporting

## Session-expiry / role injection

See `session-handling.md`. The workflow pauses at any phase on session expiry, resumes with
the same `run_id`. New accounts added mid-run refresh the surface model and re-score the
planner without discarding completed work.

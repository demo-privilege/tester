# bb-hunt — Report Reviewer (Phase 7 QC pass)

Every report that passes the 7-Question Gate then passes through the Reviewer before being
delivered. The Reviewer is a distinct step, not an add-on: reports that fail review go BACK to
the agent, not out to the operator.

## The five review dimensions

Each dimension is `pass` / `fix` / `block`.

### 1. Impact realism (`impact_realism`)

Read the Impact section against the evidence. Ask:
- Does the evidence actually demonstrate this impact?
- Is the "true ceiling" claim justified by the reproduction, or is it extrapolation?
- Any hedging language (`might`, `could potentially`, `hypothetically`)?
- Would a triager accept this impact claim, or push back with "you demonstrated X, not Y"?

Common fails:
- Claims mass PII exposure but PoC shows only one record (need to prove enumeration).
- Claims RCE but PoC shows only reflection.
- Claims ATO but PoC doesn't show attacker logging in as victim.

### 2. Steps minimality (`steps_minimal`)

Read the Steps to Reproduce as if you're the triager. Ask:
- Can I copy-paste these into a shell and reproduce, without editing?
- Are all inputs concrete (not `<your-token-here>`)?
- Is any step irrelevant to the exploit (could be removed)?
- Is any step missing that would block reproduction?

Common fails:
- Steps reference internal tooling (`use bb-idor to send…` — should be `curl -X…`).
- Steps assume knowledge only the finder has ("navigate to the invoice page as you'd
  normally" — which page? which link?).
- Steps include ceremony that isn't needed (login when the endpoint is unauth).

### 3. Redaction / secret sanitization (`redaction_clean`)

Automated check against the `redactions.json` file + a manual scan of the report body. Every:
- Session cookie → `<REDACTED:session>`
- JWT → `<REDACTED:jwt>`
- API token → `<REDACTED:api_token>`
- Real email/phone/name of non-test users → `<REDACTED:pii>`
- Attacker/victim IPs (if IPs identify real users) → `<REDACTED:ip>`

Common fails:
- Cookie value visible in a curl example.
- JWT decoded and shown with real `sub` claim.
- Screenshot didn't pixelate a real user's name.
- URL query string contains real user ID that maps to a live account (case-by-case).

### 4. Severity alignment (`severity_aligned`)

Read the program's payout table (from the mandatory inputs). Read the finding's severity band
and CVSS. Ask:
- Does the band match a payout tier the program actually pays?
- Does the CVSS vector match reality (not inflated to force a tier)?
- Would a triager down-score this? If yes, revise before submitting.

Common fails:
- CVSS AV:N when the exploit actually requires physical / local access.
- Severity: Critical when the impact is "user must click a link" (typically High at most).
- Severity: High when the affected asset is non-sensitive (Medium).

### 5. Duplicate / overlap check (`no_duplicate`)

- Cross-reference against `memory.already_reported` for this program.
- Cross-reference against this run's other confirmed findings — is this a variant of one
  already reported (same root cause, different endpoint)? If yes, either merge into one report
  or clearly differentiate (why this is separately reportable).
- Cross-reference against the program's public disclosed reports (a quick H1 search on the
  finding's endpoint / class / affected asset).

Common fails:
- Same IDOR reported 3 times on 3 endpoints — one report with a list of endpoints is stronger.
- A finding that reproduces a known-public bug on the target (waste of everyone's time).

## Reviewer output

Written to `evidence/<finding_id>/review.json`:

```json
{
  "reviewer_run_id": "review-run-2026-08-11-abc123",
  "reviewed_at": "2026-08-11T22:00:00Z",
  "verdict": "pass",       // pass | fix | block
  "dimensions": {
    "impact_realism":    {"status": "pass", "notes": ""},
    "steps_minimal":     {"status": "fix",  "notes": "Step 3 references bb-idor internal tool; rewrite as curl"},
    "redaction_clean":   {"status": "pass", "notes": ""},
    "severity_aligned":  {"status": "pass", "notes": "H matches program's payout tier"},
    "no_duplicate":      {"status": "pass", "notes": "Not in already_reported; distinct from run-abc123-idor-02 (different root cause)"}
  },
  "blocking_issues": [],
  "fixes_requested": [
    "Rewrite Step 3 as a copy-pasteable curl command."
  ]
}
```

Verdict rules:
- Any dimension `block` → `verdict: block`. Finding does NOT ship. Sent back to agent with
  notes.
- Any dimension `fix` (and no `block`) → `verdict: fix`. Finding does NOT ship. Sent back to
  the report writer (not the agent) with fixes.
- All `pass` → `verdict: pass`. Report copied to `runs/<run_id>/delivered/<finding_id>/`.

## Delivery bundle

Only `verdict: pass` findings land in `runs/<run_id>/delivered/`:

```
delivered/
├── index.md                          # ranked list of shippable reports
├── <finding_id_1>/
│   ├── report.md
│   ├── request.curl.sh
│   ├── screenshots/
│   └── review.json
├── <finding_id_2>/
│   └── ...
```

`index.md` is a table sorted by severity descending, then by potential bounty descending
(if the program's payout table is machine-readable, otherwise operator-sorted).

## Reviewer is not a rubber stamp

If a run produces 10 candidates and the Reviewer ships 3, that's the right outcome. The point
is not to maximize reports delivered — it's to maximize reports **accepted and paid** by the
triager. A rejected report costs credibility and future signal-to-noise on the program.

# bb-hunt — Session handling, role injection, WAF adaptation

The single hardest thing about long-running hunts. This is the exact protocol.

## Session expiry — detection

An agent's request against an authenticated endpoint returns any of:
- HTTP 401 / 403 with a body pattern like `{"error":"unauthorized"}`, `Session expired`,
  `Please log in`
- HTTP 302 to `/login`, `/signin`, `/auth`, `/sso`
- HTTP 200 with a login-form HTML body
- A cookie in the response that clears the session cookie (`Set-Cookie: session=; Max-Age=0`)
- A captcha challenge (`cf-mitigated`, `Cloudflare Ray ID`, `Please verify you are human`)
- HTTP 429 with `Retry-After` far exceeding the run's remaining budget

Detection lives in `live-harness/scripts/replay_authz.py` and `dom_probe.py`; both are aware
of these signatures and mark the flow as `session_invalid`.

## Session expiry — response

Immediate, without exceptions:

1. **PAUSE the entire run.** All in-flight agents finish their current request and stop.
2. **Print a session-refresh block** to the operator, exactly like:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  SESSION EXPIRED — bb-hunt paused
  Account:   acctA (user, tenantT1)
  Detected:  agent=bb-idor, endpoint=/api/v1/invoices/45123, status=302→/login
  Run ID:    run-2026-08-11-abc123
  Progress:  Wave 2, 4/9 agents complete, 2,847/12,000 candidate reqs
  Coverage:  info-disclosure 100%, xss 92%, idor 45%, ...

  To resume:
    Re-login as acctA and provide the fresh state file:
      /path/to/acctA-fresh.json
    Or paste the fresh cookie value here.

  Waiting.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

3. **Do not proceed** until operator provides the new session.
4. Once provided:
   - Update `run.json.test_accounts[<label>].session_state_path`.
   - Verify the new session works: one probe to a known-authed endpoint.
   - Resume from the exact wave/agent position (planner re-issues the interrupted request).

## Role injection mid-run

Operator has more accounts becoming available (admin, readonly, second tenant, etc.):

```
add-account label=acctAdmin role=admin tenant=T1 state=/path/to/acctAdmin.json
add-account label=acctReadonly role=support tenant=T1 state=/path/to/acctReadonly.json
add-account label=acctB2 role=user tenant=T2 state=/path/to/acctB2.json
```

The workflow does:
1. Verify each new session works (one probe each).
2. Update `run.json.test_accounts[]`.
3. Refresh the surface model — new rows appear in the authorization matrix.
4. Re-score planner:
   - `admin` account activates `authz-bypass` (admin vector), `multi-tenancy` (admin actions),
     `payment-logic` (refund path), `mfa` (admin MFA policies).
   - `readonly/support` account activates role-boundary tests (things support can do that they
     shouldn't).
   - Second tenant activates full `multi-tenancy` cross-tenant cell tests.
5. **Do not re-run already-completed cells.** Existing coverage is preserved; only new matrix
   cells get tested.

## WAF / rate-limit adaptation

Signals: HTTP 429, HTTP 403 with WAF fingerprint headers (`Cf-Ray`, `X-Sucuri-ID`,
`X-Akamai-*`, `X-Content-Security-Policy` from Imperva, `X-Amz-Cf-Id`), captcha HTML bodies,
Cloudflare's JS challenge page, sudden latency spike (>10s on previously fast endpoint).

Response ladder (climb until it works):

1. **Local backoff** — halve concurrency for the offending host. Wait 30s. Retry once.
2. **Header rotation** — cycle User-Agent to a small realistic pool; add cache-buster query
   param; test whether removing `X-*` custom headers helps.
3. **Session rotation** — if operator provided multiple accounts of the same role, rotate.
4. **Rate cut** — drop req/min/host from 400 to 60. Log the cut.
5. **Escalate to operator** — if 4 rungs of the ladder don't clear it:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  WAF / rate-limit blocking bb-hunt
  Host:      api.target.com
  Fingerprint: Cloudflare (cf-mitigated: challenge)
  Attempted: backoff, header rotation, session rotation, rate cut to 60/min → all blocked
  Recommend: request testing window from program, or provide rotating egress
  Pausing.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

The workflow does not attempt evasion techniques that would violate program rules
(bulk IP rotation, captcha solving services, header spoofing that impersonates the WAF
vendor's own health checks). Those are operator decisions.

## Rate limits — proactive respect

- Enforce `per_host_reqs_per_minute` at the run level.
- Enforce `per_agent_request_cap` at the agent level.
- Every agent's `AgentOutput.blocked_tests` gets a `reason: rate_limited` entry when the cap
  is hit — the planner reads this and shifts budget to other agents.

## Escalation checklist

Every time the workflow pauses for operator input, include:
- What triggered the pause
- Where the run stopped (wave, agent, coverage %)
- What operator input is expected
- How to resume (exact command shape)

No pause without a clear resume path.

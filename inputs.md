# bb-hunt — Mandatory Inputs

The workflow refuses to start active testing until every applicable item below is answered.
Passive analysis of user-supplied data (handoff, recon tree, APK) is allowed while waiting.

## Blocking (workflow halts without these)

| # | Item | Notes |
|---|---|---|
| 1 | **Root domain(s)** | e.g. `target.com`, `target.io` |
| 2 | **Wildcard rules** | `*.target.com` yes/no. If yes, in-scope subdomain list. |
| 3 | **In-scope list (verbatim)** | Copy exactly from program policy. |
| 4 | **Out-of-scope list (verbatim)** | Copy exactly. |
| 5 | **Program platform + slug** | `h1:target-program`, `bugcrowd:slug`, `vdp:<vendor>`, `pentest:<engagement>`, `ctf:<name>`, `owned:<self>`. |
| 6 | **Auth model** | Self-signup / invite-only / SSO / OAuth / OpenID. See §Auth Details below. |
| 7 | **At least 2 accounts** | acctA + acctB (both regular users) — mandatory for BAC/IDOR/multi-tenancy. Cookies or `state.json` from `live-harness login`. |
| 8 | **Focus areas + payout table** | So the planner knows where to burn budget. |

## Auth Details

Ask for each, mark N/A when not applicable:

- Signup URL and workflow (email verify? invite-only? SSO required?)
- Login URL and workflow (password / SSO / SAML / magic-link / OTP)
- Session token: cookie name, lifetime, rotation on privilege change, HttpOnly, SameSite
- JWT: where stored (localStorage / cookie / header), lifetime, refresh flow, `alg`, `kid`,
  `iss`, `aud` claims
- Role hierarchy — every role name in the app (guest / user / team_lead / admin / superadmin /
  moderator / support / integration_partner …). Which roles the operator can currently obtain.
- Invite / magic-link token properties: single-use? expiry? scoped to email? scoped to role?
- MFA available? Which factors? MFA required for which actions?
- Password reset flow: email token, SMS, security questions, magic link
- Account linking (multiple OAuth providers per account) supported?

## Optional but strongly recommended

| # | Item | Enables |
|---|---|---|
| 9 | **admin account** | admin-only agents (authz-bypass at role level, multi-tenancy admin actions, payment-logic refund path) |
| 10 | **readonly / support account** | role-boundary matrix cells; often surfaces authz gaps |
| 11 | **Second tenant / organization** | cross-tenant matrix cells; multi-tenancy agent full activation |
| 12 | **CLAUDE_HANDOFF.md** | prior recon + notes shortcut Phase 1 |
| 13 | **Recon tree** | `BB/<target>/` — subdomains, endpoints, JS harvest, params. Speeds Phase 1 by hours. |
| 14 | **Android APK / iOS IPA** | activates `mobile` agent FIRST (decompile → endpoints/secrets feed API agents) |
| 15 | **Source code (any snippet)** | static-analysis pass before dynamic testing |
| 16 | **Prior H1 reports on this program** | dedupe list; adds to `already_reported` memory |
| 17 | **Rate limits / testing window** | so we don't trip the program's own defenses |
| 18 | **Prohibited actions** | e.g. "no DoS", "no automated scanning", "no cross-account brute" |

## Session refresh / role injection (mid-run)

**Session expiry** — the workflow detects 401 / redirect-to-login / missing cookie / captcha
challenge. On detect: pause, print exactly which account expired, request the new cookie/state
file. Resume with the same `run_id`. See `session-handling.md`.

**Role added mid-run** — operator can inject a new account at any time:

```
add-account acctAdmin role=admin tenant=T1 state=/path/to/state.json
```

The surface-model refreshes, the authorization matrix grows new rows, the planner re-scores
(admin/readonly agents get boosted), and the next round picks up the new cells.

## Response protocol when inputs are missing

State the missing items exactly as one list:

```
Cannot start active testing. Missing:
- (3) In-scope list — please paste verbatim from the program policy
- (7) Second account — need cookies or state.json for acctB
- (6.role_hierarchy) — please list every role name the app uses
```

Do NOT attempt to proceed with best-guess assumptions. Wait for the operator.

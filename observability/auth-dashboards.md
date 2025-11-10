# Observability & SLOs — Authentication Feature

Purpose  
This document records key SLIs/SLOs, metrics, dashboards and alert thresholds required for a safe staged rollout and operation of the Authentication feature (Cognito Hosted UI, email flows, Google/Apple SSO). Use this as the canonical source for SRE, Platform/Infra, Dev and QA.

Project-internal references
- Assessment: [/features/auth/assessment.md](/features/auth/assessment.md)  
- Requirements: [/features/auth/requirements.md](/features/auth/requirements.md)  
- Cognito config: [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- Runbook: [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md)

Rollout recommendation  
Perform staged rollout behind feature flag `feature.auth.*`: 5% → 25% → 50% → 100% with validation windows and pre-defined rollback triggers between stages.

Service Level Indicators (SLIs) — what to measure
For each flow (email registration, email login, forgot, logout, refresh, Google SSO, Apple SSO) capture:

1. Success rate by flow — percentage of successful transactions in a window.  
2. Error rate by flow — percentage of 5xx and functional errors (e.g., invalid_code) in a window.  
3. Latency (p50 / p95 / p99) — end-to-end auth flow latency (record external IdP latency separately).  
4. Token refresh success rate — successful refresh attempts / total attempts.  
5. SES delivery metrics — delivery rate, bounce rate, complaint rate for verification & reset emails.  
6. PKCE / callback errors — count of state/nonce or code_verifier failures.  
7. Security signals — suspicious patterns: high verification attempts, unusual IP patterns, rate-limit triggers.

Service Level Objectives (SLOs) — proposed targets
(These are proposals; finalize with SRE and Security.)

- Auth flow success rate (per flow): ≥ 99% over 1h window  
- Error rate (per flow): < 1% over a 5–15 min window  
- p95 latency (end‑to‑end, excluding external IdP): < 500 ms  
- Token refresh success rate: ≥ 99.5%  
- SES delivery (verification/reset): ≥ 99% delivery within 60s in dev/stage (prod SLA may differ)  
- PKCE callback errors: < 0.1% of auth attempts

Dashboards — required panels
Each dashboard must have an owner and link stored here after creation.

1. Auth Overview — aggregated success rate, error rate, p95 latency, active sessions, feature flag state.  
2. Flow details — per flow panels (email reg, email login, forgot, refresh, logout, Google SSO, Apple SSO) with success/error rates, latency distribution, recent errors.  
3. SES panel — sent/min, delivery rate, bounce rate, complaints.  
4. Security & Abuse — rate-limit events, repeated failed verification attempts, anomaly counts.  
5. Infra health — Cognito API error rates, throttling, region latency.  
6. Canary/Rollout panel — current exposure %, cohort metrics per canary window.

Alerts — thresholds and actions
Alerts should include runbook link and contact info. Recommended critical triggers:

Critical (immediate rollback)
- Error rate > 5% sustained for 5–10 minutes (aggregate) → toggle feature flag OFF / rollback.  
- p95 latency > 500 ms sustained for 10+ minutes → investigate; consider rollback.  
- SES bounce rate spike > 2% (15 min) → investigate delivery/DNS/DKIM.  
- Confirmed token leakage or exposed secret → trigger security incident process.

High (fast response)
- Error rate 1–5% for 10 minutes → page oncall and investigate.  
- PKCE callback errors spike (10x baseline) → investigate redirect URIs / client config.  
- Repeated verification brute attempts → automated blocking + escalation.

Alert payload must include: timestamp, flow, metric value, region, redacted log snippets, trace link (if available), link to runbook [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md).

Monitoring instrumentation requirements
- Tag events with: flow, environment, hashed user_id, client_platform, app_version, request_id.  
- Tracing: propagate trace ids through Hosted UI callbacks and token exchange.  
- Logs: structured JSON, PII redaction, audit events for register/verify/login/forgot/logout/revoke.  
- Metrics resolution: 1-minute granularity for canary windows; retention per policy.

Data retention & privacy
- Audit logs retained per compliance policy; redact PII in logs.  
- Never log token material (access/refresh tokens must be omitted or hashed).

Roles & responsibilities (placeholders)
- Owner: <owner-placeholder> — maintain doc and finalize SLOs  
- SRE: implement dashboards & alerts  
- Platform/Infra: export Cognito/SES metrics to monitoring system  
- Mobile Lead: provide cohort mapping for canary  
- Security Lead: validate security alerts & thresholds  
- QA: execute smoke tests during canary windows

Canary verification checklist
- [ ] Canary cohort enabled at target percent  
- [ ] Auth Overview: success rate within SLO for cohort  
- [ ] No flow has error rate > agreed threshold  
- [ ] SES delivery within expected bounds  
- [ ] No suspicious abuse indicated on Security panel  
- [ ] QA smoke tests executed and green (see [/features/auth/testing.md](/features/auth/testing.md))

Acceptance criteria — “ready” for rollout
- Dashboards are implemented and links inserted here.  
- Critical alerts configured and tested.  
- Canary panel shows cohort metrics and allows % control.  
- QA and Security have signed off on SLO values and thresholds.

Operational notes
- During canary, maintain a 1:1 cadence between Lead Dev and SRE for metric reviews.  
- Use a dedicated channel (e.g., #oncall-auth) and incident Jira template.  
- Record metric snapshots and decisions in the post-deploy report [/reports/auth-post-deploy.md](/reports/auth-post-deploy.md).

Change log
- YYYY-MM-DD — Draft created by <owner-placeholder>  
- YYYY-MM-DD — Reviewed by SRE / Security

# Deployment Runbook — Authentication Feature

Purpose  
This runbook describes deployment, verification, and rollback procedures for the Authentication feature (Cognito Hosted UI, email flows, Google/Apple SSO). Use this during staged rollout and incident response. Follow assessment / ADR / docs referenced below before executing

Required references
- Assessment: [/features/auth/assessment.md](/features/auth/assessment.md)  
- ADR: [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md)  
- Cognito configuration: [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- Security & standards: project central standards (Security, OAuth §Y, Observability §O)

Contacts & RACI
- Owner / Executor: Lead Dev — Andrey Bakaev
- Platform/Infra: AWS / SES provisioning
- Mobile Lead: client verification (PKCE / secure storage)
- SRE: dashboards & alerts, rollback execution
- Security Lead: threat signoff
- Oncall escalation: paging list / Slack #oncall-auth

Pre-deploy checks (must be green)
- [ ] Assessment & ADR referenced in PR and approved [/features/auth/assessment.md](/features/auth/assessment.md), [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md) [1]
- [ ] All CI gates passed: unit, integration, e2e (critical), SAST, dependency & secrets scan, IaC scan, docs linkcheck
- [ ] Dev & staging Cognito User Pools exist and mirror prod config schema
- [ ] SES verified identitites for sending emails (dev/stage/prod)
- [ ] Google & Apple IdP configured in dev/stage and smoke-tested
- [ ] Feature flag exists: `feature.auth.*` and can be toggled by release engineer
- [ ] Dashboards created & linked (auth success rate, error rate, p95 latency, SES bounce rate)
- [ ] Runbook and rollback procedures reviewed with SRE & Security
- [ ] Test accounts created (list in Appendix)

Deployment strategy
- Gate: feature flag `feature.auth.*` controls behavior in prod (OFF by default). Perform staged rollout behind this flag. Follow progressive exposure: 5% → 25% → 50% → 100% with verification windows between steps [1].
- Do not proceed to next stage until verification window passes and metrics are within thresholds.

Metric thresholds / rollback triggers (use these as immediate decision criteria)
- Success goal during rollout windows:
  - Error rate < 1% (auth failures excluding expected 4xx)  
  - p95 latency < 500 ms (excluding external IdP latency)
- Immediate rollback triggers (if observed during stage):
  - Error rate spike above 5% over 5–10 minutes  
  - p95 latency > 500 ms sustained for 10+ minutes  
  - Critical security incident (e.g., token leakage, secrets exposure)  
  - Major functional regression (registration/verify/login/forgot failing at scale)
- Note: thresholds may be tuned by Security/SRE prior to rollout; record final thresholds in this runbook.

Deployment steps — Staging (validation)
1. Ensure staging environment is up-to-date with the release candidate.  
2. Toggle `feature.auth.*` ON in staging (or enable subflags if available).  
3. Run smoke tests (see Smoke Tests below) — must pass.  
4. Validate dashboards: auth success rate, login/verify latency, SES delivery metrics.  
5. If issues — fix in branch and redeploy to staging. Do not continue.

Deployment steps — Production (canary → full)
1. Pre-deploy: Ensure all pre-deploy checks are green and stakeholders notified (Slack + email).  
2. Deploy code & infra changes to production (DB migrations first if any). Follow normal deploy pipeline.  
3. Set feature flag to Canary 5% (or create canary group). Document time and settings in release notes.  
4. Wait for verification window: 15–30 minutes (monitor continuous).  
   - Run smoke tests.
   - Monitor metrics and logs (see Observability).  
5. If metrics OK → increase to 25%. Wait verification window (30–60 minutes).  
6. If metrics OK → increase to 50%. Wait verification window (60 minutes).  
7. If metrics OK → increase to 100%.  
8. If at any stage a rollback trigger fires → execute rollback procedure immediately.

Smoke tests (manual + automated)
- Use dedicated test accounts listed in Appendix.
- Execute the following flows (expected result noted):
  1. Register new user (email) → receive 6-digit code → verify → receive tokens (id/access/refresh) — PASS: tokens valid, session active.  
  2. Login existing user (email+password) — PASS: tokens returned, valid claims.  
  3. Forgot password → receive code → reset password → login — PASS: reset completes; previous refresh tokens revoked.  
  4. Logout → local storage cleared + GlobalSignOut invoked — PASS: refresh token invalid.  
  5. Google SSO via Hosted UI (dev/configured redirect) — PASS: tokens issued, profile attributes mapped.  
  6. Apple Sign-in via Hosted UI (iOS scenario) — PASS: tokens issued; private relay email handled.  
  7. Token refresh flow — PASS: refresh returns new access_token; rotation/expiry behavior validated.  
  8. PKCE verification: try a bad code_verifier → hosted UI rejects — PASS: rejected.  
  9. SES: check verification / reset email deliverability and correct template rendering.  
- Record timestamps, logs, and metric snapshots for each test.

Rollback procedure (fast)
1. Immediate action: toggle `feature.auth.*` OFF (or revert to previous flag percentages / configuration).  
2. If the issue is code-related and quick revert available, redeploy previous stable release. Otherwise, disable feature flag to stop traffic flows.  
3. If Cognito config change caused issue (e.g., wrong redirect URIs), revert Cognito changes or point Hosted UI to previous domain/config.  
4. Inform stakeholders (Slack channel, oncall) and open an incident ticket in Jira.  
5. SRE: capture logs, traces, and metrics for post-mortem.  
6. Do not reattempt rollout until root cause analysis completed and fixes validated in staging.

Post-rollback steps
- Triage: Lead Dev + SRE + Security to analyze logs and reproduce in staging.  
- Plan: Create Jira issues for root cause + fixes; assign owners and target dates.  
- Re-test: Apply fixes to staging, run full smoke suite. Only after green can a new rollout be scheduled.

Post-deploy validation & acceptance
- Confirm all smoke tests passed at 100% rollout.  
- Confirm SRE dashboards show stable metrics for at least 60 minutes post-100% deployment.  
- Security: ensure no token leaks, no PII in logs, secrets not exposed.  
- Update documentation: mark feature status, update /runbooks/deployment-auth.md with actual timestamps and decisions.  
- Publish release notes with links to PRs, CI evidence, SAST/DAST reports, and dashboards.

Observability & logs (where to look)
- Auth success/failure metrics: [link to SRE dashboard]  
- Latency: p50/p95/p99 for auth endpoints — monitor by flow (email, Google SSO, Apple SSO)  
- SES: delivery/bounce/complaint metrics  
- Security scans: SAST/DAST reports in CI artifacts  
- Application logs: search for "auth:error", "token:revoke", "cognito:callback" events  
- Traces: distributed traces for auth flows (correlate by request id)

Escalation matrix
- 0–15 min: Lead Dev + SRE investigate; attempt quick rollback if severe.  
- 15–60 min: Pull Security Lead and Product Owner for impact assessment.  
- >60 min or data breach: escalate to CTO and trigger full incident process (human-only mode criteria apply) [1].

Operational notes & tips
- Always test Hosted UI flows on real devices for Apple + Google; emulators sometimes behave differently.  
- Keep test accounts with multiple states (unverified, verified, with old refresh tokens).  
- Do not expose client_secret in mobile builds — verify builds are not leaking secrets.  
- When toggling flags, document who toggled, when, and why.

Appendix A — Test accounts (examples)
- dev+1@example.com — password: TempPass!123 — verified  
- dev+2@example.com — password: TempPass!123 — unverified  
- sso-google-test@example.com — Google test account  
- sso-apple-test@privaterelay.appleid.com — Apple test account (private relay)

Appendix B — Useful commands (examples)
- Toggle feature flag (example — replace with your flag service CLI):
  - curl -X POST https://flags.example.com/api/toggle -d '{"flag":"feature.auth.*","state":"off"}' -H "Authorization: Bearer $FLAG_TOKEN"
- Cognito global sign out (AWS CLI):
  - aws cognito-idp admin-user-global-sign-out --user-pool-id <pool-id> --username <username>
- Revoke token via Cognito (if applicable):
  - aws cognito-idp revoke-token --token <refresh_token> --client-id <client_id>

Appendix C — Decision log
- Record time-stamped decisions during rollout: who approved, what percent, metric snapshots, any deviations.

References & further reading
- Diataxis delivery patterns and staged rollout guidance
- [/features/auth/assessment.md](/features/auth/assessment.md)  
- [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md)  
- [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)

Document change history
- YYYY-MM-DD — Draft created by TDA
- YYYY-MM-DD — Reviewed by SRE / Security

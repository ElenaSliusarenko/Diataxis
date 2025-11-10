# Authentication — Testing & Smoke Checklist

## Purpose
This document defines smoke tests, e2e scenarios, automated test requirements and CI gating criteria for the Authentication MVP. Use it to validate dev/stage builds before any staged rollout.

## References
- Assessment: [/features/auth/assessment.md](/features/auth/assessment.md)  
- Requirements: [/features/auth/requirements.md](/features/auth/requirements.md)  
- Cognito config: [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- Runbook: [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md)

## Test environments
- dev: full feature set ON for dev accounts  
- stage: mirror prod config; feature flag OFF by default, enable for smoke  
- prod: feature flag OFF until staged rollout

## Test accounts (examples — create before testing)
- dev_verified@example.com — verified, password: TempPass!123  
- dev_unverified@example.com — unverified  
- sso_google_test@example.com — Google test account  
- sso_apple_test@privaterelay.appleid.com — Apple test account

## Smoke tests (manual / quick)
Each smoke step: expected result, pass/fail, logs to collect.

1. Register (email)
   - Steps: open Hosted UI / registration flow → fill email/password → submit  
   - Expect: verification email sent; code delivered to inbox; code verifies; tokens issued (id/access/refresh)  
2. Verify (email code)
   - Steps: input 6‑digit code  
   - Expect: verification success; user becomes confirmed; tokens valid  
3. Login (email + password)
   - Expect: id/access/refresh tokens returned; status 200; audit log entry  
4. Forgot password
   - Steps: request reset → receive code → set new password → login  
   - Expect: previous refresh tokens invalidated; new tokens issued  
5. Logout
   - Steps: app calls logout → Cognito revoke/GlobalSignOut  
   - Expect: refresh token invalid; subsequent refresh attempts fail  
6. Token refresh
   - Steps: use refresh token to get new access token  
   - Expect: new access_token returned; if rotation enabled, old refresh token invalidated per policy  
7. Google SSO (Hosted UI)
   - Expect: Hosted UI flow completes; tokens issued; profile mapped  
8. Apple Sign‑in (Hosted UI)
   - Expect: flow completes on iOS device; private relay email handled  
9. PKCE negative test
   - Steps: attempt auth with invalid code_verifier  
   - Expect: flow rejected; no tokens issued  
10. SES deliverability
    - Verify verification/reset emails are delivered and templates render correctly

## End‑to‑End automated test scenarios (for CI)
- e2e-001_register_verify_login_should_succeed  
- e2e-002_forgot_password_flow_should_reset_and_invalidate_old_tokens  
- e2e-003_logout_should_revoke_refresh_tokens  
- e2e-004_token_refresh_rotation_and_revoke  
- e2e-005_google_sso_flow  
- e2e-006_apple_sso_flow (run on real device or staging Apple config)  
- security-001_pkce_replay_protection  
- security-002_rate_limit_verification_attempts

Each automated test must assert:
- HTTP status codes, token structure (presence of required claims), audit log events, and no leakage of secrets in logs.

## CI gating requirements
- Run e2e smoke suite on PRs targeting main/rc branch (or run on commit to feature branch, per policy)  
- Required checks to pass before merge to release branch:
  - Unit tests
  - Integration tests (auth middleware)
  - Critical e2e suite (above)
  - SAST & secrets scan
  - Docs build & linkcheck (docs must compile)

## Test data & environment setup
- How to prepare dev Cognito: use configuration in [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- Provide test credentials in secrets manager accessible to CI (rotate regularly)  
- For SSO tests register test apps at Google/Apple with dev redirect URIs

## Observability checks during tests
- Watch metrics: auth success rate, auth error rate, p95 latency, SES bounce rate  
- Capture traces for failed flows (trace id in logs)  
- Save snapshots of dashboards as evidence for runbook

## Failure handling
- On failure in dev: create Jira bug, tag severity, link to failing test logs, assign owner placeholder  
- On failure in stage during canary: follow runbook rollback steps (see [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md))

## Acceptance criteria for QA sign‑off
- All smoke tests PASS on stage with feature flag enabled for staging group  
- Automated e2e suite green in CI for release candidate  
- SAST & secrets scan show no critical findings  
- Observability dashboards show expected baselines (error rate < threshold, p95 < threshold)

## Notes & troubleshooting
- Private relay emails from Apple may not deliver user's real email — use test handling per Cognito mapping  
- For local dev e2e, mock external IdP responses where possible, but run at least one end‑to‑end SSO test against real provider in staging

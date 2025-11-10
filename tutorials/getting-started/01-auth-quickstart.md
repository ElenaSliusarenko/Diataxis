# Auth Quickstart — Getting Started

TL;DR  
This quickstart shows how to set up and validate the Authentication MVP locally / on dev: Cognito User Pool + Hosted UI, PKCE for mobile, email verification flow, and basic smoke tests. It is intended to get a working dev environment and run the core E2E flow quickly. Follow Phase‑0 assessment and the feature‑flag staged rollout practices before any production rollout [/features/auth/assessment.md](/features/auth/assessment.md) [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md).

## Goals
- Provision or point to a dev Cognito User Pool and App Client (PKCE enabled).  
- Verify email registration + 6‑digit verification, login, forgot password, token refresh, and logout flows.  
- Validate SSO via Hosted UI (Google / Apple) in dev.  
- Produce a short test log / screenshots to attach to the release PR.

## Prerequisites
- AWS access to the dev account or an infra IaC module for Cognito (see [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)).  
- SES (or test SMTP) verified for dev environment.  
- Mobile app (or browser) configured to use the dev App Client redirect URIs (see mobile integration guide) [/features/auth/mobile-integration.md](/features/auth/mobile-integration.md).  
- Feature flag control and a way to toggle `feature.auth.*` in dev.  
- Test accounts and test email inboxes (see [/features/auth/testing.md](/features/auth/testing.md)).

## Quickstart checklist (high level)
- [ ] Create or reuse dev Cognito User Pool and App Client (PKCE enabled).  
- [ ] Configure Hosted UI domain and redirect URIs.  
- [ ] Verify SES templates for verification/reset.  
- [ ] Configure Google / Apple IdP in dev Cognito.  
- [ ] Configure mobile/web client redirect URIs and code_challenge handling.  
- [ ] Run core smoke scenario: register → verify → login → refresh → logout.

## 1) Provision / point to dev Cognito (IaC or Console)
Recommended (IaC): apply your Terraform/CloudFormation module that creates:
- User Pool: name `{project}-user-pool-dev`  
- App Clients: `{project}-ios-client-dev`, `{project}-android-client-dev`, `{project}-web-client-dev`  
- Hosted UI domain or custom domain

If manual (Console):
1. Open AWS Cognito → Manage User Pools → Create new pool (or use existing dev pool).  
2. Under App clients: create client for mobile with Authorization Code grant. DO NOT generate client_secret for mobile; enable PKCE.  
3. Under App client settings → Hosted UI: configure redirect URIs (exact match), allowed OAuth flows (Authorization code grant), and scopes (openid email profile).  
4. Under Message customizations / SES: verify SES templates for verification & reset.

Document created values / placeholders in your local config:
- USER_POOL_ID: <COGNITO_POOL_ID_DEV>  
- APP_CLIENT_ID: <COGNITO_APP_CLIENT_ID_DEV>  
- HOSTED_UI_DOMAIN: https://<dev-domain>.auth.<region>.amazoncognito.com

(Record these in `/reference/integration/cognito-configuration.md` or local env vars.)

## 2) Configure IdPs (Google / Apple) for dev
- Google: create OAuth client in Google Console; add dev redirect URIs; copy client_id → Cognito IdP config.  
- Apple: register Service ID, generate key, add Team ID; configure in Cognito and add redirect URIs for iOS flows.
- Ensure mapping of attributes (email, name) in Cognito attribute mapping.

## 3) Mobile / Web client setup
- Mobile: implement PKCE flow per mobile guide [/features/auth/mobile-integration.md](/features/auth/mobile-integration.md). Register app schemes/universal links in Cognito redirect URIs.
- Web: configure redirect URL that returns to a test callback page which logs code for manual exchange if needed.
- Ensure apps do NOT embed client_secret.

## 4) Run the core manual E2E (smoke) flow
Use a dev test email (e.g., dev_user@example.com) and an inbox you control.

A) Register + Verify (Hosted UI)
1. Open Hosted UI URL:
   - https://{HOSTED_UI_DOMAIN}/signup?client_id={APP_CLIENT_ID}&response_type=code&scope=openid+email+profile&redirect_uri={REDIRECT_URI}&state={RND}
2. Complete sign-up in Hosted UI with email + password.
3. Check dev inbox for 6‑digit verification code (SES template).
4. Enter code in Hosted UI (or via ConfirmSignUp API) to confirm user.

B) Exchange code for tokens (if using direct token exchange)
- If using client-side exchange with PKCE, the app will exchange authorization code + code_verifier at:
  - POST https://{HOSTED_UI_DOMAIN}/oauth2/token
  - Form data: grant_type=authorization_code, code={code}, client_id={APP_CLIENT_ID}, redirect_uri={REDIRECT_URI}, code_verifier={code_verifier}

C) Login (existing user)
- Reopen Hosted UI sign‑in flow or initiate login flow in app. On success you should receive id_token / access_token / refresh_token.

D) Forgot password
- Use Hosted UI or Cognito API to request password reset → receive 6‑digit reset code → confirm reset → login.

E) Token refresh
- Use refresh_token to obtain a new access_token; verify rotation/revoke behavior if configured.

F) Logout
- Client should call Cognito revoke or GlobalSignOut and wipe secure storage; confirm refresh_token is invalid afterwards.

Record the outputs (screenshots, token claims redacted, timestamps) for PR evidence.

## 5) Automated quick checks (curl examples)
(Replace placeholders with actual values; do NOT use production secrets.)

- Exchange code for token (example):
```
curl -X POST "https://{HOSTED_UI_DOMAIN}/oauth2/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&client_id={APP_CLIENT_ID}&code={CODE}&redirect_uri={REDIRECT_URI}&code_verifier={CODE_VERIFIER}"
```

- Refresh token:
```
curl -X POST "https://{HOSTED_UI_DOMAIN}/oauth2/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=refresh_token&client_id={APP_CLIENT_ID}&refresh_token={REFRESH_TOKEN}"
```

- Revoke (example):
```
curl -X POST "https://{HOSTED_UI_DOMAIN}/oauth2/revoke" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "token={REFRESH_TOKEN}&client_id={APP_CLIENT_ID}"
```

## 6) Validation & testing
- Run the test cases in [/features/auth/testing.md](/features/auth/testing.md). Ensure smoke passes before marking dev task Done.  
- Attach screenshots/logs and links to test artifacts to the Jira ticket and PR (use internal References per process).

## 7) Troubleshooting (quick)
- No email: check SES identity, sandbox mode, spam folder.  
- PKCE errors: verify code_challenge is SHA256(base64url(code_verifier)), no extra padding or whitespace.  
- Redirect mismatch: confirm exact redirect_uri registration (no trailing slash).  
- Token exchange errors: check time skew between client and server (NTP).

## 8) Next steps
- After dev validation, mirror config in staging and run the same E2E with staging IdPs.  
- Prepare runbook evidence and SLO dashboard links before any production flag changes [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md) [/observability/auth-dashboards.md](/observability/auth-dashboards.md).  
- Keep `feature.auth.*` OFF in prod until staged rollout per runbook.

## References (internal)
- [/features/auth/assessment.md](/features/auth/assessment.md)  
- [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- [/features/auth/mobile-integration.md](/features/auth/mobile-integration.md)  
- [/features/auth/testing.md](/features/auth/testing.md)  
- [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md)  
- [/observability/auth-dashboards.md](/observability/auth-dashboards.md)

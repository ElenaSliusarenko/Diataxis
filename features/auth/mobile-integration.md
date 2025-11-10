# Mobile Integration Guide — Authentication

## Summary
This document provides concrete, platform‑specific guidance for integrating mobile clients with AWS Cognito (Hosted UI + Authorization Code with PKCE). It covers App Client settings, redirect URI formats, PKCE sequence, secure storage recommendations (Keychain / Keystore), negative test cases, and an on‑device verification checklist. Follow Phase‑0 assessment and feature‑flag rollout practices before production rollout [/features/auth/assessment.md](/features/auth/assessment.md) [1].

## Purpose
- Give mobile engineers an unambiguous checklist to implement and verify auth flows (email/password via Hosted UI, Google SSO, Apple Sign‑in).
- Ensure security best practices (PKCE, no client_secret in apps, secure token storage).
- Provide test scenarios required for staging and canary rollout.

## Scope
- Mobile platforms: iOS (native / Swift), Android (native / Kotlin) and cross‑platform (React Native / Flutter) — platform snippets are illustrative.
- Flows covered:
  - Authorization Code Grant with PKCE (primary)
  - Hosted UI sign-up / sign-in (email/password)
  - Google SSO and Apple Sign‑in through Cognito Hosted UI
  - Token refresh using refresh_token stored securely on device

## References (internal)
- Assessment: [/features/auth/assessment.md](/features/auth/assessment.md)  
- Requirements: [/features/auth/requirements.md](/features/auth/requirements.md)  
- Cognito configuration: [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- Testing checklist: [/features/auth/testing.md](/features/auth/testing.md)  
- Runbook: [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md)

## App Client Configuration (Cognito)
For each environment (dev/stage/prod) create App Clients as described in the Cognito config reference. Key items for mobile clients:
- Grant type: Authorization code grant
- PKCE: REQUIRED (do not use implicit flow) — mobile clients must enforce PKCE [1]
- Client secret: DO NOT generate or store in mobile apps
- Redirect URIs: register exact app scheme and hosted UI redirect URIs (see Redirect URI section)
- Token TTLs: follow values agreed in requirements and ADR (documented in [/features/auth/requirements.md](/features/auth/requirements.md))

## PKCE: End-to-End Sequence (High level)
1. Client generates a random code_verifier (high entropy, e.g., 32+ bytes, URL‑safe).  
2. Client derives code_challenge = BASE64URL(SHA256(code_verifier)).  
3. Client opens Cognito Hosted UI authorization endpoint with:
   - response_type=code
   - client_id={app_client_id}
   - redirect_uri={app_redirect_uri}
   - code_challenge={code_challenge}
   - code_challenge_method=S256
   - state={opaque_state}
   - scope={openid email profile ...}
4. User authenticates (email/password or SSO) in Hosted UI; Cognito redirects back to app with code & state.  
5. Client sends code + code_verifier to secure backend token exchange endpoint OR directly to Cognito token endpoint (if architecture allows) to exchange for tokens. Prefer server-side exchange when client_secret required; if exchange happens on device, use PKCE (no client_secret).  
6. On success, receive id_token + access_token + refresh_token (as configured). Store refresh_token securely; keep access/id tokens in memory or short‑lived storage.

Notes:
- Always validate state and nonce returned by Hosted UI before exchanging code.
- Use S256 code_challenge_method (do not use plain).

## Redirect URI patterns
- iOS (custom scheme): myapp://auth/callback  
- iOS (Universal Links): https://auth.myapp.dev/auth/callback (preferred for security)  
- Android (intent filter): com.myapp://auth/callback or https universal links  
- Hosted UI redirect (web fallback): https://{dev-hosted-ui-domain}/oauth2/idpresponse

Register exact URIs in Cognito — no wildcards. For iOS and Android, register both the app scheme and the Hosted UI callback as required by platform SSO flows.

## Secure Storage of Tokens
- Refresh tokens: store ONLY in secure OS keystore:
  - iOS: Keychain (use access control set to whenUnlockedThisDeviceOnly or biometry if chosen), prefer Keychain with kSecAttrAccessibleWhenUnlockedThisDeviceOnly for non‑biometric storage.
  - Android: EncryptedSharedPreferences backed by Android Keystore or use the Jetpack Security library / StrongBox where available.
- Access / ID tokens: keep in memory and refresh frequently; avoid persistent storage when possible.
- Biometric unlock (optional UX): use biometric prompt to unlock Keychain/Keystore for refresh token access — do NOT send biometric data to server.
- Never log tokens or include them in crash reports. Redact tokens in logs.

## Build & CI notes (safety)
- Do NOT embed client secrets in app code, resource files, or CI variables accessible to builds. Replace any secret values with placeholders in docs and IaC (e.g., <COGNITO_CLIENT_SECRET_REMOVED>).
- Ensure release builds strip debug logging and do not include developer/QA accounts.
- CI should run static analyzer to detect accidental secrets and fail builds if tokens/secrets are present.

## Token Handling & Refresh Strategy
- Use short‑lived access_token (e.g., 10–15 min) and refresh_token rotation per policy.
- On token refresh:
  - Use refresh_token from secure storage to request new access_token.
  - Implement refresh token rotation (if supported) and revoke the old refresh token on logout or account changes.
- On logout:
  - Call Cognito GlobalSignOut or revoke endpoint.
  - Wipe local secure storage immediately.

## Error Handling & Negative Test Cases
Test and handle:
- network failures during redirect or token exchange
- invalid or expired authorization code
- mismatched state or missing nonce
- PKCE verification failure (invalid code_verifier)
- refresh token revoked or expired
- email verification flow (unverified users cannot exchange tokens until verified)
- SSO provider failures (Google/Apple downtime, private relay email cases)

For each error case, ensure user sees a clear, non‑PII error message and audit/log event is generated (redacted).

## Testing Checklist (on‑device / E2E)
Perform these tests on real devices and against staging Cognito:

Manual / Smoke:
- Register via Hosted UI (email) → receive 6‑digit code → verify → tokens issued
- Login via Hosted UI (email/password)
- Forgot password flow via Hosted UI → reset → login
- Google SSO via Hosted UI (full redirect on device)
- Apple Sign‑in on iOS (handle private relay email)
- Logout → confirm refresh token invalidation
- Attempt auth with invalid code_verifier → expect rejection

Automated:
- e2e-001_register_verify_login_should_succeed
- e2e-002_forgot_password_flow_should_reset_and_invalidate_old_tokens
- e2e-003_logout_should_revoke_refresh_tokens
- security-001_pkce_replay_protection

See full test cases in [/features/auth/testing.md](/features/auth/testing.md).

## Developer snippets (pseudo)
- Generate code_verifier:
  - use secure random bytes, base64url encode, length >= 43 chars
- Compute code_challenge:
  - SHA256(code_verifier) → base64url encode
- Store refresh_token:
  - iOS: Keychain.set("refresh_token", token)
  - Android: EncryptedSharedPreferences.putString("refresh_token", token)

(Provide real code snippets per platform in a follow‑up if needed.)

## Acceptance Criteria (mobile integration)
- PKCE flow implemented and verified end‑to‑end on staging for iOS and Android [1].  
- No client_secret present in mobile builds or logs.  
- Refresh token stored in secure storage and is revoked on logout.  
- E2E smoke tests from testing checklist pass on staging.  
- Mobile integration doc added to repo and linked from assessment/requirements.

## Rollout considerations
Follow feature flag staged rollout for production exposure: 5% → 25% → 50% → 100% and use canary windows defined in runbook and observability docs [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md) [/observability/auth-dashboards.md](/observability/auth-dashboards.md) [1].

## Appendix — Troubleshooting quick tips
- PKCE rejects: verify code_challenge derived exactly from stored code_verifier (check base64url variants).  
- Redirect mismatch: confirm exact redirect URI registration (no trailing slash or wildcard).  
- Apple private relay: request the email attribute handling strategy in Cognito and test with private relay addresses.  
- SES email non‑delivery: check SES sandbox status, DKIM/SPF, and bounce logs.

# How to: Login with Email

Summary  
This how‑to describes the user and developer steps to authenticate an existing user using email + password, validate tokens, handle common errors, and perform secure logout. Use this document for QA smoke tests, mobile integration verification, and support troubleshooting. Follow the project's documentation patterns (assessment‑first, reference‑first) and the staged rollout process [1].

Prerequisites
- Cognito User Pool and App Client configured for the target environment (see /reference/integration/cognito-configuration.md).  
- Mobile app configured for PKCE flows (see /features/auth/mobile-integration.md).  
- Feature flag for the login flow (example: `feature.auth.email_login`) available and OFF in production until rollout.  
- Test accounts and test environment prepared (see /features/auth/testing.md).

User flow (high level)
1. User opens the app or Hosted UI and selects "Sign in".  
2. User submits email and password.  
3. Client initiates authentication via Hosted UI or Cognito API (Authorization Code + PKCE or appropriate flow).  
4. On success, Cognito returns id_token, access_token and refresh_token.  
5. Client stores refresh_token securely and uses access_token for API calls.  
6. On logout, client revokes tokens (GlobalSignOut) and clears secure storage.

Step‑by‑step (developer / mobile notes)
- Step 0 — Environment checks:
  - Confirm App Client settings: Authorization Code Grant enabled; PKCE required for mobile clients; client_secret NOT embedded in mobile apps (/reference/integration/cognito-configuration.md).
- Step 1 — Input validation:
  - Validate email format and basic password constraints client‑side; do not rely only on client validation.
- Step 2 — Initiate auth:
  - For Hosted UI: open authorization endpoint with required params (client_id, redirect_uri, scope, state, code_challenge).  
  - For direct API (if using a backend/BFF): send credentials to BFF which exchanges/validates with Cognito; prefer server-side token exchange to avoid exposing logic.
- Step 3 — Handle success:
  - On receiving tokens: keep access_token in memory, store refresh_token in Keychain/Keystore.
  - Validate ID token claims (aud, exp, issuer) according to your token validation library.
- Step 4 — Handle failures:
  - Wrong credentials → show generic error (do not reveal whether account exists).  
  - Account unconfirmed → prompt to re-send verification code (link to register/verify how‑to).  
  - User locked out / too many attempts → show retry/backoff UX and log for audit.
- Step 5 — Token refresh & session maintenance:
  - Use refresh_token to obtain new access_token when expired.  
  - Implement refresh rotation and handle refresh revoke flows per requirements.
- Step 6 — Logout:
  - Call Cognito revoke or admin global sign-out (as appropriate).  
  - Wipe refresh_token from secure storage and clear in-memory tokens.

Error handling & UX guidance
- Always show non‑PII, generic messages for auth failures to avoid account enumeration.  
- For network errors, offer retry and offline guidance; do not silently swallow errors.  
- On receiving "user not confirmed", provide a clear flow to re‑send verification (link to /how-to/onboarding/register-with-email.md).  
- For persistent failures, capture redacted logs and correlation IDs for SRE investigation.

Security notes
- PKCE mandatory for native clients; never include client_secret in mobile builds.  
- Do not log token values or sensitive payloads; redact in logs.  
- Enforce rate limits for login attempts and account lockout/backoff policies on server side.  
- Validate tokens server‑side before granting access to protected resources.

Testing checklist (QA / Automation)
- [ ] Successful login (correct email/password) returns id/access/refresh tokens.  
- [ ] Invalid credentials produce generic error and increment rate-limit counters.  
- [ ] Unconfirmed account path: request re-send verification code and successful confirm.  
- [ ] Account lockout or progressive delay after configurable failed attempts.  
- [ ] Token refresh flow: refresh token yields new access token; old refresh token invalidation behavior per policy.  
- [ ] Logout revokes tokens and clears secure storage.  
- [ ] PKCE negative test: tampered code_verifier or state results in rejection.  
- [ ] Observability: login success/failure events emitted to audit logs and metrics.

Acceptance criteria (minimum for Done)
- End‑to‑end happy path green in staging e2e tests.  
- Security checks green (SAST, dependency & secrets scan) on the release branch.  
- No client_secret in mobile builds or CI artifacts.  
- Observability: login success and error metrics available on dashboards as defined in /observability/auth-dashboards.md.  
- Feature flag `feature.auth.email_login` exists and is toggleable per environment.

Support / Troubleshooting (quick)
- If users report cannot sign in:
  - Verify backend auth logs for correlation ID; check token exchange errors.  
  - Check Cognito console for user status (CONFIRMED/UNCONFIRMED/LOCKED).  
  - Verify SES for verification emails if account unconfirmed.
- If refresh fails:
  - Check token revocation list and any policy changes; confirm correct storage and retrieval of refresh_token on device.
- If PKCE mismatch:
  - Verify code_challenge generation and URL‑safe base64 encoding; ensure no whitespace or encoding changes.

References (internal)
- [/features/auth/assessment.md](/features/auth/assessment.md)  
- [/features/auth/requirements.md](/features/auth/requirements.md)  
- [/features/auth/testing.md](/features/auth/testing.md)  
- [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- [/features/auth/mobile-integration.md](/features/auth/mobile-integration.md)  
- [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md)

Follow assessment‑first and Diátaxis documentation practices for any edits or extensions to this how‑to.

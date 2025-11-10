# How to: Register with Email

Summary  
This how‑to explains the user and developer steps to register a new account using email and password, verify email with a 6‑digit code, and obtain an authenticated session. It follows the project's documentation patterns (assessment‑first, Diátaxis) and links to the canonical integration and testing docs [/features/auth/assessment.md](/features/auth/assessment.md) [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md) [/features/auth/testing.md](/features/auth/testing.md) [1].

Prerequisites
- Cognito User Pool created for the target environment and App Client configured with Authorization Code + PKCE for mobile (see [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)).  
- SES (or chosen email provider) verified for the environment and templates configured for verification/reset emails.  
- Feature flag for the flow (example: `feature.auth.email_registration`) available and OFF in production until rollout.  
- Test accounts and QA test environment prepared (see [/features/auth/testing.md](/features/auth/testing.md)).

User flow (high level)
1. User opens the app (or web) and chooses "Sign up" or "Register".  
2. User enters email and a password that meets the password policy.  
3. Client initiates account creation via Cognito Hosted UI (or custom UI backed by Cognito API).  
4. Cognito (or backend) sends a 6‑digit verification code to the provided email.  
5. User enters the 6‑digit code in the app.  
6. Upon successful verification, Cognito confirms the account and issues id_token, access_token and refresh_token.  
7. Client stores refresh_token in secure storage (Keychain / Keystore) and uses access_token for API calls.

Step-by-step (developer / mobile integration notes)
- Step 0 — Client configuration:
  - Ensure the App Client is set to Authorization Code grant with PKCE for mobile clients. Do NOT embed client_secret into mobile apps. See [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md).
- Step 1 — Collect credentials:
  - Validate email format and password locally according to the configured password policy (min length, complexity).
  - Do not log or store the raw password in client logs.
- Step 2 — Initiate registration:
  - Use Cognito API or Hosted UI to create a user (AdminCreateUser or SignUp flow as per your architecture).
  - If using Hosted UI, redirect user to sign‑up URL with appropriate query params (client_id, redirect_uri, scope, state, code_challenge).
- Step 3 — Send verification code:
  - Cognito will send a configurable 6‑digit code via SES. Confirm that SES template used is the verification template.
- Step 4 — Verify email:
  - Provide an input for the 6‑digit code. On submit, call Cognito ConfirmSignUp (or the equivalent endpoint) with the code and username.
  - Validate and handle common errors: expired code, invalid code, too many attempts.
- Step 5 — Post‑verification:
  - On success, perform an auth flow to obtain tokens (authorization code exchange if using Hosted UI; or AWS Cognito InitiateAuth/RespondToAuthChallenge flows).
  - Store refresh_token in secure storage and access_token in memory.
- Step 6 — Finalize UX:
  - Navigate user to onboarding / home screen.
  - Optionally prompt to enable biometrics (local unlock only) — do not substitute for server‑side auth.

Expected results (for QA)
- User receives verification email within the configured window (e.g., within 60s) and the 6‑digit code is valid for the configured TTL (e.g., 10 minutes).  
- After entering the correct code, the user is confirmed and receives id/access/refresh tokens.  
- Incorrect codes produce generic error messages and do not reveal whether an email is registered (avoid account enumeration leaks).  
- After logout or GlobalSignOut, refresh tokens are invalidated.

Security & privacy notes
- Use PKCE for mobile flows. Never store client secrets in mobile apps.  
- Do not include PII or tokens in logs. Redact sensitive values in client and server logs.  
- Implement rate limits on registration and verification endpoints to mitigate abuse.  
- Protect verification endpoints from brute force (max attempts per code / per IP).  
- Ensure email templates do not include sensitive data and comply with privacy rules.

Troubleshooting (common issues)
- No verification email received:
  - Check SES identity verification and sending quotas; check spam folder.  
  - Confirm the correct email template is used in Cognito configuration.
- Verification code expired:
  - Ensure TTL configuration in Cognito matches expectations and provide a "resend code" UX with rate limiting.
- PKCE / redirect errors:
  - Confirm exact redirect URIs and app bundle/sha fingerprints are registered in Cognito; ensure code_challenge and code_verifier match.
- Token exchange errors:
  - Verify the token endpoint is being called with correct params; check clocks for time skew.

Testing checklist (to mark before merging)
- [ ] Successful sign up → verification code delivered → confirm → tokens issued (happy path).  
- [ ] Attempt with expired code: appropriate error and option to resend.  
- [ ] Invalid code attempts: rate limiting and generic error messages.  
- [ ] Logout clears secure storage and revoke/GlobalSignOut invalidates refresh tokens.  
- [ ] PKCE negative test: invalid code_verifier rejected.  
- [ ] SES deliverability verified in environment (dev/stage).  
- [ ] No secrets or tokens appear in client or server logs.

Acceptance criteria (minimum for Done)
- End‑to‑end happy path green in E2E suite in staging.  
- Security gates green (SAST, dependency scan, secrets scan).  
- How‑to page added to docs (this file) and referenced from Assessment/Requirements.  
- Observability: send events for registration, verification success/failure, and email delivery; dashboards updated as per [/observability/auth-dashboards.md](/observability/auth-dashboards.md).  
- Feature flag `feature.auth.email_registration` exists and can be toggled per environment.

Operational notes
- Feature must be deployed behind `feature.auth.email_registration` and staged gradually per runbook. See [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md).  
- Keep a short post‑deploy observation window after enabling the flag for a cohort (canary) and monitor SLI/SLOs.

References (internal)
- [/features/auth/assessment.md](/features/auth/assessment.md)  
- [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- [/features/auth/testing.md](/features/auth/testing.md)  
- [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md)

Use assessment‑first and Diátaxis documentation patterns when editing or extending this how‑to.

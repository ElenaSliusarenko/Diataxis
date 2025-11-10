# How to: Forgot Password (Password Recovery)

## Summary
This how‑to explains the password recovery flow: how a user requests a reset, receives a 6‑digit verification code by email, sets a new password, and re-authenticates. It includes developer notes, security considerations, smoke test steps, and troubleshooting guidance. Follow Phase‑0 assessment and feature‑flag rollout practices before enabling in production [1].

## Prerequisites
- Cognito User Pool configured with email‑based recovery and SES templates (see [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)).  
- Feature flag `feature.auth.forgot_password` available and OFF in production until rollout.  
- Test accounts and QA staging environment prepared (see [/features/auth/testing.md](/features/auth/testing.md)).  
- Mobile integration guidance (PKCE / secure storage) available: [/features/auth/mobile-integration.md](/features/auth/mobile-integration.md).

## User flow (high level)
1. User taps "Forgot password" on sign‑in screen.  
2. User enters their email address.  
3. Client requests a password reset from Cognito (or via backend/BFF).  
4. Cognito sends a 6‑digit reset code to the verified email via SES.  
5. User enters the code in the app and provides a new password.  
6. Cognito confirms the code and resets the password.  
7. Optionally, previous refresh tokens are invalidated and user is prompted to sign in with new credentials.

## Step‑by‑step (developer / mobile integration)

### Step 0 — Backend / Cognito setup
- Verify the User Pool has password recovery enabled and SES templates configured for the reset email.  
- Ensure rate limiting and anti‑abuse controls are in place for reset requests.

### Step 1 — Request reset (client)
- UI: present "Forgot password" form asking for email.  
- Client POSTs request to backend or invokes Cognito ForgotPassword API (depending on architecture).  
- Include minimal metadata for auditing (no PII in logs).

### Step 2 — Send and deliver code
- Cognito sends 6‑digit numeric code via SES template.  
- The code TTL should be short (recommended 10 minutes); max attempts per code should be configured (e.g., 5 attempts).

### Step 3 — Confirm code and set new password
- Client collects code + new password and calls ConfirmForgotPassword (or equivalent backend endpoint).  
- Validate new password against the policy (minimum length, complexity) before sending; do not log passwords.

### Step 4 — Post‑reset behavior
- On success: invalidate existing refresh tokens for this user (revoke session) or require re‑login to obtain new tokens.  
- Prompt user to sign in with their new password and consider recommending (but not forcing) enabling biometric unlock (local only).

## Security & Privacy considerations
- Rate limiting: enforce per‑email and per‑IP limits on reset requests to prevent abuse.  
- Generic responses: API responses should not reveal whether an email is registered (avoid account enumeration).  
- Code entropy & TTL: use 6‑digit numeric codes but enforce short TTL and attempt limits; consider alphanumeric codes for higher security if needed.  
- Token invalidation: revoke refresh tokens after password change to reduce account takeover risk.  
- Logging: redact PII and never log codes, passwords, or tokens. Audit high‑level events (reset requested, reset succeeded/failed) with redaction.

## UX guidance
- Provide clear feedback: "If this email exists, a reset code has been sent."  
- Offer "Resend code" with exponential backoff and attempt limits.  
- Show helpful error messaging for expired/invalid codes and provide a path to request a new code.  
- Consider showing password strength meter and confirmation requirements for the new password.

## Testing checklist (QA / Automation)
- [ ] Request reset for a verified email → reset email delivered within expected window.  
- [ ] Enter valid code + valid new password → reset succeeds, user can sign in with new password.  
- [ ] Enter expired code → appropriate error and ability to request new code.  
- [ ] Enter invalid code repeatedly → rate limiting / temporary block enforced.  
- [ ] Confirm previous refresh tokens are invalidated after reset (attempt refresh fails).  
- [ ] SES deliverability and template rendering verified in staging.  
- [ ] Logging: reset events are captured in audit stream without sensitive data.

## Edge cases & negative tests
- Unverified accounts requesting reset — ensure UX guides verification path.  
- Multiple concurrent reset requests — validate that the latest valid code logic is enforced (or set behavior per policy).  
- Malformed requests and injection attempts — ensure server‑side validation and sanitization.  
- Device offline during reset — provide UX for retry and state persistence (do not store codes insecurely).

## Troubleshooting (common issues)
- No email received:
  - Check SES configuration, sandbox status, bounce/complaint metrics and spam folders.  
  - Verify the correct template is configured in Cognito.
- Code rejected as invalid:
  - Confirm user entered exact code; check for whitespace trimming.  
  - Verify TTL hasn't expired; allow resend if necessary.
- Refresh still works after reset:
  - Ensure refresh token revocation is executed on successful password reset; investigate token storage and revocation API calls.

## Acceptance criteria (minimum for Done)
- End‑to‑end happy path (request → receive code → confirm → sign in) passes in staging.  
- Security gates green (SAST, secrets scan); no sensitive data in logs.  
- Rate limits applied and validated for abuse scenarios.  
- Observability: reset request/success/failure metrics tracked and visible on auth dashboards.  
- Docs updated: this how‑to linked from requirements, testing, and runbook.

## References (internal)
- [/features/auth/assessment.md](/features/auth/assessment.md)  
- [/features/auth/requirements.md](/features/auth/requirements.md)  
- [/features/auth/testing.md](/features/auth/testing.md)  
- [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md)

Follow the assessment‑first and staged rollout patterns described in project delivery guidance.  

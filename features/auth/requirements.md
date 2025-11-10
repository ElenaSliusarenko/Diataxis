# Authentication & Authorization — Requirements (MVP)

## Summary
Deliver secure authentication and authorization for the mobile application using AWS Cognito (User Pools + Hosted UI). MVP scope: email registration with 6‑digit verification, email login, password recovery (forgot password), logout (revoke), session renewal via refresh tokens, and SSO via Google and Apple. Follow assessment‑first and reference‑first patterns; every ticket and doc must include a References block. [1]

## Scope

In scope (MVP)
- Email registration with 6‑digit verification code
- Email login (existing users)
- Password recovery (forgot password) via email code
- Logout (local wipe + Cognito revoke / GlobalSignOut)
- Session renewal via refresh token stored securely on device
- OAuth SSO: Google and Apple via Cognito Hosted UI (PKCE for mobile)

Out of scope (future increments)
- Phone number registration + SMS verification
- Facebook OAuth
- Two‑factor authentication (TOTP) and backup codes
- Biometric-first authentication (device unlock as UX convenience only)
- Change email / change password from settings (post‑MVP)

## User Stories & Acceptance Criteria

1. Title: Register with email
   - As a: New user
   - I want: to register with email and receive a 6‑digit verification code
   - So that: I can verify ownership and start a session
   - Acceptance criteria:
     - Verification email is sent via SES (or configured email provider) within 60s of registration.
     - Code format: 6 numeric digits; TTL = 10 minutes (configurable).
     - Maximum verification attempts per code: 5.
     - After successful verification the user receives Cognito id_token + access_token + refresh_token.
     - Error messages are generic (no PII leakage) and logged for audit.

2. Title: Login with email and password
   - As a: Existing user
   - I want: to login using email and password
   - Acceptance criteria:
     - Successful login returns id_token, access_token, refresh_token.
     - Failed login returns consistent error codes and increments rate‑limit counters.
     - Account lockout or progressive delays applied after configurable failed attempts.
     - All login attempts are logged to audit stream.

3. Title: Forgot password (password recovery)
   - As a: User who forgot password
   - I want: to request a password reset via email and set a new password
   - Acceptance criteria:
     - Password reset request sends a 6‑digit code to the verified email.
     - Code TTL configurable (suggest 10 minutes).
     - After successful reset, previous refresh tokens are invalidated (revoked) for security.
     - Rate limiting and anti‑abuse measures applied to reset requests.

4. Title: Logout and token revocation
   - As a: Authenticated user
   - I want: to logout and revoke my session
   - Acceptance criteria:
     - Logout triggers local secure storage wipe and Cognito revoke/GlobalSignOut.
     - After logout, previously issued refresh tokens cannot be used to obtain new access tokens.
     - Logout action recorded in audit logs.

5. Title: Sign in with Google
   - As a: User
   - I want: to sign in with Google via Cognito Hosted UI
   - Acceptance criteria:
     - Google IdP registered in Cognito; OAuth flow works end‑to‑end (Hosted UI).
     - User receives tokens equivalent to email flow.
     - Mapped attributes (email/name) populated in user profile.

6. Title: Sign in with Apple
   - As a: User on Apple devices
   - I want: to sign in with Apple via Cognito Hosted UI
   - Acceptance criteria:
     - Apple Sign‑in configured in Cognito; iOS bundle ID / redirect URIs validated.
     - Token exchange and hosted UI flow work end‑to‑end with PKCE for mobile.
     - Email attribute handled per Apple requirements (private relay cases).

## Non‑Functional Requirements (NFRs)
- Security
  - PKCE mandatory for all mobile/native OAuth flows; do not store client_secret on devices.
  - Secrets management in platform vault; no secrets in repo or app binaries.
  - Audit logging for: register, verify, login, forgot, logout, revoke.
- Performance & Availability
  - p95 auth latency: < 500 ms (excluding external IdP latency).
  - Availability target: align with Cognito SLA; aim ≥ 99.9%.
- Tokens & TTL (initial proposals — finalize with Security Lead)
  - access_token: 10–15 minutes
  - id_token: 10–60 minutes (as needed)
  - refresh_token: 7–30 days (rotation & revocation policy required)
- Observability
  - Dashboards for success rate, error rate, latency per flow before rollout.
- Rate limiting & Anti‑abuse
  - Verification/reset attempts throttled per IP and per account.
  - CAPTCHAs or additional mitigations for suspicious traffic.

## Integration Points
- AWS Cognito User Pools (primary IdP) — user lifecycle, tokens, Hosted UI
- Amazon SES (email delivery) or approved email provider for verification and reset
- Mobile clients: Authorization Code + PKCE; secure storage in Keychain (iOS) / Keystore (Android)
- Optional: BFF not required for MVP — clients interact directly with Cognito Hosted UI
- Observability: metrics exported to SRE dashboards (SLOs defined prior to rollout)

## Risks & Mitigations
- Risk: Verification/reset code abuse (enumeration/brute force)
  - Mitigation: rate limits, CAPTCHA, device fingerprint heuristics, monitor spikes.
- Risk: Token theft on device
  - Mitigation: secure storage guidance, short access_token TTL, revoke strategy for refresh tokens.
- Risk: Redirect URI misconfiguration or IdP mismatch
  - Mitigation: strict allowlist of redirect URIs; iOS bundle ID and Android SHA verification; state/nonce checks.
- Risk: Vendor lock‑in to Cognito
  - Mitigation: document migration considerations in ADR; keep contracts and API spec if BFF added later.

## Definition of Ready (DoR) for Implementation Stories
- Phase‑0 assessment completed: [/features/auth/assessment.md](/features/auth/assessment.md) [1]
- ADR accepted: [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md)
- Owners assigned for feature, infra and mobile integration
- SES domain verified or email provider chosen for dev/stage/prod
- Feature flag planned: feature.auth.* (subflags for granular flows)
- Test plan drafted (unit, integration, e2e) and automation owners assigned

## Definition of Done (DoD) for Implementation Stories
- Automated tests (unit/integration/e2e) green for critical flows (register/login/forgot/SSO)
- Security checks passed (SAST, dependency & secrets scan)
- Docs updated: How‑to (register/login/SSO), Reference (Cognito config), Explanation (token model)
- Observability dashboards and alerts configured
- Feature flag available and staged rollout plan documented
- Security Lead and Lead Dev sign‑off recorded

## Testing Requirements
- Happy paths and negative cases for codes (expired, reused, wrong)
- PKCE flow verification and replay protection tests
- Refresh token rotation tests and revoke behavior tests
- Rate limiting and abuse detection test scenarios
- Integration tests for Google and Apple IdPs (staging with real providers where possible)

## Open Questions (to resolve before development)
- Final token TTLs and refresh rotation policy (Security Lead decision)
- SES vs alternative email provider for production (Platform decision)
- Exact user profile attribute mapping from IdPs (Product/UX)
- Branding/custom domain for Hosted UI (Product/Platform)

## References (required)
- [/features/auth/assessment.md](/features/auth/assessment.md)  
- [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md)  
- Project Security / OAuth standards (canonical: see central standards repo) [1]

---

If you want, I can now:
- generate this file as a ready-to-copy Markdown (UTF‑8, LF) and prepare a suggested set of GitHub Issues (one per user story) with prefilled References and DoR, or
- expand any user story into a detailed task breakdown with estimates and owners.

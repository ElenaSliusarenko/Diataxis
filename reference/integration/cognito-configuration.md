# AWS Cognito — Configuration & Integration (dev / stage / prod)

## Summary
This document records the canonical Cognito configuration and integration points for Authentication feature (MVP). It is the reference for Infra, Backend, and Mobile teams to create matching User Pools, App Clients, Hosted UI, IdP connections (Google, Apple), and email templates (SES). See assessment and ADR for rationale: [/features/auth/assessment.md](/features/auth/assessment.md) and [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md).

## Environments
- dev: cognito-{project}-dev
- stage: cognito-{project}-stage
- prod: cognito-{project}-prod

Each environment must have its own User Pool and App Clients, or be version/namespace separated to avoid cross-environment leakage.

## User Pool — high level settings
- Name: {project}-user-pool-{env}
- Attributes:
  - Required: email
  - Optional: given_name, family_name, phone_number
- Password policy:
  - Minimum length: 8
  - Require: upper, lower, number, special (TBD by Security)
- Verification:
  - Email verification via 6-digit numeric code
  - Verification code TTL: 10 minutes (configurable)
  - Max verification attempts per code: 5
- MFA: Disabled for MVP (future increment)
- Account recovery: Email-based (enabled)

## App Clients
Create app clients per platform with explicit settings:

1. Mobile iOS client
   - Name: {project}-ios-client-{env}
   - Allowed grant types: Authorization code grant
   - PKCE: REQUIRED
   - Callback/Redirect URIs: exact bundle identifier scheme(s) and hosted UI redirect
   - Client secret: NOT GENERATED for mobile clients

2. Mobile Android client
   - Name: {project}-android-client-{env}
   - PKCE: REQUIRED
   - Redirect URIs: intent / custom scheme and hosted UI redirect
   - Do not store client_secret in app

3. Web client (if needed)
   - Name: {project}-web-client-{env}
   - Allowed grant types: Authorization code grant, implicit if required (discouraged)
   - Client secret: optional (only for server-side apps)
   - Redirect URIs: https://{dev/stage/prod-domain}/auth/callback

Common app client settings:
- Access token expiration: 10–15 minutes (TBD)
- Refresh token expiration: 7–30 days (TBD)
- ID token expiration: 10–60 minutes (TBD)
- Token revocation: enabled and documented (GlobalSignOut supported)

## Hosted UI
- Domain: use AWS Cognito hosted domain or a custom domain (if brand required)
- Hosted UI must be configured per environment
- Strict redirect URI allowlist; no wildcard redirects
- Login/Sign-up flows: Use Hosted UI for SSO (Google, Apple)
- Branding: align templates with product design (colors, logo) — request from Design

## Identity Providers (IdP)
- Google (OIDC)
  - Register OAuth application in Google Console for each environment
  - Provide client_id to Cognito
  - Map attributes: email → email, name → name
  - Scopes: openid email profile

- Apple Sign In
  - Register Service ID, private key, team ID in Apple Developer
  - Configure in Cognito (note Apple private relay / email relay behavior)
  - Map attributes appropriately; handle cases where email is private relay

- (Future) Facebook — planned for later increment

## Email (SES) integration
- SES identities for dev/stage/prod verified
- Templates:
  - Verification email template — include 6-digit code, TTL, resend instructions
  - Password reset template — include 6-digit code and secure link (if used)
- Sender address: no-reply@{domain} (verified)
- DKIM/SPF: configured and documented
- Bounce/Complaint handling: Wire to audit/logging pipeline

## Security & Best Practices
- PKCE: enforce for all native/mobile clients
- Do NOT store client_secret in mobile apps
- State & nonce: enable and validate for OAuth flows to prevent CSRF/replay
- Redirect URIs: exact-match allowlist; no wildcards
- Token storage:
  - refresh_token: Keychain (iOS) / Keystore (Android)
  - access/id tokens: in-memory (short-lived)
- Token rotation & revoke strategy: plan and document (refresh rotation recommended)
- Logs: redact PII and tokens in logs; audit auth events (register, verify, login, logout, revoke)
- Rate limiting: verification/reset actions must be throttled per IP and per account

## Monitoring & Observability
- Metrics to collect:
  - Registration success rate
  - Email delivery/failure rate
  - Login success/failure rates (per flow: email, Google, Apple)
  - Token refresh failures
  - Latency (p95) for auth flows
- Alerts:
  - Auth error rate spike
  - SES bounce rate spike
  - PKCE or callback errors surge

## Testing / Validation (dev)
- E2E scenario to validate in dev:
  1. Register with email → receive code → verify → receive tokens
  2. Login with email/password
  3. Forgot password → receive code → reset → login
  4. SSO via Hosted UI → Google and Apple
  5. Logout → GlobalSignOut → confirm refresh token invalidation
- Validate PKCE flow end-to-end for mobile apps
- Confirm email templates render in major email clients and do not leak PII

## IaC / Manual Steps
- Prefer IaC (Terraform / CloudFormation) to create User Pool and App Clients
- If manual: record all console steps and capture exported JSON config for reproducibility
- Store IaC in infra repo with variables for env and secrets in secrets manager

## Implementation notes & ownership
- Owner: Platform/Infra (create and verify pools & SES)
- Mobile lead: confirm redirect URIs and app bundle / SHA fingerprints
- Security lead: approve token TTLs and rate limits
- BA/Product: approve email copy and branding

## References
- [/features/auth/assessment.md](/features/auth/assessment.md)  
- [/docs/decisions/ADR-001-authentication.md](/docs/decisions/ADR-001-authentication.md)  
- Project Security / OAuth standards (canonical docs) — link to central standards repo

## Appendix: Quick checklist (for Infra)
- [ ] Create User Pool(s) for dev/stage/prod
- [ ] Create App Clients for iOS/Android/Web with PKCE enabled for mobile
- [ ] Configure Hosted UI domain(s)
- [ ] Register Google & Apple IdP for each environment
- [ ] Verify SES identities and create templates
- [ ] Document redirect URIs, bundle IDs and Android SHA fingerprints
- [ ] Create IaC module or export manual steps
- [ ] Add monitoring metrics and dashboards for auth flows

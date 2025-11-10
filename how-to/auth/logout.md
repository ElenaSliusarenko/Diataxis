# How to: Logout

## Summary  
This how‑to describes secure logout and session revoke procedures for the Authentication feature. It covers client and backend steps, recommended token‑revocation flows (Cognito GlobalSignOut / revoke), secure storage wipe, verification checks, automated and manual smoke tests, and troubleshooting. Follow Phase‑0 assessment and staged rollout patterns prior to production rollout [1].

## References (internal)  
- Assessment: [/features/auth/assessment.md](/features/auth/assessment.md)  
- Requirements: [/features/auth/requirements.md](/features/auth/requirements.md)  
- Cognito config: [/reference/integration/cognito-configuration.md](/reference/integration/cognito-configuration.md)  
- Testing checklist: [/features/auth/testing.md](/features/auth/testing.md)  
- Runbook: [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md)  

## Purpose & Scope  
- Purpose: ensure user sessions are ended safely and refresh tokens are invalidated so stolen tokens cannot be reused.  
- Scope: client‑side secure storage wipe, call to token revoke / GlobalSignOut, audit logging, UI/UX confirmation, and QA verification. Applies to email/password and SSO flows (Google/Apple) via Cognito Hosted UI.

## Prerequisites  
- Cognito User Pool and App Clients configured (see cognito config).  
- Mobile apps implement secure storage (Keychain/Keystore) per mobile integration guide.  
- Feature flag `feature.auth.logout` available; gated in prod per rollout policy [1].  
- Test accounts available for QA (see testing.md).

## High-level user flow  
1. User taps "Logout" (or app triggers logout on signout event).  
2. Client calls local logout routine: wipe in‑memory tokens and securely stored refresh_token.  
3. Client requests server (or calls Cognito) to revoke tokens / perform GlobalSignOut.  
4. Server (or direct Cognito call) revokes tokens and records audit event.  
5. Client confirms logout success and routes user to unauthenticated screen.

## Step-by-step: Client (mobile / web)

### Mobile (recommended)
1. Call local logout handler:
   - Clear in‑memory tokens (access_token, id_token).  
   - Delete refresh_token from secure storage (Keychain / EncryptedSharedPreferences).  
   - Remove any cached user profile/state.
2. Attempt server revoke:
   - If you have a backend BFF: call POST /auth/logout (include device identifier / session id).  
   - If clients call Cognito directly: call the Cognito revoke endpoint or use Hosted UI signout redirect (see Implementation Notes).
3. Redirect user to unauthenticated state (login screen or landing page).
4. Optionally: show confirmation toast and allow feedback.

### Web / Hosted UI
1. Initiate logout via Hosted UI sign‑out endpoint to end the server session and clear cookies.  
2. After redirect back to app, clear local storage and cookies.  
3. Verify session invalidation via a protected API call (should return 401).

## Step-by-step: Server / Backend (if applicable)
- Endpoint: POST /auth/logout
  - Input: user identifier / session id / refresh token (hashed if stored), device id (optional)  
  - Actions:
    1. Validate request and authenticate caller (CSRF/authorization).  
    2. Call Cognito revoke token endpoint OR admin‑user‑global‑sign‑out for targeted user sessions:
       - Example (AWS CLI placeholder):  
         aws cognito-idp admin-user-global-sign-out --user-pool-id <POOL_ID> --username <USERNAME>
       - Or use token revoke API with the refresh token.  
    3. Mark session/device token as revoked in server DB (hashed refresh token table) and emit audit event.  
    4. Return success to client.
- Note: Do not log raw tokens. Store hashed token identifiers if you must index tokens.

## Token Revoke vs GlobalSignOut
- Revoke specific token(s) when you have the token value to revoke.  
- GlobalSignOut invalidates all refresh tokens for the user (useful for account compromise).  
- Prefer server‑side revoke for better auditing and control; clients may call revoke when serverless design requires it.

## UX considerations
- Show a clear confirmation (e.g., “You have been logged out”) and route to login.  
- For SSO flows, if using Hosted UI, consider signout redirects to the IdP’s signout endpoints when required by policy.  
- If biometric auto‑unlock was enabled, inform the user that biometric unlock is a local convenience and requires re‑auth on critical actions.

## Audit & Observability
- Emit an audit event for each logout: {user_id (hashed), event: logout, method: client/server/global_sign_out, device_id, timestamp}.  
- Monitor logout success rate, failed revoke attempts, and unusual spikes in GlobalSignOut events on auth dashboards (see observability docs).

## Testing checklist (QA)
Manual / Smoke:
- [ ] Client logout flow clears in‑memory tokens and secure storage.  
- [ ] After logout, protected API returns 401 for the session.  
- [ ] Server revoke endpoint triggers Cognito token revocation / GlobalSignOut and audit event recorded.  
- [ ] Logout after SSO (Google/Apple) ends session and does not leave server cookies/sessions.  
- [ ] Attempt to use a revoked refresh token to obtain new access_token → should fail.  
- [ ] Network error during logout: client retries local wipe and flags for re‑attempt of revoke; user UX handles transient failure gracefully.

Automated:
- Add e2e test: login → logout → assert 401 on protected endpoint and assert refresh token reject.  
- Add unit tests for logout endpoint logic and token revocation paths.

## Acceptance criteria (minimum for Done)
- Logout clears local tokens and secure storage on client.  
- Refresh tokens are revoked (or GlobalSignOut executed) and cannot be used to obtain new access tokens.  
- Audit events logged for logout and revoke operations.  
- QA smoke tests pass in staging.  
- Docs updated, and PR includes links to runbook, testing checklist, and Cognito config.  
- Feature flag set up and behavior controlled by it in prod.

## Troubleshooting & Common Issues
- "Protected API still returns 200 after logout": verify that token was actually removed from client and server revoke executed. Check audit logs and token revocation status.  
- "Refresh token still works after logout": ensure revoke API was called with the correct token or perform GlobalSignOut. If refresh tokens are rotated, ensure correct token instance was revoked.  
- "Hosted UI cookies persist": for web Hosted UI logout, ensure you call IdP signout endpoints and clear cookies post‑redirect.  
- "Network failure prevented revoke": ensure client still cleared local storage; schedule a server‑side revoke attempt tied to device/session id when connectivity restored.

## Security notes
- Never log full tokens. Log only hashed identifiers / correlation IDs.  
- Use server‑side revoke for stronger guarantees and auditing.  
- Provide GlobalSignOut for suspected account compromise.  
- Ensure logout endpoints require proper authentication to avoid CSRF or malicious logout attempts.

## Implementation snippets & examples (placeholders)
- Server revoke (conceptual HTTP):
  - POST /auth/logout
    - body: { "device_id": "<device-id>", "refresh_token_hash": "<hash>" }
- AWS CLI (example placeholder):
  - aws cognito-idp admin-user-global-sign-out --user-pool-id <POOL_ID> --username <USERNAME>
- Client pseudo:
  - secureStorage.delete("refresh_token"); memory.clearTokens(); callApi("/auth/logout", {...})

## Rollout notes
- Gate changes behind `feature.auth.logout`. Test in dev/stage with small cohort. Use runbook for staged rollout and rollback criteria [/runbooks/deployment-auth.md](/runbooks/deployment-auth.md) [1].

## Appendix — Quick checklist for PR reviewer
- [ ] Code: logout endpoint implemented and tested.  
- [ ] Security: no tokens logged; secrets not included.  
- [ ] Tests: unit + e2e covering logout and revoke.  
- [ ] Docs: this How‑to updated and linked from assessment/requirements.  
- [ ] Observability: audit events emitted and dashboards updated.

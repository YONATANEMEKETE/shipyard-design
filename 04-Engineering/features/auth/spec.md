# Auth — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.1 · §5.11 (account/security) · UX User Flows 1–2
**Technical design:** Excluded by design — produced during this feature's implementation step (data model, API design, system design), driven by this behavioral spec.

---

## 1. What this feature is about

Auth owns **identity**: who the user is, how they prove it, and how that proof stays valid across requests. Authentication (who you are) is strictly separated from authorization (what you can do) — authorization lives in the Members feature.

## 2. What users can do

- Register an account with email + password.
- Verify their email address (required before using the product).
- Sign in with Google or GitHub OAuth.
- Log in and log out.
- Stay logged in across refreshes and days (session).
- Reset a forgotten password.
- Change their password (requires current password).
- Change their account email (requires recent re-authentication + verifying the new address).
- Resend a verification email (rate-limited).
- Sign up for a workspace through an invitation link (invitation acceptance gates on a verified account).

## 3. Main behaviors & actions

### 3.1 Registration & verification
- One account per unique email.
- New email/password accounts start unverified; protected areas are inaccessible until verification.
- Verification links are single-use and expire; expired/used links show a recoverable message with a path forward (resend / login).
- Unverified users can resend (rate-limited) but cannot access product areas.
- OAuth accounts are treated as verified (provider-verified email) — no separate verification step.

### 3.2 Login / OAuth
- Email/password login requires a verified account; unverified users are sent to a verification-pending screen.
- A provider email that matches an existing account resolves to **that** account — no duplicates, no merge UI in MVP.
- OAuth failure (cancel, provider error, missing verified email) returns to the auth screen with a retryable message; **no account or session is created on failure**.

### 3.3 Sessions
- Session cookie survives browser refresh; expired sessions redirect to login.
- Logout invalidates the session immediately, no confirmation.
- Multiple simultaneous sessions (multiple devices) allowed.

### 3.4 Password & email changes
- Password reset: request → email link (single-use, expiring) → set new password. The reset request response is generic — it never reveals whether an email has an account.
- Changing password requires the current password.
- Changing account email requires recent re-authentication **and** verification of the new address before it replaces the old one; the old email stays active until then.

## 4. User flows (high level)

1. **Register:** email + password → verification email → click link → verified → sign in → onboarding or pending invitation.
2. **Sign in:** email/password or OAuth → session → route to dashboard / workspace selection / invitation.
3. **Password reset:** forgot password → generic confirmation → reset link → new password → sign in.
4. **Account email change:** settings → re-authenticate → confirm new address → verify → email swapped.

## 5. Business rules

1. Emails are unique; a user may belong to multiple workspaces (membership ≠ identity).
2. Email/password users require a verified email before any protected access.
3. Provider-verified emails satisfy the verification requirement; identity resolution never creates duplicates.
4. Verification and reset tokens are single-use, expiring, and resendable only with rate limiting.
5. Logout invalidates the session immediately; expired/invalid sessions redirect to login.
6. Password reset responses are generic (no account-existence leak).
7. Email change requires recent re-authentication + verification of the new address.
8. Password change requires the current password.
9. Failed auth flows never leave a half-created account.
10. Authentication never implies authorization — access is decided per workspace by Members.

## 6. Out of scope (MVP)

Passkeys, 2FA, SSO, session management UI, email/push notifications for auth events, account deletion flow, additional OAuth providers.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Session lifetime policy (fixed vs sliding) | Not UX-visible; decide at implementation |
| 2 | Password policy specifics | 8+ chars; confirm if stricter |
| 3 | Multiple sessions cap | None by default |

# DeskLine — Auth Sequence Diagrams

**Status:** design artifact (Auth Flow Design)
**Owner:** Omar
**Depends on:** `requirements.md` (FR-1, FR-1a–c), `decisions.md` (D1, D7, D13, D14)

---

## 0. How this doc is organized

Two sequence diagrams, per `project-plan.md` §2.5:

1. **Register → login → refresh → logout** — the full lifecycle of a session
2. **Forgot password → reset** — the recovery flow, kept separate since it issues its own single-use token, distinct from access/refresh tokens

Both render natively on GitHub (Mermaid), no external tool or exported image needed — unlike the ERD, which benefited from dbdiagram.io's schema precision, these are pure control-flow and read fine as text-based diagrams.

`Api` below stands for the whole request pipeline (Controller → Application service), not a single layer — these diagrams are about the auth *flow*, not the Clean Architecture layering already covered in `architecture.md`.

---

## 1. Register → Login → Refresh → Logout

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant Api as DeskLine.Api
    participant Identity as ASP.NET Core Identity (Infra)
    participant DB as Postgres (RefreshToken)

    Note over User,DB: Register
    User->>Browser: Fill registration form
    Browser->>Api: POST /auth/register
    Api->>Identity: Create ApplicationUser (Role=Customer)
    Identity->>Identity: Hash password (NFR-1)
    Identity-->>Api: User created
    Api-->>Browser: 201 Created

    Note over User,DB: Login
    User->>Browser: Enter credentials
    Browser->>Api: POST /auth/login
    Api->>Identity: Verify password + lockout check (D13, NFR-4)
    Identity-->>Api: Valid
    Api->>Api: Issue access token (15 min, JWT)
    Api->>DB: Store new RefreshToken (hash only, D1)
    Api-->>Browser: 200 { accessToken, refreshToken, expiresAt }

    Note over User,DB: Using the session
    Browser->>Api: Authenticated request (Bearer accessToken)
    Api-->>Browser: 200 (normal response)

    Note over User,DB: Access token expires (after 15 min)
    Browser->>Api: Authenticated request (expired accessToken)
    Api-->>Browser: 401 Unauthorized

    Note over User,DB: Refresh
    Browser->>Api: POST /auth/refresh { refreshToken }
    Api->>DB: Look up token by hash
    DB-->>Api: Token valid, not revoked, not expired
    Api->>DB: Revoke old token, insert new one (rotation, NFR-2)
    Api-->>Browser: 200 { new accessToken, new refreshToken, expiresAt }

    Note over User,DB: Reused/rotated token (attack or bug)
    Browser->>Api: POST /auth/refresh { already-rotated token }
    Api->>DB: Look up token by hash
    DB-->>Api: Token marked revoked
    Api-->>Browser: 401 (reuse rejected, NFR-2)

    Note over User,DB: Logout (this session only)
    Browser->>Api: POST /auth/logout { current refreshToken }
    Api->>DB: Set RevokedAt on this token only (FR-1a)
    Api-->>Browser: 204 No Content

    Note over User,DB: Logout-all (every session)
    Browser->>Api: POST /auth/logout-all
    Api->>DB: Set RevokedAt on every token for this UserId (FR-1b)
    Api-->>Browser: 204 No Content
```

**What this makes concrete, beyond the prose in `decisions.md`:**
- The access token is never looked up in the database — it's just verified as a valid signed JWT (this is *why* a deactivated user's access token stays valid for up to 15 minutes, a limitation already named in README/`project-plan.md` §10).
- The refresh token *is* looked up every time, by its hash — that's what makes revocation and reuse-detection possible at all (D1, D7).
- Rotation means every successful refresh invalidates the token that was just used, replacing it with a new one — so a stolen *old* refresh token is useless the moment the legitimate client refreshes once.

---

## 2. Forgot Password → Reset

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant Api as DeskLine.Api
    participant Identity as ASP.NET Core Identity (Infra)
    participant Email as IEmailSender (Infra)
    participant DB as Postgres (RefreshToken)

    User->>Browser: Click "Forgot password", enter email
    Browser->>Api: POST /auth/forgot-password { email }
    Api->>Identity: Look up user by email

    alt Email exists
        Identity-->>Api: User found
        Api->>Identity: Generate single-use, expiring reset token
        Api->>Email: Send reset link (token embedded)
        Email-->>User: Email delivered
    else Email does not exist
        Note over Api: Do nothing — no token generated, no email sent
    end

    Api-->>Browser: 200 OK (identical response either way — FR-1c, no account enumeration)

    User->>Browser: Click link, enter new password
    Browser->>Api: POST /auth/reset-password { resetToken, newPassword }
    Api->>Identity: Validate token (not expired, not already used)

    alt Token valid
        Identity->>Identity: Hash + set new password
        Identity-->>Api: Password updated
        Api->>DB: Revoke ALL refresh tokens for this UserId (FR-1c, same op as FR-1b)
        Api-->>Browser: 200 OK — must log in again on every device
    else Token invalid or expired
        Identity-->>Api: Rejected
        Api-->>Browser: 400 Bad Request
    end
```

**What this makes concrete:**
- The `alt Email exists / else` branch is exactly what "identical response whether or not the email exists" (FR-1c) looks like as control flow — the *branch* happens, but the branch never leaks into what the browser receives.
- Reset success revokes *every* refresh token for the account (reusing FR-1b's exact mechanism), not just issuing a new password — because a password reset is frequently a reaction to a compromised account, so every existing session should die along with the old password (D14).

---

## 3. Traceability note

Both diagrams trace to FR-1/FR-1a/FR-1b/FR-1c and NFR-2/NFR-4, and formalize D1, D7, D13, and D14 as control flow rather than prose. No new decisions are introduced here — this doc is a visualization of choices already logged, not a place new ones get made.

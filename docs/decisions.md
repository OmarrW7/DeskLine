# DeskLine — Decisions Log

**Status:** Living document — updated as major decisions are made, not reconstructed after the fact
**Owner:** Omar
**Format:** Each entry — the decision, the context that forced it, what was considered and rejected, and the reasoning

---
**Methodology note — before any decision is added here:** three questions are 
applied in order: (1) does this map to an existing FR/NFR, or is it a new 
requirement being invented on the fly? (2) is the implementation cost 
proportionate to what it teaches, relative to the project's portfolio purpose? 
(3) can it be tested deterministically, without exotic setup such as concurrency 
simulation or timing dependencies? A decision that fails any of these three is 
reconsidered or scoped out explicitly, rather than added.

## D1: Refresh tokens (short-lived access + long-lived revocable refresh) over a single long-lived JWT

**Decision:** Access tokens expire in 15 minutes; refresh tokens in 7 days and are revocable (NFR-2).

**Context:** A JWT, once issued, can't be invalidated server-side by default — it's valid until it expires, no matter what happens to the account afterward.

**Alternatives considered:**

- Single long-lived JWT (e.g. 7-day access token, no refresh) — simpler to implement, no refresh endpoint needed.
- Session-based auth (server-side session store) — fully revocable at will, but reintroduces server-side session state, which JWT was chosen specifically to avoid.

**Rationale:** A single long-lived token means a stolen token stays valid for its full lifetime with no way to cut it off. Splitting into a short-lived access token plus a separate, revocable refresh token bounds the damage of a leaked access token to 15 minutes, while the refresh token — the one that matters for long-term access — can be revoked immediately (logout, compromise, admin deactivation) without needing server-side session lookups on every request.

**Trade-off accepted:** More moving parts (refresh endpoint, rotation logic, refresh token storage) than a single-token approach.

---

## D2: Clean/Layered Architecture over "fat controllers"

**Decision:** Four layers — Presentation, Application, Domain, Infrastructure — with dependencies pointing inward only (project-plan.md §2.4).

**Context:** Business logic (SLA rules, authorization, audit logging) needs to be testable independent of HTTP.

**Alternatives considered:** Putting business logic directly in controllers (common in tutorials and small CRUD apps) — faster to write initially, no extra layers to navigate.

**Rationale:** Logic embedded in controllers can't be unit tested without spinning up an HTTP pipeline, which is exactly the kind of thing a code reviewer flags. Separating Application-layer services from Presentation-layer controllers means business rules (e.g. "can this agent reassign this ticket") are tested directly, with the controller reduced to HTTP in/out only.

**Trade-off accepted:** More files and more indirection for what could, at this project's scale, be done in fewer layers — justified here specifically because demonstrating the pattern correctly is part of the portfolio goal.

---

## D3: Append-only audit log, no update/delete path exposed

**Decision:** `AuditLog` has no update or delete endpoint, ever (NFR-9). Enforced by construction — no route exists — not by convention or a permissions check.

**Context:** The audit log's entire value is that it's a trustworthy record of what happened. If it can be edited, it can be tampered with by exactly the actors it's meant to hold accountable (agents, admins).

**Alternatives considered:** Exposing an admin-only edit/delete endpoint "for corrections" — more flexible if a bad entry needs fixing.

**Rationale:** An editable audit log defeats its own purpose — "Agent tampers with or deletes log entries to hide an action" is a named threat in the threat model, and the only mitigation that fully closes it is making the capability not exist server-side at all, rather than trusting a permission check to always be enforced correctly.

**Trade-off accepted:** A genuine bad entry (e.g. a bug writing incorrect audit data) can't be corrected in place — it would need a compensating entry, not an edit. Acceptable, since that's also how real append-only audit systems behave.

---

## D4: SLA deadlines run on a 24/7 calendar-hours clock, not business hours

**Decision:** All three SLA metrics (First Response, Next Response, Resolution) count continuously, not restricted to a 9–5 weekday calendar (requirements.md §3).

**Context:** Production ticketing tools (Zendesk, Freshdesk) typically support business-hours SLA calendars so a ticket filed Friday evening doesn't breach a 4-hour SLA overnight.

**Alternatives considered:** Building a configurable business-hours calendar (timezone-aware, holiday support) as part of v1.

**Rationale:** Business-hours SLA support is real complexity, but it's _scheduling_ complexity — timezone handling, holiday calendars — not _ticketing-domain_ complexity. It doesn't teach a new concept relevant to what this project is meant to demonstrate (auth, RBAC, audit logging, SLA state machines). Deferring it to Phase 2 keeps the v1 timeline realistic without cutting anything that's actually load-bearing for the portfolio's purpose.

**Trade-off accepted:** SLA flags in v1 will technically fire outside business hours (e.g. overnight), which wouldn't match real-world helpdesk behavior — named explicitly in Known Limitations rather than hidden.

---

## D5: Round-robin auto-assignment on ticket creation, not agent self-claim

**Decision:** When a ticket is created, the system automatically assigns it to an agent via round-robin (fewest open tickets, or next in rotation) — FR-20a. Admins retain manual reassignment (FR-16). No agent self-claim queue exists.

**Context:** Under heavy ticket traffic, tickets need to reach an agent quickly without every ticket requiring manual admin assignment.

**Alternatives considered:**

- **Agent self-claim** (an unassigned queue agents pick up from) — was the original design (old FR-8), but cut: multiple agents viewing the same unassigned queue can attempt to claim the same ticket simultaneously, a race condition that requires optimistic concurrency control (a `RowVersion`/concurrency token, retry-on-conflict logic) to resolve correctly. That complexity was judged disproportionate to the portfolio value it demonstrates.
- **Pure manual admin assignment** (no automation at all) — simplest, but doesn't scale under traffic and leaves tickets sitting unassigned until an admin acts, which is worse UX than either alternative.

**Rationale:** Auto-assignment at creation avoids the concurrency problem entirely, because it's the server executing one deterministic action inside the same transaction that creates the ticket — there's no second actor racing to claim the same row. This gets the traffic-handling benefit of automation without reintroducing the concurrency complexity that got self-claim cut in the first place.

**Trade-off accepted:** Round-robin doesn't account for agent skill/category specialization — a simple, even distribution, not an intelligent routing engine. That's an intentional scope boundary, not an oversight.

---

## D6: `Role` modeled as an enum on `User`, not a separate table

**Decision:** `User.Role` is an enum (`Customer`, `Agent`, `Admin`), not a foreign key to a `Role` table.

**Context:** DeskLine has exactly three roles, used throughout for authorization checks (RBAC, NFR-3) and API contract role restrictions (project-plan.md §2.3).

**Alternatives considered:** A separate `Role` table with `User.RoleId` as a foreign key — the more "normalized" textbook approach, and the right call if roles were dynamic (e.g. an admin could create custom roles at runtime).

**Rationale:** The three roles are fixed by design and referenced constantly in authorization logic — an enum keeps every permission check a simple comparison (`user.Role == Role.Agent`) instead of requiring a join or a cached lookup just to know what a user is allowed to do. A `Role` table would model flexibility DeskLine doesn't have and doesn't need.

**Trade-off accepted:** If roles ever needed to become dynamic (admin-defined, more than three), this would require a schema migration rather than just inserting a row. Acceptable — RBAC with a small, fixed role set is the industry-standard pattern for a system this size, not a shortcut.

**Contrast with Category (project-plan.md §2.2, FR-17):** Category _is_ a table, not an enum, because Admins can create/edit categories at runtime (FR-17) — the exact condition that didn't apply to Role. Worth keeping both decisions side by side since they look similar on the surface but resolve oppositely for a specific, statable reason.

---

## D7: Multi-session refresh tokens (`RefreshToken` as a separate table) over single-session

**Decision:** `RefreshToken` is a separate table with a foreign key to `User` (one-to-many), not a single token field on `User`. A user can be logged into multiple devices simultaneously, each with its own independently valid, independently revocable refresh token.

**Context:** Auth design (D1) established that refresh tokens must be stored server-side to be revocable. The remaining question was cardinality: one refresh token per user, or several.

**Alternatives considered:**
- **Single-session** (one refresh token field on `User`) — simpler schema, no FK, no per-row queries. Logging in on a new device automatically invalidates any other active session, since there's only one slot to hold a token in.

**Rationale:** Single-session behavior — being silently logged out of your laptop because you logged in on your phone — isn't how production systems behave (Gmail, Slack, GitHub all support concurrent sessions) and isn't a security requirement DeskLine has; it would just be user-hostile with no real benefit here. A separate `RefreshToken` table lets each device hold its own row, revocable independently, which also unlocks a genuine security feature: a user can revoke a single compromised session without logging out everywhere, or revoke all sessions at once if the account is compromised — both real, demonstrable answers to threat-model scenarios.

**Testability:** Every behavior reduces to standard, deterministic assertions filtered by `UserId` — no concurrency simulation or timing dependencies required: a login creates a row scoped to that user; two logins (simulating two devices) create two independent rows; refresh rotates a token and invalidates the one it replaced; reuse of an already-rotated token is rejected; logout revokes only the session tied to the token presented; logout-all revokes every row for that `UserId` while leaving other users' rows untouched.

**Trade-off accepted:** More schema (a full table + FK instead of one column) and more logic (per-row lookups instead of a single field compare) than single-session requires. Also requires logout endpoints (single-session and "log out everywhere") to actually exercise the revocation this design enables — otherwise the extra structure buys nothing. These have been added to `requirements.md` (FR-1a, FR-1b) and the API contract table (`project-plan.md` §2.3) as a direct consequence of this decision.

---

## D9: Human-facing `TicketNumber` added outside the FR list

**Decision:** `Ticket` gets an auto-incrementing `TicketNumber` (`#1`, `#2`, ...) alongside its internal `Guid Id` — permanent once assigned, never reassigned or reused.

**Context:** Not required by any FR. A raw `Guid` works fine as an internal/API identifier but is unusable as something a customer or agent references in conversation.

**Alternatives considered:** No human-facing number at all (simplest, but poor UX — matches no real helpdesk tool).

**Rationale:** Real ticketing tools always expose a short reference number. Cost is a single Postgres `IDENTITY` column — the database handles atomic, correctly-ordered incrementing natively, no custom counter logic needed.

**Trade-off accepted:** Postgres sequences aren't strictly gapless under transaction rollback. Irrelevant here — gapless numbering matters for regulated sequences like invoices, not support ticket references.

---

## D10: `Priority` field added to `Ticket`, Agent/Admin-only, defaults to `Medium`

**Decision:** `Ticket.Priority` (`Low`/`Medium`/`High`/`Urgent`) defaults to `Medium` at creation. The Customer cannot set it; only Agent/Admin can set or change it (FR-13a).

**Context:** Not in the original FR list. Added because priority is foundational to real ticket triage, and (per D11) ends up driving SLA duration.

**Alternatives considered:**
- Customer sets priority at creation — rejected: nearly everyone would pick "Urgent," making the field meaningless as a triage signal.
- An `Unset`/`Untriaged` initial state instead of defaulting to `Medium` — rejected: since SLA deadlines are computed at creation time (FR-21a/c), before any agent has triaged the ticket, an `Unset` value would need an immediate fallback for SLA purposes anyway — reconstructing `Medium` with extra steps rather than avoiding it.

**Rationale:** Defaulting to `Medium` is simpler and avoids a fallback branch in the SLA calculation that an `Unset` state would otherwise require.

**Trade-off accepted:** Loses the ability to distinguish "not yet triaged" from "deliberately set to Medium" — a minor loss, since no FR asks for a "needs triage" view.

---

## D11: SLA durations vary by priority tier, hardcoded via `ISlaPolicyProvider` (not admin-configurable in v1)

**Decision:** All three SLA metrics scale by `Ticket.Priority` instead of one flat duration for every ticket (matrix in `requirements.md` §4). Next Response uses the same duration as First Response per tier. Durations are fixed in application code behind an `ISlaPolicyProvider` interface — not stored in a database table, not Admin-editable in v1.

**Context:** The original flat SLA (4h/4h/48h for every ticket) doesn't reflect real support urgency — an `Urgent` ticket and a `Low` ticket had identical deadlines.

**Alternatives considered:**
- **Admin-configurable `SlaPolicy` table + CRUD endpoints** — rejected for v1. Forces a rethink of the append-only, ticket-scoped `AuditLog` design (D3): a policy change has no natural `TicketId`, meaning either `AuditLog.TicketId` becomes nullable or a second, system-level log table is needed. Also duplicates the "Admin manages config via CRUD" pattern `Category` (FR-17) already demonstrates. Real added cost (new entity, migration, seed data, 2 endpoints, validation, new auth tests) disproportionate to a capability no FR currently requests.
- **Flat Next Response, independent of priority** — rejected: inconsistent with a tiered First Response — an Urgent ticket would get fast initial contact but then slow back down to a generic wait on every follow-up, undermining the point of tiering it at all.

**Rationale:** Hardcoding behind an interface gets real SLA-by-urgency behavior at minimal build cost, while staying swappable — a future Admin-configurable implementation is a new class behind the same interface, not a rewrite of the SLA background job or ticket-creation logic.

**Trade-off accepted:** Changing a duration requires a code change + redeploy in v1. Not a second demonstration of admin-managed CRUD config — `Category` already covers that pattern.

---

## D12: Audit log stays ticket-scoped; admin configuration actions go to structured logs

**Decision:** `AuditLog.TicketId` stays non-nullable. NFR-5 and NFR-10 are narrowed to ticket actions. User management (FR-14) and category management (FR-17) are recorded through structured request logging with the authenticated actor, not the audit table.

**Context:** NFR-5 said all admin/agent actions are traceable via the audit log, but the log has a `TicketId` FK (D3, project-plan §2.2). Deactivating a user or editing a category has no ticket.

**Alternatives considered:**
- Nullable `TicketId` — the table then means two things, weakens the FK guarantee, and every ticket-history query needs an extra filter.
- A second system-level log table — new entity, migration, endpoints, and tests; the same cost objection that ruled out `SlaPolicy` in D11.

**Rationale:** One meaning per table keeps D3's append-only guarantee simple to reason about and test. Admin config actions are lower-stakes and are covered by Serilog request logs (method, path, user id, status) planned for Week 4.

**Trade-off accepted:** Admin config changes aren't in a tamper-resistant table. Named in Known Limitations.

---

## D13: ASP.NET Core Identity in Infrastructure, `Role` stays an enum, lockout enabled

**Decision:** `ApplicationUser : IdentityUser` lives in Infrastructure with a `Role` enum column; Identity's role tables are not used (D6 stands). Identity is accessed through an Application-layer interface. Lockout: 5 failed attempts, 15-minute lock (NFR-4). Domain entities reference users by `Guid` only, with no navigation property to `ApplicationUser`.

**Context:** NFR-1 says "ASP.NET Core Identity defaults" and D6 says `Role` is an enum, and it was undecided how the two fit together. The answer changes the `User` table.

**Alternatives considered:** Own `User` entity plus a standalone `PasswordHasher<User>` — keeps Domain free of Identity types, but lockout, reset tokens, and the normalized-email index would be hand-rolled security code.

**Rationale:** Matches NFR-1 as written and uses battle-tested hashing, lockout, and token generation instead of writing them. The interface keeps Application and Domain testable without Identity (D2).

**Trade-off accepted:** Identity's extra columns (e.g. `SecurityStamp`) appear in the users table, and Infrastructure depends on the Identity package. Lockout lets an attacker lock a victim out for 15 minutes; accepted because the duration is short and rate limiting is the primary control.

---

## D14: Password reset in v1 (FR-1c); email verification deferred to Phase 2

**Decision:** Add FR-1c (forgot/reset password). Remove email verification from the v1 auth flow and list it in Phase 2.

**Context:** Plan §2.5 and NFR-4 referenced both flows, but neither had an FR or API rows.

**Alternatives considered:** Cut both — simplest, but the reset flow is where the security details live (single-use expiring token, no account enumeration, session revocation). Keep both — verification adds a blocked-until-verified state and resend logic for little extra learning.

**Rationale:** Three-question test on FR-1c: maps to plan §2.5/NFR-4; cost is small given D13 (Identity issues and validates tokens, and FR-6 already needs an email sender); testable with a fake `IEmailSender` capturing the link.

**Trade-off accepted:** Self-registration accepts any email address unverified. Named in Known Limitations.

---

## D15: Cross-customer ticket access returns 404, not 403

**Decision:** A Customer requesting a ticket they don't own gets 404, identical to the response for a nonexistent ID. 403 is reserved for role violations.

**Context:** Plan §4 originally expected 403, which confirms to an attacker that the ID exists.

**Alternatives considered:** 403 — clearer for debugging, but confirms existence.

**Rationale:** The ticket query filters by owner, so a ticket outside the caller's scope is simply not found. One filtered query, no separate ownership check after load.

**Trade-off accepted:** Slightly harder to debug; documented in Swagger.

**Testability:** Customer A `GET`s Customer B's ticket → 404, and the response body equals that of a random nonexistent ID. Customer calls a status-change endpoint → 403.

---

## D16: Priority changes recompute SLA deadlines, anchored to `CreatedAt`

**Decision:** See FR-21e. Deadlines are recomputed from `CreatedAt` (or `LastCustomerReplyAt` for an armed Next Response) plus the new tier's duration. Existing overdue flags are never cleared.

**Context:** D10 sets `Medium` at creation and deadlines are computed then; agents triage later. Without recomputation, triage would change nothing about the SLA, contradicting D11.

**Alternatives considered:**
- Keep creation-time deadlines — priority becomes display-only.
- Re-anchor to the time of the change — an agent could reset the clock by toggling priority, hiding real wait time.

**Rationale:** Anchoring to `CreatedAt` makes each deadline a pure function of (`CreatedAt`, `Priority`) — deterministic and not gameable.

**Trade-off accepted:** Raising priority late can make a ticket instantly overdue. That is intended: the ticket really has been waiting that long.

**Testability:** With `FakeTimeProvider`: create a Medium ticket at T0; at T0+3h set Urgent → deadline is T0+1h and the next sweep flags it; downgrade to Low → the flag stays set and the deadline becomes T0+8h.

---

## D17: Ticket lifecycle edge rules

**Decision:**
- Status transitions follow the table in FR-9.
- Comments are allowed only in Open or In Progress (FR-5).
- Ticket creation fails if no active Agent exists (FR-20a).
- An Agent with Open or In Progress tickets can't be deactivated (FR-14).

**Context:** FR-5 said "open ticket" ambiguously, FR-9 showed a linear arrow chain, FR-20a said "never unassigned" without covering zero agents, and FR-14 was silent about a deactivated agent's tickets.

**Alternatives considered:**
- Auto-reopen when a customer replies to a Resolved ticket — common in real tools, but couples reply handling to SLA re-arming; deferred.
- Auto-reassign on deactivation — needs round-robin at deactivation time and more failure paths.
- Fall back to an unassigned queue when no agents exist — contradicts FR-20a and reintroduces the self-claim race D5 removed.

**Rationale:** Each rule is a single deterministic check with an obvious test.

**Trade-off accepted:** A customer whose Resolved ticket isn't actually fixed must open a new one. An admin must reassign manually before deactivating an agent.

**Testability:** One parameterized test per row of the transition table, plus one test per rule above.

---

## D19: Postgres trigger enforces AuditLog append-only, as defense-in-depth

**Decision:** A trigger on `audit_log` raises an exception on any `UPDATE` or `DELETE`, run before the fact:

```sql
CREATE OR REPLACE FUNCTION prevent_audit_log_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_log is append-only';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_log_no_update
BEFORE UPDATE OR DELETE ON audit_log
FOR EACH ROW EXECUTE FUNCTION prevent_audit_log_mutation();
```

**Context:** NFR-9 was already satisfied by "no update/delete endpoint exists" (D3). This adds a second, independent layer that holds even if a future migration, a raw SQL script, or a bug bypasses the application layer entirely.

**Rationale:** Passes the three-question test — maps to NFR-9, costs one small migration, and is deterministically testable (attempt a raw `UPDATE`, assert it throws).

**Trade-off accepted:** One extra migration to maintain; genuine data corrections require a compensating entry, never an edit — same trade-off D3 already accepted.

**Testability:** Integration test opens a raw connection, attempts `UPDATE audit_log SET action = 'x' WHERE id = ...`, and asserts a Postgres exception is thrown.
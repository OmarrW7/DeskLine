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
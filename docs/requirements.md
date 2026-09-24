# DeskLine — Requirements Document

**Status:** Draft v1 — Week 1 design artifact
**Owner:** Omar
**Stack:** ASP.NET Core (C#) · React (TypeScript) · PostgreSQL

---

## 0. How this doc is organized

- **Functional requirements** — grouped by role (Customer, Agent, Admin, System)
- **Non-functional requirements** — grouped by quality attribute (Security, Performance, Reliability, Auditability, Usability, Maintainability, Scalability)
- **Explicitly out of scope for v1** — named on purpose, not forgotten by accident

A rule used throughout: every line item here is testable. If a requirement can't be verified with a test (or at least a manual check) that passes or fails against it, it doesn't belong on this list yet.

---

## 1. Functional Requirements

### 1.1 Customer

- FR-1: Register and log in
- FR-1a: Log out of the current session (revokes only the refresh token used in the request)
- FR-1b: Log out of all sessions (revokes every refresh token associated with the account)
- FR-1c: Reset a forgotten password via an emailed, single-use, expiring reset link. The "forgot password" response is identical whether or not the email exists (no account enumeration). A successful reset revokes all of the account's refresh tokens (same operation as FR-1b)
- FR-2: Submit a new ticket (subject, description, category, optional attachment)
- FR-3: View list of own tickets, filterable by status
- FR-4: View a single ticket's full thread (own messages + agent-visible replies — never internal notes)
- FR-5: Add a comment/reply to a ticket they own while its status is Open or In Progress (not Resolved or Closed)
- FR-6: Receive an email notification on agent reply or status change
- FR-7: Close their own ticket from any status other than Closed (the only status transition a Customer may perform)

### 1.2 Agent

- FR-8: View tickets assigned to them
- FR-9: Change ticket status (Agent/Admin). Allowed transitions: Open → In Progress, In Progress → Resolved, Resolved → Closed, and Resolved → In Progress (reopen). Any other transition is rejected. Closed is terminal: a Closed ticket accepts no comments, status changes, priority changes, or reassignment
- FR-10: Add a customer-visible reply
- FR-11: Add an internal note visible only to agents/admins
- FR-12: Reassign a ticket to another active agent
- FR-13: See SLA status at a glance on their queue — which of the three metrics (if any) are breached per ticket
- FR-13a: Set or change a ticket's priority (Low, Medium, High, Urgent). Defaults to Medium at creation; the customer cannot set or select it — only Agent/Admin can.

### 1.3 Admin

- FR-14: Manage user accounts (create, deactivate, assign roles). An Agent with tickets in Open or In Progress status cannot be deactivated until those tickets are reassigned. Deactivating any account revokes all of its refresh tokens and blocks login
- FR-15: View all tickets across all agents
- FR-16: Manually assign or reassign any ticket (the assignee must be an active Agent)
- FR-17: Manage ticket categories
- FR-18: View basic analytics — ticket volume, average resolution time, count of overdue tickets
- FR-19: View the audit log (read-only)

### 1.4 System-level

- FR-20: Every mutating action on a ticket (create, status change, comment, reassignment, priority change, attachment added) is written to an append-only audit log with actor, action, and timestamp
- FR-20a: On creation, a ticket is automatically assigned via round-robin to the active Agent with the fewest currently open tickets ("open" = status Open or In Progress); ties are broken by lowest user Id so the result is deterministic. No ticket is ever left unassigned: if no active Agent exists, ticket creation fails with a clear error. Admin may manually reassign afterward (FR-16).
- FR-21: SLA tracking — three independent metrics, each flagged by a scheduled background job (not computed on read), with durations that vary by ticket Priority (see §4 for the full matrix):
  - FR-21a: **First Response Time** — flagged "Overdue: First Response" if no agent reply within the priority-tiered duration of ticket creation (Urgent 1h / High 2h / Medium 4h / Low 8h)
  - FR-21b: **Next Response Time** — flagged "Overdue: Next Response" if, after a customer reply, no agent reply follows within the same duration used for First Response at that ticket's priority
  - FR-21c: **Resolution Time** — flagged "Overdue: Resolution" if the ticket isn't marked Resolved within the priority-tiered duration of creation (Urgent 8h / High 24h / Medium 48h / Low 120h)
  - FR-21d: SLA durations are fixed per priority tier, defined in application code (not admin-editable in v1) — see §4
  - FR-21e: When a ticket's priority changes, the First Response and Resolution deadlines are recomputed as `CreatedAt` + the new tier's duration, and an armed Next Response deadline as `LastCustomerReplyAt` + the new tier's duration. An overdue flag that is already set is never cleared by a priority change. A deadline that lands in the past is flagged on the next sweep
  - FR-21f: Only customer-visible agent replies count as a "reply" for FR-21a and FR-21b; internal notes (FR-11) never do. The SLA sweep skips Closed tickets entirely, and Next Response is also not tracked while a ticket is Resolved
- FR-22: File attachments are validated server-side before storage: allowed types are PNG, JPEG, GIF, PDF, and plain text (`.txt`, `.csv`, `.log`); max size 10 MB. A ticket may have at most one attachment, added at creation (FR-2). Any other file type, an oversized file, or a second attachment on the same ticket is rejected with a 400 and is never written to storage
- FR-23: Role-based access control is enforced server-side on every request, not just hidden in the UI

Note: FR-1a and FR-1b apply to all authenticated roles (Customer, Agent, Admin) — 
logout is a session-management action, not role-specific behavior.

---

## 2. Non-Functional Requirements

### 2.1 Security & Compliance

Role-based access control and data protection aren't optional polish on a helpdesk system — industry guidance treats fine-grained RBAC and compliance with data-handling regulations (retention rules, GDPR) as a core evaluation criterion for any helpdesk platform, not an afterthought. For DeskLine:

- NFR-1: Passwords hashed via ASP.NET Core Identity defaults, never logged or stored in plaintext
- NFR-2: JWT access tokens expire in 15 minutes; refresh tokens in 7 days and are revocable. Rotated on every use; reuse of a rotated token is rejected
- NFR-3: Authorization checked server-side on every request — a Customer must never retrieve another Customer's ticket by ID, regardless of UI state. Requesting another Customer's ticket returns 404 (never confirming it exists); calling an endpoint the caller's role doesn't permit returns 403 (see D15)
- NFR-4: Rate limiting on `/login`, `/register`, `/forgot-password`; account lockout for 15 minutes after 5 consecutive failed login attempts (ASP.NET Core Identity lockout — see D13)
- NFR-5: All ticket-related admin/agent actions are traceable to an authenticated user via the audit log. Non-ticket admin actions (user and category management) are recorded via structured request logging, not the audit log (see D12)

### 2.2 Performance

- NFR-6: Standard CRUD endpoints (list tickets, view ticket, add comment) respond in under 300ms for a single request against a seeded dataset of roughly 500 tickets, measured locally — a latency check, not a concurrent-load benchmark (see NFR-15)

### 2.3 Reliability

- NFR-7: `/health` endpoint reports API + database connectivity
- NFR-8: All three SLA checks (first response, next response, resolution) run as scheduled background jobs, not dependent on a user triggering a read

### 2.4 Auditability

- NFR-9: Audit log has no update or delete endpoint exposed, ever — append-only by construction, not convention
- NFR-10: Every state-changing action is attributable to a specific authenticated actor; no anonymous mutations. Ticket actions are attributed through the audit log, admin configuration actions through structured logs (see D12)

### 2.5 Usability

Industry guidance is consistent that a system's value collapses if the people using it daily find it cumbersome — usability for both agents and customers is treated as a top evaluation factor for helpdesk software, not a nice-to-have. For DeskLine, scoped to what's testable:

- NFR-11: Role-based UI — a Customer never sees controls for actions they can't perform (this is a UX layer on top of server-side authorization, never a substitute for it)
- NFR-12: Agent queue surfaces overdue tickets without requiring a separate report

### 2.6 Maintainability

- NFR-13: Clean/Layered Architecture — business logic is unit-testable without spinning up HTTP
- NFR-14: API exposes DTOs, never raw EF Core entities, so the contract doesn't leak persistence details to clients

### 2.7 Scalability (stated honestly)

- NFR-15: v1 runs as a single API instance and a single Postgres instance — no horizontal scaling or load testing in scope. This is named as a known limitation, not hidden.

---

## 3. Explicitly Out of Scope for v1

Helpdesk platforms in the market bundle in a wide surface area — multi-channel intake across phone, chat, and email; knowledge bases and self-service portals; AI-generated ticket summaries; and workflow automation engines with configurable assignment rules and escalation templates. None of that belongs in a one-month portfolio build, and pretending otherwise is exactly the scope creep the project plan warns against. Named explicitly:

- Multi-channel intake (phone, social, chat) — v1 is web + email only
- Knowledge base / self-service article portal
- AI-generated ticket summaries or auto-categorization — deferred to Phase 2 (AI triage), single narrow use case only
- v1 has fixed, code-defined SLA tiers (§4), not a rules builder
- Real-time chat — Phase 2, via SignalR
- Email-to-ticket parsing — Phase 2
- Third-party integrations (CRM, external knowledge bases) via public API — no external API consumers in v1
- Horizontal scaling / load testing — named in NFR-15 above
- **Operational hours (business-hours SLA calendar)** — all three SLA metrics in Section 4 run on a 24/7 calendar-hours clock. A production ticketing system typically excludes nights, weekends, and holidays from SLA deadlines (e.g. a ticket filed at 6pm Friday doesn't breach a 4-hour SLA at 10pm the same night). That requires a business-calendar model with timezone and holiday handling — real complexity, but complexity about scheduling, not about ticketing domain design, so it's cut here deliberately
- **Admin-configurable SLA policy durations** — v1's priority-tiered SLA durations (§4) are hardcoded in the Application layer, not stored in the database or editable by Admin. Deferred to Phase 2, alongside Operational Hours, since both are refinements to the SLA engine rather than new ticketing-domain concepts.
- **Email verification on registration** — v1 accepts unverified email addresses at registration; password reset (FR-1c) is in scope, but confirming email ownership is deferred to Phase 2 (D14)

Each of these is a legitimate feature _category_ in the market — the point isn't that they're bad ideas, it's that a focused, deeply-tested v1 is stronger portfolio material than a shallow attempt at all of them.

---

## 4. SLA Domain Design

Three independent deadline/flag pairs on `Ticket`, each maintained by its own background-job pass, with durations that vary by the ticket's `Priority`:

| Priority | First Response | Next Response | Resolution |
|---|---|---|---|
| Urgent | 1h | 1h | 8h |
| High | 2h | 2h | 24h |
| Medium | 4h | 4h | 48h |
| Low | 8h | 8h | 120h (5 days) |

**Next Response deliberately mirrors First Response per tier** rather than being a separately-justified number — once a conversation is underway, the same urgency expectation should hold for follow-up replies as for the first one. This also means the code only needs to define one independent response-time value per tier, not two.

| Metric | Deadline field | Flag field | Set on | Cleared/resets on |
|---|---|---|---|---|
| First Response | `FirstResponseDeadline` | `IsFirstResponseOverdue` | `CreatedAt + tier duration` | Never resets — permanent breach record |
| Next Response | `NextResponseDeadline` | `IsNextResponseOverdue` | `LastCustomerReplyAt + tier duration`, armed only while awaiting an agent reply | Cleared and re-armed on the next customer reply; not tracked while the ball is in the customer's court |
| Resolution | `ResolutionDeadline` | `IsResolutionOverdue` | `CreatedAt + tier duration` | Never resets — permanent breach record |

Design decisions locked in during Week 1:

- **Once breached, a flag does not always clear itself.** Same reasoning as before — First Response and Resolution are permanent breach records; Next Response is the one metric that legitimately toggles.
- **Next Response only applies while the ticket is waiting on the agent** — unchanged from the original design.
- **Durations are priority-tiered, not flat.** `Priority` defaults to `Medium` at creation and is set only by Agent/Admin (FR-13a) — never the Customer, to avoid every ticket being marked Urgent by whoever files it.
- **Durations are hardcoded, not admin-configurable, in v1.** Implemented behind an `ISlaPolicyProvider` interface in the Application layer so a database-backed, Admin-editable implementation can be swapped in later (Phase 2) without changing the SLA background job or ticket-creation logic. An admin-configurable `SlaPolicy` table was considered and deferred — it would force a rethink of the append-only, ticket-scoped `AuditLog` design (a policy change has no natural `TicketId`), and duplicates a "Admin manages config via CRUD" pattern `Category` already demonstrates.
- **Priority changes recompute deadlines, anchored to `CreatedAt`** — not to the time of the change. This keeps each deadline a pure function of (`CreatedAt`, `Priority`), so an agent can't reset the SLA clock by toggling priority. Raising priority late can make a ticket instantly overdue; that is intended (D16).

---

## 5. Traceability note

Every FR/NFR above should eventually map to: an ERD entity or field (Section 2.2 of the project plan), an API endpoint (Section 2.3), or a threat-model row (Section 2.6) if it has security implications. If something on this list doesn't map to any of those by the end of Week 1, that's a signal it's either underspecified or doesn't actually belong in v1.

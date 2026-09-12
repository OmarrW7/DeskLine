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
- FR-2: Submit a new ticket (subject, description, category, optional attachment)
- FR-3: View list of own tickets, filterable by status
- FR-4: View a single ticket's full thread (own messages + agent-visible replies — never internal notes)
- FR-5: Add a comment/reply to an open ticket they own
- FR-6: Receive an email notification on agent reply or status change
- FR-7: Close their own ticket

### 1.2 Agent
- FR-8: View tickets assigned to them, plus an unassigned queue they can pick up from
- FR-9: Change ticket status (Open → In Progress → Resolved → Closed)
- FR-10: Add a customer-visible reply
- FR-11: Add an internal note visible only to agents/admins
- FR-12: Reassign a ticket to another agent
- FR-13: See SLA status at a glance on their queue — which of the three metrics (if any) are breached per ticket

### 1.3 Admin
- FR-14: Manage user accounts (create, deactivate, assign roles)
- FR-15: View all tickets across all agents
- FR-16: Manually assign or reassign any ticket
- FR-17: Manage ticket categories
- FR-18: View basic analytics — ticket volume, average resolution time, count of overdue tickets
- FR-19: View the audit log (read-only)

### 1.4 System-level
- FR-20: Every mutating action on a ticket (create, status change, comment, reassignment) is written to an append-only audit log with actor, action, and timestamp
- FR-21: SLA tracking — three independent metrics, each flagged by a scheduled background job (not computed on read):
  - FR-21a: **First Response Time** — flagged "Overdue: First Response" if no agent reply within 4 hours of ticket creation
  - FR-21b: **Next Response Time** — flagged "Overdue: Next Response" if, after a customer reply, no agent reply follows within 4 hours
  - FR-21c: **Resolution Time** — flagged "Overdue: Resolution" if the ticket isn't marked Resolved within 48 hours of creation
- FR-22: File attachments are validated server-side (type + size) before storage
- FR-23: Role-based access control is enforced server-side on every request, not just hidden in the UI

---

## 2. Non-Functional Requirements

### 2.1 Security & Compliance
Role-based access control and data protection aren't optional polish on a helpdesk system — industry guidance treats fine-grained RBAC and compliance with data-handling regulations (retention rules, GDPR) as a core evaluation criterion for any helpdesk platform, not an afterthought. For DeskLine:
- NFR-1: Passwords hashed via ASP.NET Core Identity defaults, never logged or stored in plaintext
- NFR-2: JWT access tokens expire in 15 minutes; refresh tokens in 7 days and are revocable
- NFR-3: Authorization checked server-side on every request — a Customer must never retrieve another Customer's ticket by ID, regardless of UI state
- NFR-4: Rate limiting on `/login`, `/register`, `/forgot-password`
- NFR-5: All admin/agent actions traceable to an authenticated user via the audit log

### 2.2 Performance
- NFR-6: Standard CRUD endpoints (list tickets, view ticket, add comment) respond in under 300ms under normal load

### 2.3 Reliability
- NFR-7: `/health` endpoint reports API + database connectivity
- NFR-8: All three SLA checks (first response, next response, resolution) run as scheduled background jobs, not dependent on a user triggering a read

### 2.4 Auditability
- NFR-9: Audit log has no update or delete endpoint exposed, ever — append-only by construction, not convention
- NFR-10: Every state-changing action is attributable to a specific authenticated actor; no anonymous mutations

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
- Workflow automation engine / configurable escalation rules — v1 has one fixed rule (4-hour SLA), not a rules builder
- Real-time chat — Phase 2, via SignalR
- Email-to-ticket parsing — Phase 2
- Third-party integrations (CRM, external knowledge bases) via public API — no external API consumers in v1
- Horizontal scaling / load testing — named in NFR-15 above
- **Operational hours (business-hours SLA calendar)** — all three SLA metrics in Section 4 run on a 24/7 calendar-hours clock. A production ticketing system typically excludes nights, weekends, and holidays from SLA deadlines (e.g. a ticket filed at 6pm Friday doesn't breach a 4-hour SLA at 10pm the same night). That requires a business-calendar model with timezone and holiday handling — real complexity, but complexity about scheduling, not about ticketing domain design, so it's cut here deliberately

Each of these is a legitimate feature *category* in the market — the point isn't that they're bad ideas, it's that a focused, deeply-tested v1 is stronger portfolio material than a shallow attempt at all of them.

---

## 4. SLA Domain Design

Three independent deadline/flag pairs on `Ticket`, each maintained by its own background-job pass (or one job checking all three — an implementation choice for Week 1, not a requirements-level decision):

| Metric | Deadline field | Flag field | Set on | Cleared/resets on |
|---|---|---|---|---|
| First Response | `FirstResponseDeadline` | `IsFirstResponseOverdue` | `CreatedAt + 4h` | Never resets — either the first agent reply lands before the deadline or the flag stays true as a permanent record that this SLA was missed once |
| Next Response | `NextResponseDeadline` | `IsNextResponseOverdue` | `LastCustomerReplyAt + 4h`, set only while awaiting an agent reply | Cleared and re-armed on the *next* customer reply; not tracked while the ball is in the customer's court |
| Resolution | `ResolutionDeadline` | `IsResolutionOverdue` | `CreatedAt + 48h` | Never resets — a ticket resolved late keeps a permanent record of the breach |

Two design decisions locked in during Week 1 rather than left to be discovered mid-build:
- **Once breached, a flag does not always clear itself.** First Response and Resolution are treated as permanent breach records — useful for analytics (e.g. "how often was SLA missed") since that history shouldn't self-heal — while Next Response is the one metric that legitimately toggles on and off as the conversation continues.
- **Next Response only applies while the ticket is waiting on the agent.** If the ticket is Closed, Resolved, or waiting on the *customer* for more information, this deadline does not count down. This is a status-dependent condition encoded explicitly in the background job's query, not something that falls out for free.

---

## 5. Traceability note

Every FR/NFR above should eventually map to: an ERD entity or field (Section 2.2 of the project plan), an API endpoint (Section 2.3), or a threat-model row (Section 2.6) if it has security implications. If something on this list doesn't map to any of those by the end of Week 1, that's a signal it's either underspecified or doesn't actually belong in v1.

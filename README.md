# DeskLine

A multi-role helpdesk ticketing system built to demonstrate production-level practices in authentication, security, and system design — not just CRUD.

**Status:** 🟡 Design phase in progress

---

## Why this project exists

Most portfolio CRUD apps demonstrate that you can move data in and out of a database. DeskLine is built to demonstrate the things that actually separate a junior from a mid-level engineer: a real authentication flow (JWT + refresh token rotation, not "store a token in localStorage forever"), server-enforced authorization across multiple roles, an audit trail that's actually tamper-resistant, and SLA tracking logic that requires thinking about state correctly — not just displaying a countdown.

It's also informed by hands-on helpdesk support experience, which shaped the feature set (internal vs. customer-visible replies, SLA breach visibility, audit logging) around how these systems are actually used day to day, not just how they look in a tutorial.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | ASP.NET Core (C#) |
| Frontend | React (TypeScript) |
| Database | PostgreSQL |
| Auth | JWT (access + refresh token rotation) |
| Architecture | Clean / Layered Architecture |

---

## Core Features (v1)

- **Multi-role access:** Customer, Agent, Admin — enforced server-side, not just hidden in the UI
- **Ticket lifecycle:** creation, status transitions, customer-visible replies vs. agent-only internal notes
- **Append-only audit log:** every mutating action (creation, status change, comment, reassignment, priority change, attachment added) is recorded with actor, action, and timestamp — no update/delete endpoint exposed, ever
- **Priority levels** (Low/Medium/High/Urgent) — set only by Agent/Admin during triage, never the customer, and default to Medium
- **SLA tracking (3 independent metrics), durations tiered by priority,** flagged by a scheduled background job rather than computed on read:
  - First Response Time (1h–8h depending on priority)
  - Next Response Time (mirrors First Response per tier; resets as the conversation continues, pauses while waiting on the customer)
  - Resolution Time (8h–120h depending on priority)
- **File uploads** with server-side type/size validation
- **Email notifications** on reply and status change
- **JWT auth** with short-lived access tokens and revocable, rotating refresh tokens
- **Password reset** via an emailed, single-use link (no account enumeration; revokes all sessions)

## Explicitly Out of Scope for v1 (Phase 2)

Deferred deliberately, not discovered mid-build — see [`docs/requirements.md`](docs/requirements.md) for the full rationale on each:

1. AI-assisted ticket triaging (OpenAI API)
2. Email-to-ticket parsing (inbound webhook)
3. Real-time chat (SignalR)
4. Operational Hours / business-calendar SLA support (v1 SLAs run on a 24/7 clock)
5. Admin-configurable SLA policy durations (v1 SLA tiers are hardcoded)
6. Email verification on registration (v1 accepts unverified emails)

---

## Architecture

```
Presentation   → Controllers (HTTP in/out only, no business logic)
Application    → Services / Use Cases (business rules)
Domain         → Entities, enums, domain logic
Infrastructure → EF Core, DbContext, ASP.NET Core Identity (behind an interface), external services (email, storage)

```

Business logic lives in the Application layer specifically so it's unit-testable without spinning up HTTP — a deliberate rejection of the "fat controller" pattern.

*Architecture diagram, ERD, and permission matrix: see [`docs/architecture.md`](docs/architecture.md) (in progress).*

---

## Documentation

Design artifacts are committed early and intentionally — the git history itself is meant to show a design-first process, not just a finished result.

- [`docs/project-plan.md`](docs/project-plan.md) — full build plan, timeline, and working agreement
- [`docs/requirements.md`](docs/requirements.md) — functional/non-functional requirements, SLA domain design, explicit scope boundaries
- `docs/architecture.md` — system architecture, ERD, permission matrix *(coming next)*
- `docs/decisions.md` — key design decisions and rationale 
- `docs/diagrams/` — auth sequence diagrams, SLA background job flow *(coming next)*

---

## Project Status

| Phase | Status |
|---|---|
| Environment setup | ✅ Done |
| System design (requirements, ERD, architecture, threat model) | 🟡 In progress |
| Backend build (auth, tickets, SLA, audit log) | ⬜ Not started |
| Frontend build (React) | ⬜ Not started |
| Security hardening pass | ⬜ Not started |
| Testing | ⬜ Not started |
| CI/CD + deployment | ⬜ Not started |

This README will be updated as each phase lands — including a "Getting Started" section with local setup instructions once the API is running, and a live demo link once deployed.

---

## Known Limitations (by design)

Named up front rather than left for someone to discover:

- Single API instance, single Postgres instance — no horizontal scaling or load testing in v1
- No caching layer — every read hits the database directly
- SLA deadlines run on a 24/7 clock, not a business-hours calendar (see Phase 2)
- SLA durations are hardcoded per priority tier, not admin-editable (see Phase 2)
- Registration accepts unverified email addresses (see Phase 2)
- Admin configuration actions (user and category management) are logged via structured logs, not the ticket-scoped audit log
- A deactivated user's access token remains valid for up to 15 minutes; refresh tokens are revoked immediately

---

## License

MIT — see [`LICENSE`](LICENSE)
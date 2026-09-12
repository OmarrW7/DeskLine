# DeskLine

A multi-role helpdesk ticketing system built to demonstrate production-level practices in authentication, security, and system design — not just CRUD.

**Status:** 🟡 Design phase complete — implementation starting. See [Project Status](#project-status) below.

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
- **Append-only audit log:** every mutating action (status change, comment, reassignment) is recorded with actor, action, and timestamp — no update/delete endpoint exposed, ever
- **SLA tracking (3 independent metrics),** flagged by a scheduled background job rather than computed on read:
  - First Response Time (4h)
  - Next Response Time (4h, resets as the conversation continues, pauses while waiting on the customer)
  - Resolution Time (48h)
- **File uploads** with server-side type/size validation
- **Email notifications** on reply and status change
- **JWT auth** with short-lived access tokens and revocable, rotating refresh tokens

## Explicitly Out of Scope for v1 (Phase 2)

Deferred deliberately, not discovered mid-build — see [`docs/requirements.md`](docs/requirements.md) for the full rationale on each:

1. AI-assisted ticket triaging (OpenAI API)
2. Email-to-ticket parsing (inbound webhook)
3. Real-time chat (SignalR)
4. Operational Hours / business-calendar SLA support (v1 SLAs run on a 24/7 clock)

---

## Architecture

```
Presentation   → Controllers (HTTP in/out only, no business logic)
Application    → Services / Use Cases (business rules)
Domain         → Entities, enums, domain logic
Infrastructure → EF Core, DbContext, external services (email, storage)
```

Business logic lives in the Application layer specifically so it's unit-testable without spinning up HTTP — a deliberate rejection of the "fat controller" pattern.

*Architecture diagram, ERD, and permission matrix: see [`docs/architecture.md`](docs/architecture.md) (in progress).*

---

## Documentation

Design artifacts are committed early and intentionally — the git history itself is meant to show a design-first process, not just a finished result.

- [`docs/project-plan.md`](docs/project-plan.md) — full build plan, timeline, and working agreement
- [`docs/requirements.md`](docs/requirements.md) — functional/non-functional requirements, SLA domain design, explicit scope boundaries
- `docs/architecture.md` — system architecture, ERD, permission matrix *(coming next)*
- `docs/decisions.md` — key design decisions and rationale *(coming next)*
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

---

## License

MIT — see [`LICENSE`](LICENSE)
# DeskLine — Project Plan
### ASP.NET Core (C#) + React (TypeScript) + PostgreSQL

This document outlines DeskLine's engineering process: design before implementation, security treated as a first-class concern rather than an afterthought, and a deliberate design-then-build workflow that mirrors how a production engineering team operates.

---

## 0. Development Approach

**Design-first discipline:** system design decisions (architecture, data model, API contracts), threat modeling, test strategy, monitoring, and deployment configuration are all worked out and documented *before* the corresponding code is written.

**Rule applied throughout the project:** no code is written for a component until a design artifact exists for it — a diagram, a table, or a short document. This is slower up front and faster overall, and it's also the part of a junior engineer's work that's hardest to fake: the ability to design, not just to type.

---

## 1. Environment Setup

| Tool | Purpose | Install |
|---|---|---|
| .NET 8 SDK | Runs/builds the API | https://dotnet.microsoft.com/download |
| Visual Studio 2022 Community *or* VS Code + C# Dev Kit extension | IDE — VS 2022 offers a friendlier debugger for a first .NET project | Either works; VS 2022 recommended for Week 1 |
| Docker Desktop | Runs PostgreSQL locally in a container, later containerizes the whole app | https://www.docker.com/products/docker-desktop |
| DBeaver *or* pgAdmin | GUI to inspect the PostgreSQL database | DBeaver is lighter, works for any DB |
| Postman *or* Insomnia | Manually test API endpoints, build a request collection | Either |
| Node.js LTS + npm | Runs the React frontend | https://nodejs.org |
| Git + GitHub account | Version control, CI/CD, the public portfolio repo | — |
| EF Core CLI tools | Database migrations | `dotnet tool install --global dotnet-ef` |

**Verification checklist** — run these and confirm output before proceeding:
```
dotnet --version        # should show 8.x
docker --version
docker run hello-world  # confirms Docker actually works, not just installed
node --version           # should show an LTS 20.x or 22.x
git --version
dotnet ef --version      # confirms EF CLI tool installed
```

**PostgreSQL runs in Docker, not installed natively**, from day one — this also mirrors the pattern used for containerized deployment later.
```
docker run --name helpdesk-db -e POSTGRES_PASSWORD=devpassword -p 5432:5432 -d postgres:16
```

Environment setup is verified in full before Week 1 begins. A broken environment discovered mid-build costs a day; discovered at setup, it costs ten minutes.

---

## 2. System Design Phase — Week 1

This phase produces the design artifacts the rest of the project is implemented against — the step that actually demonstrates the ability to design software, not just follow a tutorial.

### 2.1 Requirements

See `requirements.md` for the full functional and non-functional requirements document, covering:
- **Functional requirements**, grouped by role:
  - Customer: submit ticket, view own tickets, comment on own tickets, view status
  - Agent: view assigned tickets, update status, add internal notes vs. customer-visible replies
  - Admin: manage users, assign/reassign tickets, view all tickets, basic analytics
  - Every ticket mutation (created, status changed, comment added, reassigned) is written to an audit log with actor, action, and timestamp
  - SLA tracking across three metrics — First Response, Next Response, and Resolution Time (see 2.2 below)
- **Non-functional requirements**, specific and defensible rather than vague:
  - Auth tokens expire in 15 min (access) / 7 days (refresh)
  - API responses under 300ms for standard CRUD operations
  - Passwords never logged or stored in plaintext, anywhere, including logs
  - All admin actions traceable to a user via the audit log

### 2.2 Domain Modeling

Entities: `User`, `Role`, `Ticket`, `TicketComment`, `Attachment`, `Category`, `AuditLog`.

For each: fields, types, relationships, cardinality — documented as an **ERD**, built with [dbdiagram.io](https://dbdiagram.io) or sketched by hand in Excalidraw.

Two entities carry the most design weight:

- **`AuditLog`** — captures every mutating action on a ticket: who did it (`ActorId`), what happened (`Action` — e.g. `StatusChanged`, `CommentAdded`, `Reassigned`), when (`Timestamp`), and enough detail to reconstruct it (`OldValue`/`NewValue` or a JSON payload column). This is implemented as a dedicated table with a foreign key to `Ticket`, rather than a generic polymorphic log — simpler, and sufficient at this project's scale.
- **`Ticket`** carries three deadline/flag pairs to support SLA tracking — each flipped by a scheduled background job, never computed live on read:
  - `FirstResponseDeadline` (`CreatedAt + 4h`) / `IsFirstResponseOverdue` — never self-clears; a permanent record that this SLA was missed once, useful for breach reporting even after the ticket closes
  - `NextResponseDeadline` (`LastCustomerReplyAt + 4h`, armed only while awaiting an agent reply) / `IsNextResponseOverdue` — the one flag that legitimately resets, re-armed each time the customer replies again; does not count down while the ticket is Closed, Resolved, or waiting on the customer
  - `ResolutionDeadline` (`CreatedAt + 48h`) / `IsResolutionOverdue` — never self-clears, same rationale as First Response

  **Explicitly out of scope for v1: Operational Hours.** All three deadlines run on a 24/7 calendar-hours clock rather than a business-hours calendar (9–5, excluding weekends/holidays). Business-hours SLA calendars are standard in production tools (Zendesk, Freshdesk) but require timezone and holiday-calendar handling — real complexity, but scheduling complexity rather than ticketing-domain complexity, so it's cut deliberately and named here rather than left to be discovered mid-build.

### 2.3 API Design

Before any controller exists, the API contract is documented as a table:

| Method | Route | Auth required | Roles allowed | Purpose |
|---|---|---|---|---|
| POST | /api/v1/auth/register | No | — | Create account |
| POST | /api/v1/auth/login | No | — | Get access + refresh token |
| POST | /api/v1/auth/refresh | No (refresh token) | — | Rotate tokens |
| GET | /api/v1/tickets | Yes | Customer (own), Agent (assigned), Admin (all) | List tickets |
| POST | /api/v1/tickets | Yes | Customer | Create ticket |
| PATCH | /api/v1/tickets/{id}/status | Yes | Agent, Admin | Update status |
| ... | | | | remaining endpoints follow the same pattern |

Request/response DTOs (field lists) for the 3–4 most important endpoints are sketched alongside this table — this is the contract implementation is written against.

### 2.4 Architecture Design

The system uses **Clean/Layered Architecture**:

```
Presentation   → Controllers (HTTP in/out only, no business logic)
Application    → Services / Use Cases (business rules live here)
Domain         → Entities, enums, domain logic
Infrastructure → EF Core, DbContext, external services (email, storage)
```

This avoids the "fat controller" pattern common in tutorials, where business logic lives directly in the controller: that pattern is untestable — business logic can't be unit tested without spinning up HTTP — and is exactly the kind of thing a code reviewer at a real company flags. The folder structure is sketched from this layering before implementation begins.

### 2.5 Auth Flow Design

Two **sequence diagrams** define the auth flow (Excalidraw, or numbered steps in a doc):
1. Register → email verification → login → receive access + refresh token → access token expires → refresh flow → logout (refresh token revoked)
2. Forgot password → reset token emailed → token validated → password updated

The rationale — a short-lived access token paired with a longer-lived, revocable refresh token is safer than a single long-lived token — is documented in the corresponding ADR (see Section 3).

### 2.6 Threat Modeling (STRIDE-lite)

For each major flow — auth, ticket creation, file upload, admin actions — the threat model documents what could go wrong and the corresponding mitigation. Example row:

| Flow | Threat | Mitigation |
|---|---|---|
| Login | Brute-force password guessing | Rate limit login endpoint, account lockout after N failures |
| Ticket view | Customer A reads Customer B's ticket (broken access control) | Authorization check on every query, not just UI hiding |
| File upload | Malicious file uploaded (webshell, oversized file) | Validate file type + size server-side, store outside web root |
| Audit log | Agent or admin tampers with or deletes log entries to hide an action | Audit log is append-only — no update/delete endpoint exposed, ever |

This is the project's security design document, and among the most interview-relevant artifacts in the whole project — "walk me through how you thought about security" is a real interview question, and most junior candidates have nothing comparable to show for it.

**End of Week 1 deliverables:**
- [ ] Requirements doc (functional + non-functional)
- [ ] ERD
- [ ] API contract table
- [ ] Architecture diagram + folder structure sketch
- [ ] 2 auth sequence diagrams
- [ ] Threat model table

Implementation does not begin until all deliverables above are complete and reviewed.

---

## 3. Build Phase — Weeks 2–3

Built in **vertical slices** (one full feature end-to-end: DB → API → frontend), not horizontal layers (not "all DB tables first, then all controllers"). This mirrors how production teams ship and keeps something demoable at every stage.

**Week 2:**
- Day 1–2: Auth (register, login, JWT issuance, refresh token flow) — implemented against the Week 1 design
- Day 3: User/role management, EF Core migrations, seed data (including demo accounts — see below)
- Day 4–5: Ticket CRUD (Customer-facing side), authorization checks, audit log writes wired into every mutation from day one

**Week 3:**
- Day 1–2: Agent workflows (assignment, status transitions, internal vs. customer-visible notes)
- Day 3: Admin endpoints, basic analytics query, SLA background service (checks all three metrics — First Response, Next Response, Resolution — against their respective deadlines)
- Day 4: File upload with validation, email notification on status change
- Day 5: React frontend wiring — auth flow, role-based routing, ticket dashboard, Swagger/OpenAPI docs enabled on the API

Each vertical slice follows a test-alongside-implementation approach: a test is written for each slice before moving to the next, rather than deferring testing to the end — so gaps in understanding of a feature surface immediately, not in Week 4.

**Portfolio-readiness items folded in as work progresses, not bolted on at the end:**
- **Demo accounts + seed data** (Week 2, Day 3): one seeded Customer, Agent, and Admin login, documented in the README, so a reviewer can explore the live demo in under a minute without registering
- **Feature branches + PRs into `main`**, even solo, with a short PR template — costs nothing and produces a genuine git workflow history
- **One ADR (Architecture Decision Record) per major design choice**, written the week the choice is made — e.g. "why refresh tokens over long-lived JWTs," "why Clean Architecture," "why an append-only audit log." A few paragraphs each, kept in `/docs/adr/`, written as decisions happen across Weeks 1–3 rather than reconstructed retroactively in Week 4.

---

## 4. Security Hardening Pass — dedicated days, end of Week 3 / start of Week 4

A checklist walked explicitly, in order, each item treated as pass/fail:

- [ ] Password hashing verified (ASP.NET Core Identity defaults — work factor confirmed, not just assumed)
- [ ] JWT signing key strength + expiry correctly configured; refresh tokens rotate and can be revoked
- [ ] Rate limiting on `/login`, `/register`, `/forgot-password`
- [ ] Input validation (FluentValidation) on every incoming DTO
- [ ] Authorization tested with a *negative* test: can Customer A fetch Customer B's ticket by ID? (test expects a 403 before it's trusted to work)
- [ ] CORS restricted to known origins only, not `AllowAnyOrigin`
- [ ] HTTPS enforced, HSTS enabled
- [ ] Secrets never in source control — checked in git history, not just current files
- [ ] No raw SQL string concatenation anywhere (EF Core parameterizes by default — confirmed it hasn't been bypassed)
- [ ] File upload: type + size validated server-side, stored outside the web root
- [ ] Dependency vulnerability scan: `dotnet list package --vulnerable` and `npm audit`
- [ ] Basic security headers: `X-Content-Type-Options`, `X-Frame-Options`, a minimal CSP
- [ ] Audit log table has no exposed update/delete endpoint — genuinely append-only, not just append-only "by convention"
- [ ] Public demo hardened against abuse: since this is a live, writable link anyone can hit, a reset strategy (scheduled job to wipe/reseed) or a read-only demo mode prevents a stranger from filling it with junk or leaving it broken for the next visitor

This checklist itself is portfolio material — a cleaned-up version appears in the README as "Security Considerations."

---

## 5. Testing Strategy

- **Unit tests** (xUnit): business logic in the Application layer, with repositories mocked
- **Integration tests**: `WebApplicationFactory` hitting real API endpoints against a test database (Testcontainers for Postgres — the production-realistic approach)
- **Manual pass**: a Postman collection specifically covering role-based access *negative* cases (wrong role, wrong owner, expired token, missing token)
- **Frontend**: basic component tests, stretch goal only if time allows

Coverage is not chased for its own sake. What's covered is authorization logic and the business rules that would actually break something if wrong — that's what a reviewer cares about, and it's worth being explicit about that priority.

---

## 6. Monitoring & Observability — Week 4

- Structured logging with **Serilog**, sinks to console + file (Seq for a nicer local viewer)
- `/health` endpoint using ASP.NET Core's built-in health checks
- Request logging middleware: method, path, status code, duration
- Optional: exceptions hooked to **Sentry** free tier on the live deployment
- Once deployed, a free **UptimeRobot** check provides genuine evidence of monitoring, not just a claim

---

## 7. CI/CD & Deployment — Week 4

- **GitHub Actions**: run build + tests on every PR; deploy on merge to `main`
- **Docker Compose** for local dev: API + PostgreSQL + React, one command to spin up
- **Deploy targets**: Render or Railway for API + DB (both have workable free/cheap tiers for .NET + Postgres), Vercel or Netlify for the React frontend. Azure App Service is also worth considering, since Azure shows up frequently in Gulf-region .NET job postings — relevant extra setup effort for that market.
- Secrets managed via **GitHub Secrets** in CI, never hardcoded
- Final README: architecture diagram, setup instructions, and a short "Security Considerations" section pulled from the Section 4 checklist

---

## 8. Timeline

**A one-month timeline is realistic if scope stays disciplined.** Assuming roughly 3–4 hours/day, 6 days/week (a part-time pace, run alongside an active job search):

| Week | Focus |
|---|---|
| 0 (few days) | Environment setup, tooling familiarization |
| 1 | System design — all diagrams and docs from Section 2 |
| 2–3 | Build, vertical slice by vertical slice |
| 4 | Security hardening, testing pass, monitoring, CI/CD, deployment, docs |

**The risk to the timeline isn't the code — it's scope creep.** Real-time notifications, payment integration, a polished UI design system, multi-language support — none of that is in Section 2's core feature list, and none of it is added until the core system is deployed and solid. A "Stretch Goals" list is kept separately, addressed only after Week 4 is complete.

If available time is closer to 1–2 hours/day, the plan extends to 5–6 weeks rather than cutting the security or testing sections short — those are the parts doing the most work for the underlying job search.

---

## 9. Phase 2 — Deferred Features (after v1 is deployed and solid)

These were considered and deliberately scoped out of the one-month core build — not because they're bad ideas, but because each adds enough complexity or external dependency to put the deadline at risk. Attempted in this order, only once Sections 1–8 are done and deployed:

1. **AI Ticket Triaging** — OpenAI API call on ticket creation to auto-categorize and set priority. Lowest cost/complexity of the four; API key never client-side, and the triggering endpoint rate-limited since it carries a real (small) per-call cost.
2. **Email-to-Ticket Parsing** — inbound email webhook (SendGrid Inbound Parse or Mailgun Routes, both with workable free tiers) that creates a ticket from an incoming email. Requires webhook signature verification and safe parsing of untrusted content — a good security exercise, just not a fast one.
3. **Real-Time Live Chat** — in .NET this is **SignalR**, not Socket.io (Socket.io is Node-only). Free to use, but introduces persistent connection state, hub testing, and scaling considerations meaningfully different from the rest of this stack. Highest complexity of the four — attempted last, if at all.
4. **Operational Hours (Business-Calendar SLAs)** — extends all three SLA deadlines to respect a configurable business-hours calendar (e.g. 9am–5pm, weekdays only) instead of running 24/7. Requires timezone-aware scheduling and holiday-calendar support. Lowest priority of the four Phase 2 items — pure scheduling complexity, with no new ticketing-domain concepts to demonstrate.

---

## 10. Known Limitations

Stated plainly here and in the README, rather than left for a reviewer to discover — naming these boundaries deliberately reads as engineering judgment, not an oversight:

- **No horizontal scaling or load testing** — this is a single API instance and a single database with no redundancy. If either goes down, the app goes down. Reasonable for a portfolio deploy; if raised, the answer is "here's what production would add" (managed DB with replicas/backups, multiple app instances behind a load balancer), not a claim that it's already handled.
- **Feature set is intentionally narrow** — no real-time chat, email ingestion, or AI triage in v1, by design (see Phase 2 above). A deliberate trade-off to go deep on auth, security, and testing rather than wide on features, given the available time.
- **No caching layer** — every read hits the database directly. Reasonable at this scale; worth naming as a next step if performance becomes a factor.

---

## 11. Competencies Demonstrated by Phase

By the end of each phase, the following should be explainable out loud, interview-style — not just recognizable on sight:

- **After Week 1:** the layers of Clean Architecture and why controllers shouldn't hold business logic; JWT vs. session-based auth and why refresh tokens exist; what STRIDE is and how it was applied here
- **After Week 2–3:** how EF Core migrations work; the difference between authentication and authorization, and where each is enforced in the codebase
- **After Week 4:** what the CI/CD pipeline actually does on every push; what each item in the security checklist protects against, specifically — not just "it's more secure"

Anything that can't be explained clearly at this level is a signal to revisit it before moving on, not to skip past it.

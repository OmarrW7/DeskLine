# DeskLine — API Contract

**Status:** API Design
**Owner:** Omar
**Depends on:** `requirements.md`, `database-design.md`, `decisions.md` (D1–D19)

---

## 0. What this doc is, and isn't

This is a **design-time draft**, written before any controller exists, per the project's own design-first discipline (`project-plan.md` §0). It is not — and isn't meant to become — the permanent source of truth.

Once controllers and DTOs exist in code, ASP.NET Core's built-in OpenAPI support (`AddOpenApi()` / `MapOpenApi()`, already wired in `Program.cs`) generates a live, always-accurate OpenAPI (Swagger) document directly from the code. That generated document supersedes this one — it can't drift from the implementation the way a hand-maintained Markdown file can. This doc's job is narrower: get the request/response shapes and authorization rules right on paper first, so the controllers are fast and correct to write.

GraphQL and gRPC were considered and don't fit: GraphQL solves a client-driven, flexible-query problem this project doesn't have (a small, fixed set of role-based views), and gRPC is for service-to-service communication, not a single API talking to a browser frontend (NFR-15).

---

## 1. Endpoint Table

### 1.1 Auth

| Method | Route | Auth | Roles | FR |
|---|---|---|---|---|
| POST | `/api/v1/auth/register` | No | — | FR-1 |
| POST | `/api/v1/auth/login` | No | — | FR-1 |
| POST | `/api/v1/auth/refresh` | No (refresh token) | — | NFR-2 |
| POST | `/api/v1/auth/logout` | Yes | All | FR-1a |
| POST | `/api/v1/auth/logout-all` | Yes | All | FR-1b |
| POST | `/api/v1/auth/forgot-password` | No | — | FR-1c |
| POST | `/api/v1/auth/reset-password` | No (reset token) | — | FR-1c |

### 1.2 Tickets

| Method | Route | Auth | Roles | FR |
|---|---|---|---|---|
| GET | `/api/v1/tickets` | Yes | Customer (own), Agent (assigned), Admin (all) | FR-3/8/15 |
| POST | `/api/v1/tickets` | Yes | Customer | FR-2 (attachment as multipart, not a separate call — FR-22 allows only one, at creation) |
| GET | `/api/v1/tickets/{id}` | Yes | Customer (own), Agent, Admin | FR-4 |
| POST | `/api/v1/tickets/{id}/comments` | Yes | Customer, Agent, Admin | FR-5/10/11 |
| PATCH | `/api/v1/tickets/{id}/status` | Yes | Agent, Admin | FR-9 |
| POST | `/api/v1/tickets/{id}/close` | Yes | Customer | FR-7 — the one transition a Customer performs, kept separate from general status change |
| PATCH | `/api/v1/tickets/{id}/priority` | Yes | Agent, Admin | FR-13a |
| PATCH | `/api/v1/tickets/{id}/assignment` | Yes | Agent, Admin | FR-12/16 |
| GET | `/api/v1/tickets/{id}/attachments/{attachmentId}` | Yes | Customer (own), Agent, Admin | Download — same ownership check as NFR-3 |

### 1.3 Categories

| Method | Route | Auth | Roles | FR |
|---|---|---|---|---|
| GET | `/api/v1/categories` | Yes | All (Customer needs it for the create-ticket dropdown) | FR-17 |
| POST | `/api/v1/categories` | Yes | Admin | FR-17 |
| PATCH | `/api/v1/categories/{id}` | Yes | Admin | FR-17 (rename, toggle `IsActive`) |

### 1.4 Users & Agents

| Method | Route | Auth | Roles | FR |
|---|---|---|---|---|
| GET | `/api/v1/users` | Yes | Admin | FR-14 — full detail, all roles, for account management |
| POST | `/api/v1/users` | Yes | Admin | FR-14 |
| PATCH | `/api/v1/users/{id}` | Yes | Admin | FR-14 (deactivate, change role) |
| GET | `/api/v1/agents` | Yes | Agent, Admin | FR-12/16 — narrow lookup, active Agents only, minimal fields, for the reassignment dropdown |

### 1.5 Audit Log, Analytics, Health

| Method | Route | Auth | Roles | FR |
|---|---|---|---|---|
| GET | `/api/v1/audit-log` | Yes | Admin | FR-19 |
| GET | `/api/v1/analytics` | Yes | Admin | FR-18 |
| GET | `/health` | No | — | NFR-7 |

**Total: 26 endpoints.**

---

## 2. DTOs

### 2.1 Auth

**`RegisterRequest`**
| Field | Type | Notes |
|---|---|---|
| `email` | string | |
| `password` | string | Identity validates strength |
| `fullName` | string | |

**`LoginRequest`** — `email`, `password`

**`AuthResponse`** (login, register, refresh)
| Field | Type | Notes |
|---|---|---|
| `accessToken` | string | 15-min JWT |
| `refreshToken` | string | Raw token — the only point the raw value is ever sent (D1) |
| `expiresAt` | timestamptz | For the frontend to schedule a refresh |

**`RefreshRequest`** — `refreshToken`

**`ForgotPasswordRequest`** — `email`

**`ResetPasswordRequest`** — `resetToken`, `newPassword`

`logout` / `logout-all` take no body — the refresh token comes from the auth context.

### 2.2 Tickets

**`TicketSummaryDto`** — `GET /tickets`, and reused anywhere a ticket appears in a list

| Field | Type | Notes |
|---|---|---|
| `id` | Guid | |
| `ticketNumber` | int | Human-facing reference (D9) |
| `subject` | string | |
| `status` | string (enum) | |
| `priority` | string (enum) | |
| `categoryName` | string | Not `categoryId` — avoids forcing a second lookup client-side |
| `assignedAgentName` | string | |
| `isFirstResponseOverdue` | bool | FR-13, SLA status at a glance |
| `isNextResponseOverdue` | bool | |
| `isResolutionOverdue` | bool | |
| `createdAt` | timestamptz | |

**`TicketDetailDto`** — `GET /tickets/{id}`. Everything in `TicketSummaryDto`, plus:

| Field | Type | Notes |
|---|---|---|
| `description` | string | |
| `customerName` | string | |
| `comments` | `TicketCommentDto[]` | Server-filtered by role (5c) — a Customer's response never contains internal notes |
| `attachments` | `AttachmentDto[]` | |

**`TicketCommentDto`**
| Field | Type | Notes |
|---|---|---|
| `id` | Guid | |
| `authorName` | string | |
| `body` | string | |
| `isInternal` | bool | Only ever `true` in an Agent/Admin response |
| `createdAt` | timestamptz | |

**`AttachmentDto`**
| Field | Type | Notes |
|---|---|---|
| `id` | Guid | |
| `fileName` | string | Never `storagePath` (NFR-14) |
| `contentType` | string | |
| `sizeBytes` | long | |

**`CreateTicketRequest`** — `POST /tickets` (multipart)

| Field | Type | Notes |
|---|---|---|
| `subject` | string | |
| `description` | string | |
| `categoryId` | Guid | |
| `attachment` | file | Optional, ≤10 MB, whitelisted types (FR-22) |

No `priority` field — a Customer can't set it (FR-13a), so the DTO has no place to put one.

**`AddCommentRequest`** — `body`, `isInternal` (rejected server-side if caller is a Customer, per 5c)

**`ChangeStatusRequest`** — `newStatus` (enum, validated against the FR-9 transition table)

**`ChangePriorityRequest`** — `newPriority` (enum)

**`ReassignRequest`** — `newAgentId` (Guid, must be an active Agent)

`POST /tickets/{id}/close` takes no body — a fixed transition, not a general one.

### 2.3 Categories

**`CategoryDto`** — `id`, `name`, `isActive`

**`CreateCategoryRequest`** — `name`

**`UpdateCategoryRequest`** — `name` (optional), `isActive` (optional)

### 2.4 Users & Agents

**`UserDto`** — `GET /users` (Admin only)

| Field | Type | Notes |
|---|---|---|
| `id` | Guid | |
| `email` | string | |
| `fullName` | string | |
| `role` | string (enum) | |
| `isActive` | bool | |
| `createdAt` | timestamptz | |

Never `passwordHash` — stated explicitly here as a deliberate omission, not an oversight.

**`CreateUserRequest`** — `email`, `password`, `fullName`, `role`

**`UpdateUserRequest`** — `role` (optional), `isActive` (optional)

**`AgentOptionDto`** — `GET /agents`. Deliberately the narrowest DTO in the doc.

| Field | Type |
|---|---|
| `id` | Guid |
| `fullName` | string |

### 2.5 Audit Log, Analytics, Health

**`AuditLogEntryDto`**
| Field | Type | Notes |
|---|---|---|
| `id` | Guid | |
| `ticketNumber` | int | Not the raw `ticketId` Guid |
| `actorName` | string | |
| `action` | string (enum) | |
| `oldValue` | string? | |
| `newValue` | string? | |
| `timestamp` | timestamptz | |

**`AnalyticsDto`** (FR-18)
| Field | Type | Notes |
|---|---|---|
| `totalTickets` | int | |
| `ticketsByStatus` | Dictionary<string, int> | |
| `averageResolutionTimeHours` | double | From `ResolvedAt - CreatedAt` on resolved tickets |
| `overdueTicketCount` | int | Any of the three `Is...Overdue` flags true |

**`HealthDto`** — `status` ("Healthy"/"Unhealthy"), `database` (bool)

**~19 DTOs cover all 26 endpoints** — several are reused across multiple routes rather than one DTO per endpoint.

---

## 3. Example Payloads

### 3.1 `POST /api/v1/tickets`

Request (multipart — JSON fields shown; `attachment` is a file part):
```json
{
  "subject": "Cannot access shared drive",
  "description": "Getting a 403 error when opening the Finance shared drive since this morning.",
  "categoryId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

Response (`201 Created`) — `TicketDetailDto`:
```json
{
  "id": "9c858901-8a57-4791-81fe-4c455b099bc9",
  "ticketNumber": 142,
  "subject": "Cannot access shared drive",
  "description": "Getting a 403 error when opening the Finance shared drive since this morning.",
  "status": "Open",
  "priority": "Medium",
  "categoryName": "Access & Permissions",
  "customerName": "Sara Al-Mutairi",
  "assignedAgentName": "Youssef Nabil",
  "isFirstResponseOverdue": false,
  "isNextResponseOverdue": false,
  "isResolutionOverdue": false,
  "createdAt": "2026-09-29T08:14:00Z",
  "comments": [],
  "attachments": []
}
```
Note `priority` returns `"Medium"` — the server-assigned default — even though the request never included it.

### 3.2 `GET /api/v1/tickets`

Response — `TicketSummaryDto[]`:
```json
[
  {
    "id": "9c858901-8a57-4791-81fe-4c455b099bc9",
    "ticketNumber": 142,
    "subject": "Cannot access shared drive",
    "status": "Open",
    "priority": "Medium",
    "categoryName": "Access & Permissions",
    "assignedAgentName": "Youssef Nabil",
    "isFirstResponseOverdue": true,
    "isNextResponseOverdue": false,
    "isResolutionOverdue": false,
    "createdAt": "2026-09-29T08:14:00Z"
  }
]
```
This is FR-13's "SLA status at a glance" on the wire — the three overdue booleans are in the list response itself, no per-ticket detail call needed to render the queue.

---

## 4. Traceability note

Every endpoint maps to an FR/NFR. Every DTO field maps to a `database-design.md` entity field, except display-only additions (`categoryName`, `assignedAgentName`, `authorName`, `customerName`, `actorName`) — these are Application-layer projections over the underlying `Guid` foreign keys (D18), not new persisted fields.

# DeskLine — Threat Model

**Status:**design artifact (Threat Model)
**Owner:** Omar
**Depends on:** `requirements.md`, `database-design.md`, `api-contract.md`, `decisions.md` (D1, D3, D7, D10, D13, D14, D15, D18, D19, D20)

---

## 0. How this doc is organized

This is a **STRIDE-lite** threat model, not a full STRIDE matrix (Spoofing / Tampering / Repudiation / Information Disclosure / Denial of Service / Elevation of Privilege) walked category by category. For a project this size, a flat table — one row per realistic attack against one real flow in the system — surfaces the same risks without the overhead of filling in six columns per entity just to demonstrate the format exists (the project's own cost-vs-teaching test, applied here too).

Each row below is: the **flow** being attacked, the concrete **threat** against it, and the **mitigation** actually built (or explicitly accepted as a bounded trade-off) — with pointers back to the FR/NFR/decision that put it in place. Nothing here invents a new control; every mitigation either already exists elsewhere in the design or is logged as a new decision (D20) the same way every other decision in this project is.

---

## 1. Threats and Mitigations

| # | Flow | Threat | Mitigation |
|---|---|---|---|
| 1 | Login | Brute-force password guessing | Rate limiting on the login endpoint; account lockout after 5 failed attempts for 15 minutes (NFR-4, D13). |
| 2 | Ticket view | Customer A reads Customer B's ticket | Authorization is enforced as a query filter on every read, not hidden by the UI alone — an out-of-scope ticket simply isn't returned (NFR-3, D15). |
| 3 | File upload | Malicious file uploaded as a ticket attachment | File type and size validated server-side against a fixed whitelist (not just the client), stored outside the web root so it's never directly executable or served as a path (FR-22). |
| 4 | Audit log | Agent or Admin tampers with or deletes a log entry to hide an action | Closed two ways independently: no update/delete endpoint exists at all (D3), and a Postgres trigger rejects any raw `UPDATE`/`DELETE` on the table even if the application layer is bypassed entirely (D19). |
| 5 | Auth / Refresh | Refresh token theft and reuse | The raw token is never persisted, only its hash (D1). Reuse of an already-rotated token is rejected on that call (NFR-2, D7) **and** now also revokes every other active session for that user (D20) — closing the gap where a stolen token's first, successful use would otherwise stay valid independently of the rejected reuse attempt. |
| 6 | Ticket / Priority | Mass assignment — a client sets `priority`, `role`, or `isInternal` fields it isn't allowed to control | Prevented by construction, not by a role check alone: `CreateTicketRequest` has no `priority` field (D10); `ChangePriorityRequest` is a separate endpoint restricted to Agent/Admin (FR-13a); `AddCommentRequest.isInternal` is rejected server-side whenever the caller is a Customer, regardless of what the request body contains (5c, NFR-14). |
| 7 | Attachments | IDOR on attachment download (guessing or reusing an attachment ID) | The same ownership check used for ticket access (NFR-3) gates attachment downloads too: the attachment's parent ticket must be visible to the caller before the file is served, independent of whether the attachment ID itself is guessable. An out-of-scope request returns 404, identical to a nonexistent one (D15). |
| 8 | Login | User enumeration via differing error messages | Login returns one generic "invalid email or password" response regardless of which was wrong — the same no-enumeration principle FR-1c already applies to forgot-password, extended here to login. |
| 9 | Ticket / Email | Stored XSS in ticket or comment text, surfacing in the UI or in outbound emails | React escapes rendered text by default — no `dangerouslySetInnerHTML` is used on user-supplied content. Ticket/comment text is HTML-encoded when interpolated into outbound email templates (FR-6). A minimal CSP header is added in the Week 4 hardening pass (`project-plan.md` §4 checklist). |
| 10 | Admin / Deactivation | A deactivated user's still-valid access token keeps working for up to 15 minutes | Accepted as a bounded trade-off, not fully closed: access tokens are stateless JWTs, verified without a database lookup by design (D1). Deactivation revokes all refresh tokens immediately, so the window closes at the next refresh attempt, 15 minutes after deactivation at the latest. Named explicitly in `README.md` / `project-plan.md` §10 Known Limitations rather than left undocumented. |

---

## 2. Why rows 4, 5, and 10 look different from the rest

Most rows above are a single control that closes the threat outright. Three are worth calling out because they don't:

- **Row 4** uses two independent mitigations rather than one, specifically because the attacker being defended against (an Agent or Admin) is the same actor who would normally be trusted to operate the system correctly — a single application-layer check has a weaker guarantee against that actor than a database-enforced one.
- **Row 5** was a genuinely open question during design (see `decisions.md` D20) — whether rejecting the one reused call was sufficient, or whether the blast radius needed to be the whole account. It's resolved now, but the table reflects a real decision point rather than an obvious default.
- **Row 10** is not closed at all — it's a named, bounded, and accepted trade-off, the direct consequence of choosing stateless JWTs (D1) over a database-checked session on every request. Pretending otherwise here would contradict what D1 already states plainly.

---

## 3. Traceability note

Every mitigation in §1 points to an existing FR, NFR, or decision in `decisions.md` — this document introduces exactly one new decision of its own (D20), logged there in full rather than only here, so the reasoning behind it isn't duplicated or allowed to drift between the two files.

# Build Log

A running, chronological record of the concrete technical work done on this system — exact configuration values, expressions, and identifiers, as they were built. Where a piece of work has its own detailed write-up (like a specific flow), this log gives the short version and links out to it. This file grows as each new piece gets built — check back here for the latest state of any flow, list, or identifier.

## Environment

| Item | Value |
|---|---|
| SharePoint site | `IT Help Desk` — `https://nabatisnackcoid.sharepoint.com/sites/ITHelpDesk` |
| Microsoft 365 tenant | `nabatifood.com.ph` (Power Automate environment shows as `nabatisnack.co.id`) |
| `IT Tickets` list internal GUID | `82145de6-54d2-43ba-bd62-3a59249fcf41` |

## Phase 1 — Data

Built directly in the SharePoint UI (no scripts/commands — list and column creation was done by hand through **Site contents → New → List**, and **+ Add column** per field).

- Site `IT Help Desk` created.
- List `IT Tickets` created with all columns, choice values, and colors. Full schema: [`sharepoint-schema.md`](./sharepoint-schema.md).
- List `Ticket Settings` created with one row (`AssignmentMode: Manual`). Same schema doc.

**Status:** Complete.

## Phase 2 — Automation

### Flow 1 — `ITHD-01 - New Ticketing Routing` (renamed from `IT Ticket - New Request Routing` once it grew past ID minting)

Built in Power Automate, launched via the `IT Tickets` list's **Workflows** button → **Build from scratch** (the simplified in-SharePoint flow builder doesn't support expressions/conditions, so this hands off to the full designer at make.powerautomate.com).

**Trigger** — When an item is created (SharePoint), Site Address `IT Help Desk`, List Name `IT Tickets`.

Mints a Ticket ID, then branches on `Ticket Settings.AssignmentMode`: Manual mode posts a Teams adaptive card to `ManualAssignmentApprover` and waits for a pick (Nikko/Tristan/JP); Auto mode round-robins between the two Assistants (workload-aware, out-of-office-aware, falls back to a Manager FYI card if both are out). Both paths write `Assigned To` / `Status: Assigned` back to the ticket; Auto also updates `Ticket Settings.LastAssignedTo`.

**Tested:** `TCK-2026-0000001`/`0000002` (ID minting), `TCK-2026-0000013` (Manual mode, full card + button click), Test Tickets 15/16/18 (Auto mode: no-tie, one-out, both-out). All paths of both branches confirmed working end-to-end.

Full write-up including every issue hit while building this (Entra ID device-auth error, stale connection references, the Id/Ticket ID field mix-up, "New designer" hiding fields, the flat-vs-nested-bracket dynamic-content bug hit three times, the `select()` failure workaround, Teams action naming, Initialize-variable top-level requirement, OutOfOffice multi-select column fix): [`flow-1-ticket-routing.md`](./flow-1-ticket-routing.md).

**Status:** Complete and tested (both Manual and Auto branches, all sub-paths).

### Flow(s) still to build

- `ActingApprover` field on `Ticket Settings` (delegates the Manual-mode card away from `ManualAssignmentApprover` without changing that setting) — designed, not yet built.
- Escalation handling (reassignment + `Escalated To` + notification).
- Resolution notice to the requester.

*(Each of these will get logged here with its exact configuration once built, same as Flow 1 above.)*

## Phase 3 — App

Not yet started.

## Phase 4 — Teams

Not yet started.

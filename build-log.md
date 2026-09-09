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

### Flow 1 — `IT Ticket - New Request Routing` (Ticket ID minting)

Built in Power Automate, launched via the `IT Tickets` list's **Workflows** button → **Build from scratch** (the simplified in-SharePoint flow builder doesn't support expressions/conditions, so this hands off to the full designer at make.powerautomate.com).

**Trigger** — When an item is created (SharePoint), Site Address `IT Help Desk`, List Name `IT Tickets`.

**Action 1 — Compose - Ticket ID**, final expression:

```
concat('TCK-', formatDateTime(utcNow(), 'yyyy'), '-', if(greaterOrEquals(length(string(triggerOutputs()?['body/ID'])), 7), string(triggerOutputs()?['body/ID']), substring(concat('000000', string(triggerOutputs()?['body/ID'])), sub(length(concat('000000', string(triggerOutputs()?['body/ID']))), 7), 7)))
```

**Action 2 — Update item** (SharePoint): `Id` = `triggerOutputs()?['body/ID']` (expression), `Ticket ID` = Outputs of Compose - Ticket ID, every other field left blank.

**Tested:** `TCK-2026-0000001` (first, failed run — see gotchas), `TCK-2026-0000002` (confirmed working end-to-end).

Full write-up including every issue hit while building this (Entra ID device-auth error, stale connection references, the Id/Ticket ID field mix-up, "New designer" hiding fields): [`flow-1-ticket-routing.md`](./flow-1-ticket-routing.md).

**Status:** Complete and tested.

### Flow(s) still to build

- `AssignmentMode` branching (Condition on `Ticket Settings`, manual Teams adaptive card vs. auto round-robin using `LastAssignedTo`)
- Writing `Assigned To` / `Status` back to `IT Tickets`
- Escalation handling
- Resolution notice to the requester

*(Each of these will get logged here with its exact configuration once built, same as Flow 1 above.)*

## Phase 3 — App

Not yet started.

## Phase 4 — Teams

Not yet started.

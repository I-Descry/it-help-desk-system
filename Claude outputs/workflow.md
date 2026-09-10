# Workflow

How a ticket moves from submission to resolution.

1. **Submit** — Employee opens the Power Apps app (pinned as a Teams tab) and fills out "New Request." SharePoint's built-in `Created By` field automatically and correctly attributes the ticket to them — no custom "requester" field needed, since there's no service identity submitting on anyone's behalf.

2. **Route** — A Power Automate flow triggers on `When an item is created`, mints a `Ticket ID` (e.g. `TCK-2026-0000001`) off the list's own counter, and checks the `AssignmentMode` setting in `Ticket Settings`. See [`flow-1-ticket-routing.md`](./flow-1-ticket-routing.md) for the built and tested flow.

3. **Assign — manual mode** — The flow posts an adaptive card to whoever `Ticket Settings.ManualAssignmentApprover` currently points to (the Manager by default) with a button for the Specialist and each of the 2 Assistants, plus an FYI note on the card listing anyone currently marked `OutOfOffice`. They tap one; the flow writes `Assigned To` and `Status: Assigned` back to the list.

4. **Assign — auto mode** — The flow round-robins between the **2 Assistants only** (never the Specialist), preferring whichever one currently has fewer open tickets and falling back to strict rotation via `LastAssignedTo` when tied — see [`decisions.md`](./decisions.md#assignment-pool-and-availability) for why. Anyone marked `OutOfOffice` is skipped; if both Assistants are out, it falls back to the Manager. It still sends the Manager an FYI card in Teams so he can reassign if it landed on the wrong person.

5. **Triage & resolve** — The assigned staffer works the ticket from the Power Apps triage screen (visible to IT and the Manager only), moving it through `In Progress`, and writing resolution notes directly onto the record.

6. **Escalate — as needed** — If the assigned staffer can't resolve it, they escalate directly from Power Apps:
   - Either Assistant can escalate to the Specialist, the other Assistant, or the Manager.
   - The Specialist can escalate to either Assistant or the Manager.
   - This reassigns `Assigned To` to the new target, records `Escalated To` (which persists even after `Assigned To` changes again later), sets `Status: Escalated`, and triggers the same Teams notification pattern as a normal assignment, so the new assignee is notified immediately.

7. **Notify & close the loop** — The moment `Status` becomes `Resolved`, the flow messages the original requester directly in Teams — they don't need to reopen the app to find out. The Manager can reassign a ticket, flip `AssignmentMode`, or change the `ManualAssignmentApprover`/`OutOfOffice` settings from Teams, Power Apps, or the list at any point along the way.

## Roles

- **Employees** — submit requests, check status. No special access needed beyond the site itself.
- **IT and Project Assistant Manager** (Charles Caldito Jr.) — default Manual-mode approver, receives Auto-mode FYI cards, can assign or reassign anything at any time, can flip `AssignmentMode`, and is the fallback assignee if both Assistants are out.
- **2 IT Assistants** (Tristan Tan, JP Villacorta) — the Auto-mode round-robin pool; receive assigned tickets, resolve them, or escalate what they can't handle.
- **1 IT Specialist** (John Nikko Alvarez) — never auto-assigned; receives tickets only via the Manager's manual choice or an Assistant's escalation. Escalation target for technically difficult tickets.

## Ticket ID and status conventions

- Ticket IDs look like `TCK-2026-0000001` — year label + a 7-digit, never-resetting counter. See [`decisions.md`](./decisions.md#ticket-id-format) for why it doesn't reset each year.
- Status progression: `New` → `Assigned` → `In Progress` → (optionally `Escalated`) → `Resolved` → `Closed`.

# Workflow

How a ticket moves from submission to resolution.

1. **Submit** — Employee opens the Power Apps app (pinned as a Teams tab) and fills out "New Request." SharePoint's built-in `Created By` field automatically and correctly attributes the ticket to them — no custom "requester" field needed, since there's no service identity submitting on anyone's behalf.

2. **Route** — A Power Automate flow triggers on `When an item is created`, mints a `Ticket ID` (e.g. `TCK-2026-0000001`) off the list's own counter, and checks the `AssignmentMode` setting in `Ticket Settings`. See [`flow-1-ticket-routing.md`](./flow-1-ticket-routing.md) for the built and tested flow.

3. **Assign — manual mode** — The flow posts an adaptive card to the supervisor in Teams with a button for each of the two IT staff. They tap one; the flow writes `Assigned To` and `Status: Assigned` back to the list.

4. **Assign — auto mode** — The flow round-robins between the two staff itself, tracked via `LastAssignedTo` in `Ticket Settings` — no tap needed. It still sends the supervisor an FYI card in Teams so they can reassign if it landed on the wrong person.

5. **Triage & resolve** — The assigned staffer works the ticket from the Power Apps triage screen (visible to IT and the supervisor only), moving it through `In Progress`, and writing resolution notes directly onto the record.

6. **Escalate — as needed** — If the assigned staffer can't resolve it, they escalate directly from Power Apps to either a senior specialist or the IT manager. This:
   - reassigns `Assigned To` to the specialist/manager
   - records `Escalated To`, which persists even after `Assigned To` changes again later
   - sets `Status: Escalated`
   - triggers the same Teams notification pattern as a normal assignment, so the new assignee is notified immediately

7. **Notify & close the loop** — The moment `Status` becomes `Resolved`, the flow messages the original requester directly in Teams — they don't need to reopen the app to find out. The supervisor can reassign a ticket or flip `AssignmentMode` from Teams, Power Apps, or the app at any point along the way.

## Roles

- **Employees** — submit requests, check status. No special access needed beyond the site itself.
- **Supervisor** — sees new tickets, assigns manually or lets auto-assignment run, can reassign or escalate anything at any time, can flip `AssignmentMode`.
- **2 primary IT staff** — receive assigned tickets, resolve them, or escalate what they can't handle.
- **1 senior/specialist** — escalation target for technically difficult tickets.
- **1 IT manager** — escalation target for tickets needing management-level attention or decisions.

## Ticket ID and status conventions

- Ticket IDs look like `TCK-2026-0000001` — year label + a 7-digit, never-resetting counter. See [`decisions.md`](./decisions.md#ticket-id-format) for why it doesn't reset each year.
- Status progression: `New` → `Assigned` → `In Progress` → (optionally `Escalated`) → `Resolved` → `Closed`.

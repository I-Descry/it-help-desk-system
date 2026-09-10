# SharePoint Schema

Two lists, both in the `IT Help Desk` site.

## List: `IT Tickets`

| Column | Type | Notes |
|---|---|---|
| `Subject` | Single line of text (renamed from default `Title`) | Required |
| `Description` | Multiple lines of text (plain) | Required |
| `Category` | Choice | See values below |
| `Priority` | Choice | See values below; default `Medium` |
| `Status` | Choice | See values below; default `New` |
| `Ticket ID` | Single line of text | Left blank on creation; filled by Power Automate |
| `Assigned To` | Person (single) | Current owner of the ticket |
| `Escalated To` | Person (single) | Filled only when a ticket is escalated; preserves escalation history even after `Assigned To` changes again |
| `Resolution Notes` | Multiple lines of text (plain) | |
| `Date Resolved` | Date and time (date only) | |
| `Created By` *(built-in)* | — | Used directly as the requester |
| `Created` *(built-in)* | — | Used directly as the submission timestamp |

### `Category` choices

```
Incident
Service Request
Access Request
Password Reset
Hardware Issue
Printer Issue
Software Issue
Network Issue
Email Issue
Asset Request
Security
Other
```

Consolidated from a legacy system's 21 request types — see [`decisions.md`](./decisions.md) for the full mapping and reasoning.

### `Priority` choices + colors

| Value | Hex |
|---|---|
| Low | `#2E7D32` |
| Medium | `#F9A825` |
| High | `#EF6C00` |
| Urgent | `#C62828` |

### `Status` choices + colors

| Value | Hex |
|---|---|
| New | `#1976D2` |
| Assigned | `#7B1FA2` |
| In Progress | `#F9A825` |
| Escalated | `#AD1457` |
| Resolved | `#2E7D32` |
| Closed | `#616161` |

Colors are intended for badges/pills in the Power Apps screens. Applying them to the raw SharePoint list view is optional and requires column formatting JSON (Column settings → Format this column → Advanced mode).

## List: `Ticket Settings`

Holds exactly one row, controlling the assignment logic.

| Column | Type | Notes |
|---|---|---|
| `Title` | Single line of text (default) | Not functionally used; set to `Default` |
| `AssignmentMode` | Choice — `Manual`, `Auto` | Default `Manual` |
| `LastAssignedTo` | Person (single) | Tracks round-robin state in Auto mode. Rotates between the 2 IT Assistants only — the Specialist is never part of auto-assignment. See [`decisions.md`](./decisions.md#assignment-pool-and-availability). |
| `ManualAssignmentApprover` | Person (single) | Who receives the Teams adaptive card in Manual mode. Defaults to the IT and Project Assistant Manager; changeable by anyone (e.g. to the Specialist) when the Manager is unavailable — no flow edit required. |
| `OutOfOffice` | Person (allow multiple selection) | Anyone among the Specialist + 2 Assistants currently unavailable. Shown as an FYI on the Manual-mode card; skipped when picking who's next in Auto mode. |

`AssignmentMode` colors: `Manual` `#455A64`, `Auto` `#00897B`.

## Staff reference (not a list — referenced directly in Power Automate)

At this team size, staff emails are referenced directly inside the Power Automate flow rather than maintained in a separate SharePoint list. If the team grows meaningfully, a dedicated `IT Staff` list would be the natural next step.

| Role | Name | Email | Assignment pool |
|---|---|---|---|
| IT and Project Assistant Manager | Charles Caldito Jr. | `charles.caldito@nabatifood.com.ph` | Manual-mode approver (default); can assign to anyone, including himself; escalation target |
| IT Specialist | John Nikko Alvarez | `johnnikko.alvarez@nabatifood.com.ph` | Manual assignment only (Manager's choice), or via escalation — never auto-assigned |
| IT Assistant | Tristan Railey Tan | `tristan.tan@nabatifood.com.ph` | Manual or Auto (round-robin pool of 2) |
| IT Assistant | John Paul Villacorta | `jp.villacorta@nabatifood.com.ph` | Manual or Auto (round-robin pool of 2) |

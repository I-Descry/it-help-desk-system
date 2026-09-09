# Architecture Decisions

The reasoning behind the key choices in this build, so future changes don't accidentally undo them.

## Why no custom web app

An earlier version of this plan included a custom web app as one of the submission channels. It was dropped because it was the only piece that ever required admin involvement: a web app's backend has no signed-in user, so it needs its own identity (an Entra ID app registration) to read or write SharePoint data, and granting that identity access requires a Microsoft 365 admin to consent to a `Sites.Selected` permission and grant it to the specific site.

Every other piece — Power Apps, Power Automate, Teams — connects to SharePoint using the standard SharePoint connector, authenticating as whoever is actually signed in. No background service identity means no app registration, no admin favor, and the whole system can be built and maintained by one person with a standard Microsoft 365 license.

If a web app is ever added back later, it can still plug into the same `IT Tickets` list — that's the one path that brings back the Entra ID app registration requirement.

## Why no separate database

SharePoint Online storage is pooled at roughly 1 TB plus 10 GB per licensed user, and a single SharePoint list can hold up to 30 million items — ticket records (a few KB of text each) will never meaningfully approach either limit. The only thing that meaningfully consumes storage over time is file attachments, not the records themselves.

More importantly, an external database (MySQL, XAMPP, Azure SQL, etc.) can't be reached from Power Automate or Power Apps without the **SQL Server connector, which is a Premium connector** — using one would reintroduce the exact licensing cost this design avoids, on top of creating a second data store that could drift out of sync with the first. XAMPP specifically is a local development tool (bundles Apache, MySQL, PHP for offline testing) with no built-in remote access, backups, or uptime guarantees — not suited for production use regardless of licensing.

## Why attribution ("who submitted this") needs no extra field

Because every submission goes through the employee's own Microsoft sign-in (Power Apps or a Microsoft Forms link), SharePoint's built-in `Created By` field is always accurate automatically. This wasn't true in the web-app version of the plan, where a service identity submitting on the employee's behalf would have made `Created By` show the app, not the person — that version required a separate `Requester` field as a workaround. Dropping the web app removed the need for that workaround entirely.

## Category consolidation

The original category list, carried over from a legacy ticketing system, had 21 values with significant overlap across different dimensions (request *type* vs. affected *system* vs. *kind of work*). It was consolidated to 12 values, with every original value still represented:

| Legacy value(s) | Consolidated into |
|---|---|
| Incident, Troubleshooting | `Incident` |
| Service Request, Configuration, Maintenance, Upgrade, Backup & Restore | `Service Request` |
| Access Request, Account Creation, Account Deactivation | `Access Request` |
| Password Reset | `Password Reset` (kept separate — typically the single highest-volume ticket type) |
| Hardware Issue | `Hardware Issue` |
| Printer Issue | `Printer Issue` (kept separate by request, rather than folded into Hardware Issue) |
| Software Issue, Installation | `Software Issue` |
| Network Issue | `Network Issue` |
| Email Issue | `Email Issue` |
| Asset Management, Relocation / Deployment, Data Migration | `Asset Request` |
| Security | `Security` (kept separate deliberately, so it isn't buried under generic Incident and can be triaged/escalated faster) |
| *(new)* | `Other` — catch-all safety net |

Firewall/URL-unblock requests fall under `Access Request` in this scheme.

## Escalation design

Escalation reuses existing fields rather than introducing new infrastructure: `Assigned To` is reassigned to the specialist or manager (the same Person field used for normal assignment), a new `Escalated To` field preserves the record of who it was escalated to even after `Assigned To` changes again, and `Status: Escalated` makes it visible at a glance. No new list, no new automation pattern — the Teams notification on escalation reuses the same flow logic as a normal assignment notification.

## Ticket ID format

Ticket IDs use the pattern `TCK-2026-0000001` — a 4-digit year followed by a 7-digit, zero-padded, never-resetting counter.

The counter is built off the SharePoint list's own `ID` column, which is a global, all-time counter that never resets each calendar year on its own. An earlier version of this design used a per-year 3-digit counter (`TCK-2026-014`), but that breaks in two ways: first, once the list's raw `ID` passes 999, a fixed 3-digit format would either overflow or — if implemented as a simple "take the last 3 digits" truncation — silently collide (ticket #1042 and ticket #42 would both render as `042`). Second, because the underlying `ID` never resets per year, "how many digits a ticket needs" is really a function of the list's total lifetime volume, not that year's volume — so a per-year reset wasn't actually achievable off this field without extra flow logic (a separate "count items created this calendar year" query on every run).

The chosen format keeps things simple: the year is just a label, refreshed automatically from `utcNow()` at the moment each ticket is created (so it updates on its own every January with no manual step), while the number itself is a single continuous count across the list's entire history — closer to how systems like Jira or Zendesk actually number tickets. The 7-digit padding is generous headroom, and the underlying expression falls back to the unpadded number if the count ever exceeds 9,999,999, so it grows instead of colliding.

## Why two IT staff aren't in a separate list

At a team size of two primary staff (plus one specialist and one manager for escalation), maintaining a full `IT Staff` SharePoint list would be unnecessary overhead. Staff are referenced directly by email inside the Power Automate flow. This is a candidate to revisit if the team grows.

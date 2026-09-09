# IT Ticketing System — Documentation

An IT ticketing system built entirely on Microsoft 365 — SharePoint, Power Automate, Power Apps, and Teams. No custom web app, no external database, no admin dependency for day-to-day operation.

**Site:** `IT Help Desk` (SharePoint Team site)

## In this folder

- [`workflow.md`](./workflow.md) — how a ticket moves from submission to resolution, including assignment and escalation
- [`sharepoint-schema.md`](./sharepoint-schema.md) — full column-by-column schema for both SharePoint lists, choice values, and colors
- [`decisions.md`](./decisions.md) — the reasoning behind the key architecture choices (why no web app, no separate database, no admin dependency, ticket ID format)
- [`roadmap.md`](./roadmap.md) — the four build phases and current progress
- [`flow-1-ticket-routing.md`](./flow-1-ticket-routing.md) — the built and tested Power Automate flow for Ticket ID minting, including exact expressions and build gotchas
- [`build-log.md`](./build-log.md) — running chronological log of exact configuration values, expressions, and identifiers as each piece gets built; the quickest place to find "what did we actually set that to"

## One-paragraph summary

Employees submit and check ticket status from a Power Apps app pinned inside Teams. A supervisor assigns each ticket to one of two IT staff — manually via a Teams card, or automatically via round-robin when they're busy or away. IT staff who hit something beyond their ability can escalate a ticket to a senior specialist or the IT manager. Everything reads and writes one shared SharePoint list, so nothing needs syncing, and because every piece authenticates as whoever is actually signed in, no Entra ID app registration or admin favor is required.

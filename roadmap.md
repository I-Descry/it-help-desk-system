# Build Roadmap

Four phases, built in order — each one depends on the last.

- [x] **Phase 1 — Data.** Two SharePoint lists (`IT Tickets`, `Ticket Settings`) with all columns, choice values, and colors defined. See [`sharepoint-schema.md`](./sharepoint-schema.md). **Complete.**

- [ ] **Phase 2 — Automation.** Power Automate flow(s) covering:
  - [x] Ticket ID minting on creation — built and tested (Flow 1). See [`flow-1-ticket-routing.md`](./flow-1-ticket-routing.md).
  - [ ] `AssignmentMode` branching (manual Teams card vs. auto round-robin)
  - [ ] Writing `Assigned To` / `Status` back to the list
  - [ ] Escalation handling (reassignment + `Escalated To` + notification)
  - [ ] Resolution notice to the original requester

- [ ] **Phase 3 — App.** One Power Apps canvas app:
  - "New Request" and "My Tickets" screens, visible to everyone
  - A role-gated triage + escalation screen, visible to IT staff and the supervisor only (via Microsoft 365 group membership check)

- [ ] **Phase 4 — Teams.** Pin the Power Apps app as a Teams tab; confirm all notification types (assignment, escalation, resolution) land correctly.

Update the checkboxes above as each phase completes.

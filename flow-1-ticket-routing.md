# Flow 1 — Ticket ID Minting

The first Power Automate flow built for Phase 2. Built, saved, and tested against two live test tickets in the `IT Tickets` list.

**Flow name:** `IT Ticket - New Request Routing`

## What it does

On every new item in `IT Tickets`, generates a unique, never-resetting Ticket ID and writes it back onto that same record — nothing else on the record is touched.

## Trigger

**When an item is created** (SharePoint)

| Field | Value |
|---|---|
| Site Address | `IT Help Desk` — `https://nabatisnackcoid.sharepoint.com/sites/ITHelpDesk` |
| List Name | `IT Tickets` |

## Actions

### 1. Compose - Ticket ID

Builds the ID string. Renamed from the default "Compose" so the flow stays readable as more steps get added.

**Inputs** (expression):

```
concat('TCK-', formatDateTime(utcNow(), 'yyyy'), '-', if(greaterOrEquals(length(string(triggerOutputs()?['body/ID'])), 7), string(triggerOutputs()?['body/ID']), substring(concat('000000', string(triggerOutputs()?['body/ID'])), sub(length(concat('000000', string(triggerOutputs()?['body/ID']))), 7), 7)))
```

Produces IDs like `TCK-2026-0000001`. See [`decisions.md`](./decisions.md#ticket-id-format) for why it's shaped this way (year label + continuous 7-digit counter, no per-year reset).

### 2. Update item (SharePoint)

Writes the composed ID back onto the triggering record.

| Field | Value |
|---|---|
| Site Address | `IT Help Desk` (same as trigger) |
| List Name | `IT Tickets` (same as trigger) |
| Id | Expression: `triggerOutputs()?['body/ID']` — the SharePoint item number of the ticket that triggered this run |
| Ticket ID | Dynamic content: **Outputs** (from Compose - Ticket ID) |
| *(every other field)* | Left blank — SharePoint's Update item action only touches fields you explicitly set, so this can't overwrite anything else on the record |

## Tested

Two live test tickets confirmed the flow end-to-end:

- `Test Ticket 1` — first attempt; the Update item action ran "successfully" but wrote nothing, because a mid-build reconnect had reset its field bindings back to blank (see gotchas below). Left as-is in the list as a record of the issue.
- `Test Ticket 2` — after the fix, correctly produced and wrote `TCK-2026-0000002` to the `Ticket ID` column.

Both test tickets can be deleted once the rest of Phase 2 is built and tested.

## Build notes / gotchas

Worth keeping in mind if this flow ever needs editing, or if the same issues resurface on later flows in this system:

- **`AADSTS135011: Device used during the authentication is disabled`** — hit this trying to save the flow. It's an Entra ID (Azure AD) conditional-access block on the specific device/browser, unrelated to the flow's design. Fixed via **Power Automate → Data → Connections → SharePoint connection → "..." → Switch account**, which forces a full fresh sign-in (a "repair/fix connection" option wasn't available in this environment). If it recurs and re-signing in throws the same error, that means the device itself is blocked at the tenant level and needs either a different device/browser or the tenant admin to re-enable it in Entra ID.
- **Stale Site Address after reconnecting** — right after the account switch, both the trigger and Update item cards showed "SharePoint Site Address ... is not valid" on save, even though the field displayed the correct-looking URL. Fix was to clear the field completely and reselect the site fresh from the dropdown rather than trusting the previously-saved value.
- **Re-selecting Site Address / List Name resets field bindings below it** — when the Update item card's Site Address and List Name were cleared and reselected, the Id and Ticket ID field values underneath were also wiped (this is what caused the first test ticket to silently fail — the action still ran, but with no fields to actually update). Any time the site/list selectors on an already-configured SharePoint action get touched again, re-check every field below it before saving.
- **"New designer" mode hides fields behind Advanced parameters** — the designer occasionally switched into a mode where most of a SharePoint action's list fields are collapsed behind an "Advanced parameters → Show all" control instead of being listed directly. If an expected field (like Ticket ID) doesn't appear on the card, check for that toggle before assuming the field is missing or was deleted.
- **Id vs. Ticket ID mix-up** — after fixing the above, it's easy to accidentally swap which field gets which value. `Id` (singular, required, tells the action *which* item to update) should hold `triggerOutputs()?['body/ID']`. `Ticket ID` (the actual list column being written to) should hold **Outputs** from the Compose step. Worth double-checking the icon on each chip — the trigger's fields show a teal SharePoint icon, Compose's Outputs shows a purple `{}` icon.

## Next

`AssignmentMode` branching — pulling the setting from `Ticket Settings` and splitting into the manual Teams-card path vs. the auto round-robin path. Not yet built.

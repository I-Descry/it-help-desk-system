# Flow 1 — Ticket Routing

The core Power Automate flow for Phase 2. Started as Ticket ID minting only; now covers the full routing logic — Manual approval via Teams and Auto round-robin — end to end. Built, saved, and tested against live test tickets in the `IT Tickets` list throughout.

**Flow name:** `ITHD-01 - New Ticketing Routing` (originally built and documented as `IT Ticket - New Request Routing`; renamed once the flow grew past ID minting alone).

## What it does

On every new item in `IT Tickets`: mints a unique, never-resetting Ticket ID and writes it back onto the record, then reads `Ticket Settings` and branches on `AssignmentMode`.

- **Manual mode** — posts an adaptive card to `ManualAssignmentApprover` in Teams listing the ticket details and who's currently out of office, waits for them to pick Nikko/Tristan/JP, then writes `Assigned To` and `Status: Assigned` back to the ticket.
- **Auto mode** — round-robins strictly between the two Assistants (Tristan, JP — the Specialist is never auto-assigned, see [`decisions.md`](./decisions.md#assignment-pool-and-availability)). Skips whoever is in `OutOfOffice`, prefers whoever has fewer currently-open tickets, and alternates via `LastAssignedTo` on a tie. If both Assistants are out, falls back to notifying `ManualAssignmentApprover` directly with an FYI card (no buttons — it's informational, not a choice) and assigns the ticket to them. Either way, writes `Assigned To` / `Status: Assigned` to the ticket and updates `LastAssignedTo` on `Ticket Settings`.

## Trigger

**When an item is created** (SharePoint)

| Field | Value |
|---|---|
| Site Address | `IT Help Desk` — `https://nabatisnackcoid.sharepoint.com/sites/ITHelpDesk` |
| List Name | `IT Tickets` |

## Actions

### 1. Compose - Ticket ID

```
concat('TCK-', formatDateTime(utcNow(), 'yyyy'), '-', if(greaterOrEquals(length(string(triggerOutputs()?['body/ID'])), 7), string(triggerOutputs()?['body/ID']), substring(concat('000000', string(triggerOutputs()?['body/ID'])), sub(length(concat('000000', string(triggerOutputs()?['body/ID']))), 7), 7)))
```

Produces IDs like `TCK-2026-0000001`. See [`decisions.md`](./decisions.md#ticket-id-format).

### 2. Update item (SharePoint)

Writes the composed ID back onto the triggering record. `Id` = `triggerOutputs()?['body/ID']`; `Ticket ID` = Outputs of Compose - Ticket ID; every other field left blank.

### 3. Get - Ticket Settings (SharePoint)

Plain "Get items" against `Ticket Settings`, no filter — the list holds exactly one row. Feeds the `Apply to each` below (the loop only ever runs once, but a loop is required because `Get items` returns an array even for a single-row list, and array items are how the `AssignmentMode`, `ManualAssignmentApprover`, `LastAssignedTo`, and `OutOfOffice` values become available as dynamic content downstream).

### 4. Initialize - Tristan Open Count / Initialize - JP Open Count

Two `Initialize variable` actions, both Integer, both starting at `0`. **Must sit at the true top level of the flow** — between `Get - Ticket Settings` and the `Apply to each`, not nested inside it. See gotchas below.

### 5. Apply to each (over `Get - Ticket Settings`'s value — internal name `Apply_to_each`)

#### Condition: `AssignmentMode` is equal to `Manual`

##### If yes — Manual branch

1. **Compose - Out of Office List** — builds the "who's currently out" text shown on the card.
2. **Compose - Category**: `triggerOutputs()?['body/Category']?['Value']`
3. **Compose - Priority**: `triggerOutputs()?['body/Priority']?['Value']`
4. **Compose - Build Adaptive Card** — the whole card as one `concat()` string:

   ```
   concat(
   '{"type":"AdaptiveCard","$schema":"http://adaptivecards.io/schemas/adaptive-card.json","version":"1.4","body":[{"type":"TextBlock","text":"New IT Ticket - Needs Assignment","weight":"Bolder","size":"Medium","wrap":true},{"type":"FactSet","facts":[{"title":"Ticket ID:","value":"',
   outputs('Compose_-_Ticket_ID'),
   '"},{"title":"Subject:","value":"',
   triggerOutputs()?['body/Title'],
   '"},{"title":"Category:","value":"',
   outputs('Compose_-_Category'),
   '"},{"title":"Priority:","value":"',
   outputs('Compose_-_Priority'),
   '"},{"title":"Submitted by:","value":"',
   triggerOutputs()?['body/Author']?['DisplayName'],
   '"}]},{"type":"TextBlock","text":"Description:","weight":"Bolder","wrap":true},{"type":"TextBlock","text":"',
   replace(coalesce(triggerOutputs()?['body/Description'], ''), '"', ''),
   '","wrap":true},{"type":"TextBlock","text":"Currently out of office: ',
   outputs('Compose_-_Out_of_Office_List'),
   '","wrap":true,"isSubtle":true,"spacing":"Medium"},{"type":"TextBlock","text":"Assign this ticket to:","weight":"Bolder","spacing":"Medium"}],"actions":[{"type":"Action.Submit","title":"Nikko (Specialist)","data":{"assignee":"johnnikko.alvarez@nabatifood.com.ph","assigneeName":"John Nikko Alvarez"}},{"type":"Action.Submit","title":"Tristan (Assistant)","data":{"assignee":"tristan.tan@nabatifood.com.ph","assigneeName":"Tristan Railey Tan"}},{"type":"Action.Submit","title":"JP (Assistant)","data":{"assignee":"jp.villacorta@nabatifood.com.ph","assigneeName":"John Paul Villacorta"}}]}'
   )
   ```

5. **Post adaptive card and wait for a response** (Teams) — Post as: Flow bot; Post in: Chat with Flow bot; Recipient: `ManualAssignmentApprover Email` (inserted via the dynamic-content picker); Message: a single dynamic-content reference to Compose - Build Adaptive Card's Outputs.
6. **Update item 2** (SharePoint) — `Id`: `triggerOutputs()?['body/ID']`; `Status`: `Assigned`; `Assigned To` Claims: `concat('i:0#.f|membership|', body('Post_adaptive_card_and_wait_for_a_response')?['data']?['assignee'])`.

##### If no — Auto branch

1. **Get - Open Tickets** (SharePoint `Get items`, `IT Tickets`) — Filter Query: `Status eq 'Assigned' or Status eq 'In Progress'`.
2. **Apply to each - Count Open Tickets** (over Get - Open Tickets' value):
   - **Condition - Is Tristan**: `Assigned To Email` (picker-inserted) is equal to `tristan.tan@nabatifood.com.ph` → if yes, **Increment - Tristan Count** (`TristanOpenCount`, by `1`).
   - **Condition - Is JP**: `Assigned To Email` (picker-inserted) is equal to `jp.villacorta@nabatifood.com.ph` → if yes, **Increment - JP Count** (`JPOpenCount`, by `1`).
3. **Compose - Is Tristan Out**:
   ```
   contains(string(coalesce(items('Apply_to_each')?['OutOfOffice'], json('[]'))), 'tristan.tan@nabatifood.com.ph')
   ```
4. **Compose - Is JP Out**:
   ```
   contains(string(coalesce(items('Apply_to_each')?['OutOfOffice'], json('[]'))), 'jp.villacorta@nabatifood.com.ph')
   ```
5. **Compose - Determine Assignee**:
   ```
   if(
     and(outputs('Compose_-_Is_Tristan_Out'), outputs('Compose_-_Is_JP_Out')),
     items('Apply_to_each')?['ManualAssignmentApprover']?['Email'],
     if(
       outputs('Compose_-_Is_Tristan_Out'),
       'jp.villacorta@nabatifood.com.ph',
       if(
         outputs('Compose_-_Is_JP_Out'),
         'tristan.tan@nabatifood.com.ph',
         if(
           less(variables('TristanOpenCount'), variables('JPOpenCount')),
           'tristan.tan@nabatifood.com.ph',
           if(
             less(variables('JPOpenCount'), variables('TristanOpenCount')),
             'jp.villacorta@nabatifood.com.ph',
             if(
               equals(items('Apply_to_each')?['LastAssignedTo']?['Email'], 'tristan.tan@nabatifood.com.ph'),
               'jp.villacorta@nabatifood.com.ph',
               'tristan.tan@nabatifood.com.ph'
             )
           )
         )
       )
     )
   )
   ```
6. **Compose - Determine Assignee Name**:
   ```
   if(
     equals(outputs('Compose_-_Determine_Assignee'), 'tristan.tan@nabatifood.com.ph'),
     'Tristan Railey Tan',
     if(
       equals(outputs('Compose_-_Determine_Assignee'), 'jp.villacorta@nabatifood.com.ph'),
       'John Paul Villacorta',
       items('Apply_to_each')?['ManualAssignmentApprover']?['DisplayName']
     )
   )
   ```
7. **Condition - Manager Fallback**: `outputs('Compose_-_Is_Tristan_Out')` is equal to `true` **And** `outputs('Compose_-_Is_JP_Out')` is equal to `true`.
   - **If yes**:
     - **Compose - Manager FYI Card**:
       ```
       concat(
       '{"type":"AdaptiveCard","$schema":"http://adaptivecards.io/schemas/adaptive-card.json","version":"1.4","body":[{"type":"TextBlock","text":"Auto-assigned to you - both Assistants are currently out of office","weight":"Bolder","size":"Medium","wrap":true},{"type":"FactSet","facts":[{"title":"Ticket ID:","value":"',
       outputs('Compose_-_Ticket_ID'),
       '"},{"title":"Subject:","value":"',
       triggerOutputs()?['body/Title'],
       '"},{"title":"Assigned To:","value":"',
       outputs('Compose_-_Determine_Assignee_Name'),
       '"}]},{"type":"TextBlock","text":"This will stay in your queue until one of them is back, unless you reassign it.","wrap":true,"isSubtle":true,"spacing":"Medium"}]}'
       )
       ```
     - **Post card in a chat or channel** (Teams) — Post as: Flow bot; Post in: Chat with Flow bot; Recipient: `ManualAssignmentApprover Email`; Adaptive Card: a single dynamic-content reference to Compose - Manager FYI Card's Outputs. No buttons — this card is informational only, unlike the Manual-mode card.
   - **If no**: empty.
8. **Update - Ticket Assignment** (SharePoint, `IT Tickets`) — `Id`: `triggerOutputs()?['body/ID']`; `Status`: `Assigned`; `Assigned To` Claims: `concat('i:0#.f|membership|', outputs('Compose_-_Determine_Assignee'))`.
9. **Update - Last Assigned** (SharePoint, `Ticket Settings`) — `Id`: `items('Apply_to_each')?['ID']`; `LastAssignedTo` Claims: `concat('i:0#.f|membership|', outputs('Compose_-_Determine_Assignee'))`.

## Tested

- `Test Ticket 1` / `Test Ticket 2` — ID minting only, see gotchas below for the first failed attempt.
- `Test Ticket 13` (`TCK-2026-0000013`) — Manual-mode card rendered correctly with all real data (Category/Priority/Ticket ID/Submitted-by all resolved after the Category+Priority raw-object fix below); button click correctly wrote `Assigned To` / `Status: Assigned`.
- `Test Ticket 15` — Auto mode, both Assistants available, no tie → correctly assigned to Tristan; `LastAssignedTo` confirmed written.
- `Test Ticket 16` — Auto mode, one Assistant (Tristan) out → correctly assigned to JP.
- `Test Ticket 17` (`TCK-2026-0000017`) — Auto mode, both Assistants out → FYI card sent correctly, but "Assigned To" rendered blank and `Update - Ticket Assignment` failed with `BadRequest` (empty Claims value) — this was the flat-vs-nested-bracket bug on `ManualAssignmentApprover`/`LastAssignedTo`, see gotchas.
- `Test Ticket 18` (`TCK-2026-0000018`) — same both-out scenario, after the fix: FYI card correctly showed "Assigned To: John Paul Villacorta," and the full run (including `Update - Ticket Assignment` and `Update - Last Assigned`) completed with no errors.

All three Auto-mode paths and the full Manual-mode path are now confirmed working end to end.

## Build notes / gotchas

Worth keeping in mind if this flow ever needs editing, or if the same issues resurface on later flows in this system:

- **`AADSTS135011: Device used during the authentication is disabled`** — hit this trying to save the flow. It's an Entra ID (Azure AD) conditional-access block on the specific device/browser, unrelated to the flow's design. Fixed via **Power Automate → Data → Connections → SharePoint connection → "..." → Switch account**, which forces a full fresh sign-in (a "repair/fix connection" option wasn't available in this environment). If it recurs and re-signing in throws the same error, that means the device itself is blocked at the tenant level and needs either a different device/browser or the tenant admin to re-enable it in Entra ID.
- **Stale Site Address after reconnecting** — right after the account switch, both the trigger and Update item cards showed "SharePoint Site Address ... is not valid" on save, even though the field displayed the correct-looking URL. Fix was to clear the field completely and reselect the site fresh from the dropdown rather than trusting the previously-saved value.
- **Re-selecting Site Address / List Name resets field bindings below it** — when a SharePoint action's Site Address and List Name are cleared and reselected, the fields underneath are wiped too. Any time the site/list selectors on an already-configured SharePoint action get touched again, re-check every field below it before saving.
- **"New designer" mode hides fields behind Advanced parameters** — the designer occasionally switches into a mode where most of a SharePoint action's list fields are collapsed behind an "Advanced parameters → Show all" control instead of being listed directly. If an expected field doesn't appear on the card, check for that toggle before assuming the field is missing or was deleted.
- **Id vs. Ticket ID mix-up** — `Id` (singular, required, tells the action *which* item to update) should hold `triggerOutputs()?['body/ID']`. `Ticket ID` (the actual list column being written to) should hold **Outputs** from the Compose step.
- **Flat "Field SubProperty" dynamic-content strings only work when inserted via the picker — never hand-type them.** SharePoint Choice and Person fields resolve correctly when picked from the dynamic-content UI (which silently emits nested bracket access), but the same-looking flat string typed by hand — `'Category Value'`, `'Priority Value'`, `'ManualAssignmentApprover Email'`, `'LastAssignedTo Email'`, `'ManualAssignmentApprover DisplayName'` — resolves to nothing, silently (`?[]` returns null rather than erroring). This surfaced three separate times: Category and Priority coming through as raw SharePoint objects in the adaptive card (traced via the resolved Compose output in run history, which showed the actual JSON object instead of plain text), and later `ManualAssignmentApprover`/`LastAssignedTo` inside `Compose - Determine Assignee` and `Compose - Determine Assignee Name`, which produced an empty `Assigned To` Claims value and a `BadRequest` on `Update - Ticket Assignment`. **Fix, every time**: hand-typed expressions need nested bracket access instead — `['Category']?['Value']`, `['ManualAssignmentApprover']?['Email']`, `['LastAssignedTo']?['Email']`, `['ManualAssignmentApprover']?['DisplayName']`. Because this bug produces no error at its own step, the most reliable way to catch it is checking the actual resolved Outputs of the relevant Compose action in run history (not just the expression text) whenever a downstream card field or SharePoint write is unexpectedly blank.
- **"Post a message in a chat or channel" doesn't exist in this environment's Teams connector** (confirmed by scrolling the full action list). The correct one-way (fire-and-forget) action is **"Post card in a chat or channel,"** which takes the same Post as / Post in / Recipient / Adaptive Card shape as "Post adaptive card and wait for a response," minus the wait/response handling.
- **"Initialize variable" must be at the flow's true top level** — it errors ("can only be used at top level") if placed inside a Condition or an Apply to each. `TristanOpenCount`/`JPOpenCount` had to move up to sit directly between `Get - Ticket Settings` and the outer `Apply to each`.
- **`select()`/`item()` failure inside a Compose action, cause unresolved.** `Compose - Is Tristan Out` originally used `select()` and failed with `InvalidTemplate ... 'select' is not defined or not valid` at "line 0 column 0," even though Peek Code confirmed the stored expression was byte-identical and structurally the same as an already-working `select()` elsewhere in the flow. Hypothesized (not confirmed) to be scope confusion from a second `Apply to each` loop now present earlier in the same branch. **Workaround**: avoid `select()`/`item()` entirely — stringify the array and substring-search it instead: `contains(string(coalesce(items('Apply_to_each')?['OutOfOffice'], json('[]'))), 'someone@domain.com')`. Confirmed reliable across every subsequent test run. Worth retrying `select()` again someday if the flow's structure changes significantly, since the root cause was never pinned down.
- **`OutOfOffice` initially rejected a second person.** Diagnosed as the SharePoint column defaulting to `Allow multiple selections = No`. Fix: List → column header dropdown → **Column settings → Edit → Allow multiple selections → Yes**.
- **Adaptive card architecture**: build the entire card JSON as one `concat()` expression inside a single Compose action, then reference that Compose's Outputs as the *only* thing in the Teams action's Message/Adaptive Card field. Trying to assemble the JSON inline across multiple dynamic-content insertions was what caused the original `InvalidJsonInBotAdaptiveCard` errors. This pattern is now standard for both the Manual-mode card and the Auto-mode Manager FYI card, and should be reused for any future Teams card in this flow (escalation, resolution notice).

## Next

- `ActingApprover` field (Person, on `Ticket Settings`) — lets the Manual-mode card be delegated to someone else (e.g. Nikko) while `ManualAssignmentApprover` (Charles) is out, without changing the actual approver setting. Not yet built.
- Escalation handling.
- Resolution notice to the requester.

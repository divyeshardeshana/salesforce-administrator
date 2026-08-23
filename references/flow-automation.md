# Flow & Declarative Automation

---

## Automation Tool Landscape

Salesforce has consolidated point-and-click automation into Flow. Current status of the
legacy tools:

- **Workflow Rules and Process Builder** — creation is disabled. Salesforce set
  31 December 2025 as the end-of-support date. This is *end of support*, not retirement:
  existing automations continue to run, but no bug fixes, patches, or enhancements are
  issued after that date. Salesforce may retire them fully later.
- **Migrate to Flow tool** — converts existing automations 1:1. One hundred Workflow Rules
  produce one hundred Flows. Useful for trivial single-action rules; actively harmful as a
  migration strategy, because it multiplies the maintenance problem instead of solving it.

The correct approach is **rebuild and enhance**: inventory every legacy automation per
object, map what each actually does, and consolidate into one before-save and one
after-save Flow per object.

### Flow Types

| Type | Trigger | Primary use |
|---|---|---|
| **Record-Triggered** | Record created, updated, or deleted | The workhorse. Field updates, related-record changes, notifications |
| **Screen Flow** | User launches it | Guided data entry, wizards, call scripts, Experience Cloud forms |
| **Scheduled-Triggered** | Time-based, optionally with a filter | Nightly hygiene, batch updates, recurring reports data prep |
| **Platform Event-Triggered** | A platform event message | Integration responses, decoupled architectures |
| **Autolaunched (no trigger)** | Called by another Flow, Apex, REST, or a button | Reusable subflow logic |
| **Record-Triggered Orchestration** | Record change, multi-stage | Long-running processes with human steps across days |

---

## Record-Triggered Flow Architecture

### Before-Save vs After-Save

This is the single most consequential choice in declarative automation.

| | Before-Save (Fast Field Updates) | After-Save (Actions and Related Records) |
|---|---|---|
| When it runs | Before the record is written to the database | After the record is committed |
| Modify the triggering record | Assign directly to `$Record` — no DML | Requires an Update Records element — extra DML |
| Record ID available | No, on insert | Yes |
| Can create/update related records | No | Yes |
| Can send email, post to Chatter, call actions | No | Yes |
| Can call Apex | No | Yes |
| Relative performance | Roughly 10x faster | Baseline |
| Recursion risk | None | Real — guard for it |

**Rule: if the automation only sets fields on the record being saved, it must be
before-save.** Converting after-save field updates to before-save is often the single
highest-impact performance fix available in an inherited org.

### One Flow Per Object Per Context

Multiple record-triggered Flows on the same object and trigger point have **no guaranteed
execution order**. Flow Trigger Order (the `Trigger Order` field, 1–2,000) makes order
deterministic, but relying on it produces a fragile web of interdependencies.

Preferred architecture:

```
Opportunity
├── Opportunity_BeforeSave_Main        (Fast Field Updates)
│     Decision: What changed?
│       ├── Amount changed        → set Discount_Tier__c, Requires_Approval__c
│       ├── Stage → Closed Won    → set Actual_Close_Date__c
│       └── Owner changed         → set Owner_Region__c
│
└── Opportunity_AfterSave_Main         (Actions and Related Records)
      Decision: What changed?
        ├── Stage → Closed Won  → create Renewal Opportunity, post to Chatter
        ├── Stage → Closed Lost → update Account.Last_Loss_Reason__c
        └── Amount > 100,000    → submit for approval
```

Exception where separate flows are defensible: a genuinely independent concern owned by a
different team or shipped in a managed package. Document the trigger order when you do.

### Entry Criteria

Entry criteria are evaluated before the Flow starts and cost almost nothing. Filtering
inside the Flow with a Decision element costs a Flow interview per record.

Three configuration options:

1. **All conditions / any conditions** — simple field comparisons
2. **Custom condition logic** — `(1 AND 2) OR 3`
3. **Formula evaluates to true** — the most powerful; gives access to `ISCHANGED()`,
   `PRIORVALUE()`, `ISNEW()`

```
Formula entry criteria:
  ISNEW()
  || ISCHANGED({!$Record.StageName})
  || (ISCHANGED({!$Record.Amount}) && {!$Record.Amount} > 50000)
```

Also set **"When to Run the Flow for Updated Records"**:

- *Every time a record is updated* — default, runs on every save
- *Only when a record is updated to meet the condition requirements* — runs only on the
  transition into the criteria. This is the equivalent of the old "is changed" evaluation
  and prevents an automation firing repeatedly while a record sits in a qualifying state.

Choosing the second option is usually correct and is a frequent source of duplicate-action
bugs when missed.

### Recursion Control

After-save Flows that update the triggering record re-enter the trigger. Salesforce has
recursion detection, but relying on it produces unpredictable behaviour.

Guards, in order of preference:

1. **Move the logic to before-save.** Eliminates the problem entirely.
2. **Tighten entry criteria.** `ISCHANGED()` on the specific field, plus a check that the
   field is not already at the target value.
3. **Use a checkbox flag field** — `Automation_Processed__c` — set in the same transaction
   and checked in entry criteria. Requires cleanup logic.

### Fault Paths

Every element that performs DML or invokes an action can fail. Without a fault path, the
failure surfaces as an unhandled Flow exception — an ugly modal for the user and an email
to the Flow's last modifier.

```
Update Records
  ├── Success → continue
  └── Fault   → Assignment: build error text from {!$Flow.FaultMessage}
                → Create Records: Error_Log__c
                → Screen (if screen flow): friendly message
                   or Send Email to admin (if record-triggered)
```

For record-triggered Flows, a custom `Error_Log__c` object beats email notification —
searchable, reportable, and it does not depend on someone reading their inbox.

Set the **Flow error email recipient** deliberately: Setup → Process Automation Settings →
"Send Process or Flow Error Email to" → Apex Exception Email recipients, not just the last
modifier. Otherwise errors follow whoever last touched the Flow.

---

## Flow Elements Reference

### Data Elements

| Element | Purpose | Key settings |
|---|---|---|
| **Get Records** | Query records | Store all fields vs selected fields; first record vs all; sort order |
| **Create Records** | Insert | Single record from fields, or from a collection variable |
| **Update Records** | Update | By ID, by criteria, or from a collection |
| **Delete Records** | Delete | By ID or by criteria — dangerous with criteria; always test |

Performance settings on Get Records that matter:

- **"Automatically store all fields"** retrieves every field on the object. On a wide
  object this is measurable overhead. Choose fields explicitly for anything running at
  volume.
- **"How many records to store: Only the first record"** returns a record variable rather
  than a collection — simpler downstream and cheaper.
- Always handle the **no-records-found** case. A Get Records that finds nothing returns
  null, and referencing a field on it throws at runtime.

### Logic Elements

| Element | Purpose | Notes |
|---|---|---|
| **Decision** | Branch | Outcomes evaluate in order; first match wins. Always define the default outcome meaningfully |
| **Assignment** | Set variable values | Also used to add items to collections (`Add` operator) |
| **Loop** | Iterate a collection | Never put DML inside |
| **Collection Sort** | Order a collection | Also supports limiting to first N |
| **Collection Filter** | Subset a collection | Avoids a Loop + Decision pattern |
| **Transform** | Map one collection shape to another | Cleaner than Loop + Assignment for reshaping |

### Interaction and Action Elements

| Element | Purpose |
|---|---|
| **Screen** | Display and collect input (screen flows only) |
| **Action** | Send Email, Post to Chatter, Submit for Approval, Send Custom Notification, invoke Apex, call External Service |
| **Subflow** | Call an autolaunched flow — the reuse mechanism |
| **Pause** | Wait for a time or a platform event (async paths) |
| **Custom Error** | Throw a validation-style error from a before-save flow |

**Custom Error** deserves attention: it lets a before-save record-triggered Flow block a
save with a message, either at record level or on a specific field. This covers validation
requirements that exceed what a validation rule formula can express — cross-object checks,
queries, or multi-condition logic — without writing Apex.

---

## Bulkification

A Flow runs once per batch of up to 200 records, with elements executing in bulk behind
the scenes. This holds only if the Flow is built correctly.

### The Cardinal Rule

**No DML and no Get Records inside a Loop.**

```
WRONG — one DML per iteration, hits the 150 DML statement limit at 151 records
  Loop over Opportunities
    Get Records: Account where Id = {!loopItem.AccountId}
    Update Records: Account
  End Loop

CORRECT — one query, one DML, regardless of record count
  Get Records: Accounts where Id IN {!accountIdCollection}
  Loop over Opportunities
    Assignment: set fields on a record variable
    Assignment: add record variable to {!accountsToUpdate}
  End Loop
  Update Records: {!accountsToUpdate}
```

### Building Collections Correctly

The pattern for accumulating records inside a loop:

```
1. Create a record variable of the right type:        {!varAccount}
2. Create a record collection variable:               {!colAccountsToUpdate}
3. Inside the Loop:
     Assignment "Prepare record"
       {!varAccount.Id}              = {!loopItem.AccountId}
       {!varAccount.Priority__c}     = "High"
     Assignment "Add to collection"
       {!colAccountsToUpdate}  Add   {!varAccount}
4. After the Loop:
     Update Records ← {!colAccountsToUpdate}
```

Note the two separate Assignment elements. Combining them works, but separating "prepare"
from "add" makes the intent readable to the next admin.

### Governor Limits That Bite Flows

| Limit | Value per transaction | Flow implication |
|---|---|---|
| SOQL queries | 100 | Each Get Records is one query — outside loops |
| DML statements | 150 | Each Create/Update/Delete element is one |
| DML rows | 10,000 | Total records committed |
| CPU time | 10,000 ms synchronous | Complex Flows on large batches hit this |
| Flow element executions | 2,000 per interview | Loops over large collections consume these |
| Heap | 6 MB synchronous | Storing all fields on large collections |

When approaching these limits, options in order: move to before-save, reduce fields
retrieved, use Collection Filter instead of Loop + Decision, split into an asynchronous
path, or escalate to Apex batch processing.

### Asynchronous Paths

A record-triggered Flow can define a **Run Asynchronously** path that executes after the
transaction commits, in its own set of governor limits. Use for:

- Callouts to external systems (not permitted in the synchronous path after DML)
- Heavy processing that would blow the CPU limit
- Work that need not block the user's save

Scheduled paths run at a defined offset from a date field or from record creation — useful
for renewal reminders, SLA escalations, and follow-up tasks.

---

## Screen Flows

### Design Principles

- One decision per screen. A screen with fourteen fields is a form, not a guided process.
- Set **help text** on every input component; users cannot ask you at 11pm.
- Use **Display Text** components to explain *why* the information is needed.
- Validate at the component level (`Validate Input` formula) rather than letting the user
  reach the end and fail.
- Provide a confirmation screen before any destructive or irreversible action.
- Set the **Pause** and **Previous** button visibility deliberately; both default on and
  both can corrupt a wizard's state assumptions.

### Reactive Screen Components

Screen components can react to other components on the same screen without a page
refresh — a picklist selection that reveals dependent fields, a calculated total that
updates as quantities change. Configure via component visibility filters referencing other
components' values, and via formula resources evaluated reactively.

This removes most of the multi-screen sequences older screen flows required.

### Surfacing Screen Flows

| Location | Method |
|---|---|
| Lightning record page | Flow component, with `recordId` passed in |
| Quick Action | Object Manager → Buttons, Links, and Actions → New Action → Flow |
| App page / Home page | Flow component in Lightning App Builder |
| Utility bar | Flow component in App Manager |
| Experience Cloud | Flow component on a community page |
| List view button | URL button or a Lightning Web Component wrapper |

Pass the record into the flow by creating a variable named exactly `recordId`, type Text,
marked **Available for input**.

---

## Validation Rules

### Mechanics

Validation rules fire on save, **after** before-save Flows have run and **before** the
record is committed. A formula evaluating to TRUE **blocks** the save.

Write the error condition. This inversion is the most common source of rules that appear to
do nothing.

### Useful Patterns

Require a field only at a specific stage:

```
AND(
  ISPICKVAL(StageName, "Negotiation/Review"),
  ISBLANK(Decision_Maker__c)
)
```

Prevent backward stage movement:

```
AND(
  ISCHANGED(StageName),
  TEXT(PRIORVALUE(StageName)) = "Closed Won",
  TEXT(StageName) != "Closed Won"
)
```

Restrict editing after a record is locked, with an admin bypass:

```
AND(
  Is_Locked__c,
  NOT($Permission.Edit_Locked_Records),
  $Profile.Name != "System Administrator"
)
```

Prefer `$Permission.Custom_Permission` over `$Profile.Name`. Profile names change; custom
permissions are stable, assignable via permission set, and auditable.

Validate a related field:

```
AND(
  ISPICKVAL(Account.Type, "Customer - Direct"),
  ISBLANK(Account.VAT_Number__c)
)
```

Cross-object references in validation rules only traverse *upward* (child to parent). To
validate against children, use a roll-up summary field or a before-save Flow with a Custom
Error element.

### Bypass Architecture

Every org eventually needs to suspend validation for a data migration or an integration.
Build the hook in from the start:

1. Create a Custom Permission: `Bypass_Validation_Rules`
2. Append to every validation rule: `&& NOT($Permission.Bypass_Validation_Rules)`
3. Create a permission set granting it
4. Assign to the migration user for the window, then unassign

This is dramatically safer than deactivating rules one by one and forgetting to reactivate
three of them.

### Validation Rule vs Custom Error in Flow

| Use a validation rule when | Use a Flow Custom Error when |
|---|---|
| The logic fits a single formula | You need to query other records |
| Only same-record and parent fields are needed | You need child-record or unrelated-object data |
| You want it enforced in every context including bulk API | Complex multi-step conditional logic |
| Simplicity and visibility matter | The message must vary by condition |

---

## Approval Processes

Still the correct tool for structured human sign-off. Flow Orchestration handles more
complex multi-stage, multi-actor sequences.

Key configuration:

- **Entry criteria** — which records can enter
- **Record editability** — administrators only, or administrators and the approver
- **Approver assignment** — hierarchy field, queue, related user field, or manual
- **Initial submission actions** — typically lock the record and set a status field
- **Approval / rejection / recall actions** — field updates, email alerts, outbound messages
- **Final approval actions** — unlock or keep locked, set status

Practical notes:

- The record locks on submission by default; users will report "I can't edit this" —
  document it
- Delegated approvers must be configured on the User record
- Recall is only available to the submitter unless configured otherwise
- A Flow can submit for approval via the **Submit for Approval** action, which is how you
  combine automated qualification with human sign-off

---

## Legacy Automation Migration

### Inventory

Before migrating anything, build the picture:

```
Setup → Process Automation → Workflow Rules      (filter: Active)
Setup → Process Automation → Process Builder     (filter: Active)
Setup → Flows                                     (existing flows per object)
Object Manager → [object] → Triggers              (Apex already in play)
```

Record for each: object, trigger condition, actions, active status, last modified date,
and — critically — whether anyone can still explain why it exists.

### Consolidation Approach

For each object, produce a single table:

| Legacy item | Trigger | Actions | Same-record only? | Target |
|---|---|---|---|---|
| WF: Set Opp Tier | Amount changed | Field update | Yes | Before-save branch |
| WF: Notify Manager | Stage = Closed Won | Email alert | No | After-save branch |
| PB: Create Renewal | Stage = Closed Won | Create record | No | After-save branch |
| PB: Update Account | Stage = Closed Won | Update parent | No | After-save branch |

Three legacy items firing on the same condition collapse into one Decision outcome in the
after-save Flow. This is the value the 1:1 migration tool cannot deliver.

### Cutover Sequence

1. Build the consolidated Flows in a sandbox, **inactive**
2. Activate in sandbox, deactivate the legacy automations, run the full regression
3. Test bulk behaviour — load 200 records, confirm one interview and correct results
4. Deploy Flows to production, still inactive
5. In a low-traffic window: activate the Flows, deactivate the legacy automations
6. Monitor for one full business cycle before deleting the legacy items
7. Delete only after the monitoring period; deactivation is reversible, deletion is not

Never run both the legacy automation and its replacement active simultaneously. Duplicate
emails and double-counted field updates are the immediate symptom.

---

## Order of Execution

Understanding this resolves most "my automation ran but the value is wrong" problems. On
save, Salesforce executes approximately in this order:

```
 1. Load the original record (or initialise for insert)
 2. Overwrite with new field values from the request
 3. System validation: required fields, field formats, max length
 4. Apex before triggers
 5. Custom validation rules  ─┐
 6. Before-save record-triggered Flows   ← runs BEFORE validation rules in practice
 7. Duplicate rules
 8. Save to the database (not yet committed)
 9. Apex after triggers
10. Assignment rules
11. Auto-response rules
12. Workflow rules (legacy)
13. Escalation rules
14. After-save record-triggered Flows
15. Roll-up summary recalculation on the parent
16. Criteria-based sharing recalculation
17. Commit — DML is now permanent
18. Post-commit: async paths, platform events, email sends
```

Practical consequences:

- Before-save Flow values are visible to validation rules
- Roll-up summaries recalculate *after* after-save Flows, so a Flow cannot read a rollup
  value it just caused to change
- Sharing recalculation is last, which is why a newly-shared record may not be visible
  within the same transaction
- Anything that re-triggers a save (an after-save Flow updating its own record) re-enters
  the cycle from step 1

---

## Flow Testing and Debugging

### Debug Mode

Setup → Flows → open the flow → **Debug**. Options that matter:

- **Run the latest version** — otherwise you debug the active version, not your draft
- **Show details of what's executed** — the element-by-element trace with variable values
- **Roll back changes** on record-triggered debug — test without committing data

For record-triggered flows, debug lets you select an existing record and simulate the
trigger, including setting "prior values" to test `ISCHANGED()` logic.

### Flow Tests

Salesforce supports saving Flow Tests against record-triggered Flows — a stored set of
input record values and assertions. These persist with the Flow, run on demand, and are
the closest declarative equivalent to Apex test coverage.

Create at least one test per Decision outcome. Assertions can check field values on the
triggering record after execution.

### Common Failure Modes

| Symptom | Likely cause |
|---|---|
| Flow does not run at all | Not activated; entry criteria not met; "only when updated to meet criteria" set |
| Runs but takes no effect | Before-save flow with an unnecessary Update Records; assigned to a variable instead of `$Record` |
| Runs twice | Two active flows on the object; recursion from an after-save self-update |
| "Too many SOQL queries" | Get Records inside a Loop |
| "Too many DML statements" | Create/Update inside a Loop |
| Null pointer at runtime | Get Records returned nothing and the next element referenced a field on it |
| Works for one record, fails on import | Not bulkified; test with 200 records |
| Works for admin, fails for users | Flow runs in user context by default — check FLS and record access. "System Context Without Sharing" is available but should be a deliberate, documented choice |
| Wrong value after save | Order of execution — a later automation overwrote it |

### Flow Run Context

Set on the Flow's advanced properties:

- **User context** (default for screen and most record-triggered flows) — respects the
  running user's FLS, sharing, and object permissions
- **System context with sharing** — bypasses object and field permissions, respects sharing
- **System context without sharing** — bypasses everything

Default to user context. Escalate only when the requirement genuinely needs it, and note
the reason in the Flow description. A Flow running in system context is a sharing bypass
that will not show up in a permission audit.

---

## Flow Governance

In an org with more than a handful of flows, adopt these conventions:

- **Naming**: `Object_TriggerPoint_Purpose` — sortable, self-describing in the Flows list
- **Description**: what it does, why it exists, who requested it, and the date. The Flow
  list view shows descriptions; use them
- **Version comments**: every new version gets a note on what changed
- **Deactivate, do not delete** — keep the audit trail
- **One owner per object's automation** in a team setting, to avoid conflicting flows
- **Quarterly review**: list all active flows, confirm each still has a business purpose

A useful health query for an inherited org — count active flows per object. Any object with
more than two record-triggered flows in the same context is a consolidation candidate.

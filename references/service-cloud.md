# Service Cloud

---

## Case Management Foundations

### Case Object Anatomy

Standard fields that carry behaviour, not just data:

| Field | Behaviour |
|---|---|
| `Status` | Drives Case lifecycle. Values marked "Closed" in the Support Process stop entitlement milestone clocks and count toward closure metrics |
| `Priority` | Free picklist by default; only meaningful if escalation rules or assignment rules reference it |
| `Origin` | Set automatically by Email-to-Case, Web-to-Case, and channel integrations. Do not let agents edit it — it destroys channel reporting |
| `OwnerId` | User **or** Queue. This dual nature is why Case ownership behaves differently from other objects |
| `ParentId` | Case hierarchy. Enables parent-child rollup for mass incidents |
| `IsEscalated` | Set by escalation rules; reportable |
| `ClosedDate` | Stamped automatically when Status moves to a Closed value |
| `ContactId` / `AccountId` | Contact drives entitlement lookup and customer notification |
| `SuppliedEmail`, `SuppliedName` | Populated by Web-to-Case when no Contact matches |

### Support Processes

The Case equivalent of a Sales Process. Controls which `Status` values are available per
record type.

```
Support Process: Technical_Support
  New → In Progress → Awaiting Customer → Escalated → Resolved → Closed

Support Process: Billing_Enquiry
  New → In Progress → Resolved → Closed
```

Mark the correct values as "Closed" in **Setup → Case Status**. This is a separate,
global list from the Support Process — a status not flagged as Closed will never stamp
`ClosedDate` regardless of what it is called.

### Record Type Strategy

Common and defensible split:

```
Case Record Types
├── Technical_Support     → Technical support process, technical layout,
│                            Product/Component picklists visible
├── Billing               → Billing support process, finance layout,
│                            Invoice lookup visible
└── Complaint             → Complaint process, compliance layout,
                             Regulatory fields, restricted visibility
```

Assign via permission set. Keep the count low — every record type multiplies layout,
process, and picklist maintenance.

---

## Case Assignment and Routing

Three mechanisms, applied in order of increasing sophistication.

### 1. Assignment Rules

Setup → Case Assignment Rules. **One active rule** per org, containing multiple ordered
rule entries. First matching entry wins.

```
Rule: Standard_Case_Assignment
  Entry 1  Origin = "Email"     AND  Product__c = "Platform"
           → Queue: Platform_Support
  Entry 2  Origin = "Web"       AND  Priority   = "High"
           → Queue: Priority_Triage
  Entry 3  Account.Tier__c = "Enterprise"
           → Queue: Enterprise_Support
  Entry 4  (no criteria — catch-all)
           → Queue: General_Support
```

Always define a final catch-all entry. A Case matching nothing is assigned to the record
creator, which for an integration user means it becomes invisible.

Assignment rules run:
- Automatically for Email-to-Case, Web-to-Case, and On-Demand Email-to-Case
- For manual creation **only** if "Assign using active assignment rules" is checked on the
  layout. Add the checkbox to the layout and set its default to checked, or agents will
  create orphan Cases.
- From a Flow via the `Assignment Rule Header` on Create Records — set
  `useDefaultRule = true`

### 2. Queues

A Queue is an ownership container. Members pull work from it.

```
Setup → Queues → New
  Label:            Platform Support
  Queue Email:      platform-support@example.com
  Send Email to Members:  yes/no
  Supported Objects:      Case
  Members:          Public Group "Platform Support Team"
```

Add **public groups**, not individual users. Membership changes then happen in one place.

Queue behaviour worth knowing:
- Queue members see queue-owned records regardless of sharing, provided they have object
  Read
- Accepting a record transfers ownership to the accepting user
- A queue can own records across multiple objects
- Queue list views are the primary agent work surface in orgs not using Omni-Channel

### 3. Omni-Channel

Push-based routing. Work is assigned to available agents based on capacity, skill, and
priority — the agent does not pull from a queue.

Configuration sequence:

```
1. Enable Omni-Channel                Setup → Omni-Channel Settings
2. Service Channel                    one per object routed (Case, Lead, Chat, custom)
3. Routing Configuration              routing model, priority, capacity per item
4. Assign Routing Config to a Queue   the queue becomes an Omni queue
5. Presence Statuses                  Available - Cases, Available - Chat, Break, etc.
6. Presence Configuration             capacity, auto-accept, decline behaviour
7. Add Omni-Channel to the Utility Bar in the Lightning app
8. Permission sets                    grant presence statuses to agents
```

**Routing models:**

| Model | Behaviour |
|---|---|
| Least Active | Agent with the lowest current workload |
| Most Available | Agent with the greatest remaining capacity |
| External Routing | Third-party routing engine via API |

**Capacity model:** each agent has a capacity number (default 100 in the presence config);
each work item consumes a "capacity weight" or a percentage. A Case weighted 20 means an
agent at capacity 100 holds five concurrent Cases.

**Skills-Based Routing** matches required skills on the work item to skills on the agent.
Configure skills in Setup → Skills, assign to users, and set required skills either on the
routing configuration or dynamically via Apex/Flow. Adds real capability but also real
maintenance — do not enable it before queue-based Omni is working.

### Escalation Rules

Time-based automatic escalation. One active rule, multiple entries, each with escalation
actions at defined age thresholds.

```
Rule Entry:  Priority = "High" AND Status != "Closed"
  Business hours:  EMEA Support Hours
  Age based on:    Case creation time  (or last modification)

  Escalation Action 1  — after 2 hours
      Reassign to: Queue "Tier 2"
      Notify:      Case owner, Support Manager
      Set IsEscalated = true

  Escalation Action 2  — after 4 hours
      Reassign to: Queue "Tier 3"
      Notify:      Head of Support
```

Escalation runs against **Business Hours**, so a 4-hour SLA does not expire overnight if
business hours are configured. Configure Business Hours per region and set the Case's
`BusinessHoursId` — often via assignment rule or Flow — before relying on escalation.

Escalation rules cannot be triggered by Flow and do not fire on records created before the
rule was activated.

---

## Entitlements and Milestones

The formal SLA engine. Use when contractual response and resolution times must be tracked
and reported, not merely aspired to.

### Object Model

```
Account / Contact / Asset
      │
      ├── Entitlement            ← the customer's support contract
      │     Start Date, End Date, Type, Business Hours
      │     Cases Per Entitlement (optional limit)
      │
      └── Service Contract  (optional)
            └── Contract Line Item  (optional, per Asset)

Entitlement Process               ← the milestone template
      └── Milestone               ← First Response, Resolution Time
            Criteria, Time Trigger, Business Hours
            Success Actions / Warning Actions / Violation Actions
```

### Setup Sequence

```
1. Setup → Entitlement Settings → Enable Entitlement Management
2. Create Milestone Types              (First Response, Resolution)
3. Create Entitlement Process          object = Case, version 1
4. Add milestones to the process with:
     - Criteria       (e.g. Priority = High)
     - Time trigger   (minutes from process start, or a formula)
     - Business hours source
5. Define actions:
     Success   — stamp a field, post to Chatter
     Warning   — at 80% elapsed, notify the owner
     Violation — notify the manager, escalate, stamp a breach flag
6. Activate the process
7. Create Entitlements against Accounts/Contacts
8. Add the Milestones related list and the Entitlements lookup to the Case layout
9. Add the Milestone Tracker component to the Case Lightning page
```

### Practical Notes

- Milestones only start when the Case has an `EntitlementId`. Populate it via the
  "Entitlement Name" lookup, an assignment rule, or a Flow that looks up the Contact's or
  Account's active entitlement. A Case with no entitlement has no SLA clock — silently.
- **Completing** a milestone requires setting `MilestoneStatus`, usually via a Flow on the
  Case Comment/Email or a manual "Mark First Response" action. It does not complete itself.
- Entitlement Processes are **versioned**. Editing an active process requires creating a new
  version; existing Cases stay on the old version.
- `StopMilestoneTime` on the Case pauses all clocks — used for "Awaiting Customer" status.
  Drive it from a Flow on Status change.
- Milestone data is reportable via the Case Milestone object. Build the SLA compliance
  report before go-live, not after the first breach dispute.

---

## Channels

### Email-to-Case

Two implementations:

| | Standard Email-to-Case | On-Demand Email-to-Case |
|---|---|---|
| Agent required | Yes — installed behind the firewall | No |
| Attachment size | Larger | 25 MB total per email |
| Email volume | Higher | Subject to daily limits |
| Setup complexity | High | Low |

On-Demand is correct for almost every org now. Standard exists for compliance scenarios
requiring email never to leave the corporate network.

Configuration:

```
Setup → Email-to-Case → Enable
  Routing Address:  support@yourcompany.com
    → Salesforce generates a long forwarding address
    → Configure your mail server to forward support@ to it
    → Verify the address
  Default Case Owner:     Queue
  Save Email Headers:     yes (needed for threading diagnostics)
  Insert Thread ID:       in subject and body, or use Lightning Threading
```

**Lightning Threading** replaces the legacy `[ ref:_00D...:ref ]` token with a header-based
match, keeping subject lines clean. Enable it unless you have integrations parsing the
token.

Threading failure modes:
- Customer replies from a different address → new Case, no thread. Mitigate with
  Contact-based matching and a duplicate-detection Flow.
- Customer strips the thread token → new Case
- Auto-replies and out-of-office loops → configure an Auto-Response Rule carefully and
  filter on common auto-reply headers

**Bounce and loop protection:** set a Case Origin filter so Cases created from a
`no-reply@` sender do not trigger auto-response.

### Web-to-Case

```
Setup → Web-to-Case → Enable
  Default Case Origin:   Web
  Default Response Template
  Generate the HTML form → embed on the website
```

Hard limits: **5,000 Cases per day**. Beyond that, submissions are dropped or queued
depending on settings, and Salesforce notifies the default contact.

Web-to-Case has no spam protection natively. Add reCAPTCHA (supported in the generated
form) or, better, replace it with a Screen Flow on an Experience Cloud site or an API
integration, which give validation, duplicate checking, and authentication.

### Auto-Response Rules

Separate from assignment rules; one active rule per object with ordered entries. First
match wins.

```
Rule: Case_Auto_Response
  Entry 1  Origin = "Web"    → Template: Web_Acknowledgement,  From: support@
  Entry 2  Origin = "Email"  → Template: Email_Acknowledgement, From: support@
  Entry 3  (catch-all)       → Template: Generic_Acknowledgement
```

Set the From address to a verified **Org-Wide Email Address**, not the running user.

### Messaging and Digital Channels

Web Chat (Messaging for In-App and Web), WhatsApp, SMS, and Facebook Messenger route
through **Messaging Channels** into Omni-Channel. Each requires:

1. A Messaging Channel record with the provider configuration
2. A routing configuration and queue
3. Presence statuses covering the channel
4. Agents granted the channel in a permission set

Conversation records (`MessagingSession`) are separate from Cases. Decide deliberately
whether every conversation creates a Case — most orgs should not, and should instead let
the agent escalate a session to a Case when it needs tracking.

---

## Knowledge

### Setup

```
1. Setup → Knowledge Settings → Enable Lightning Knowledge
   (irreversible — enable in a sandbox first)
2. Create Record Types on the Knowledge object (article types)
3. Define Data Categories and Category Groups
4. Configure Data Category Visibility per role/permission set
5. Assign Knowledge User permission set licences
6. Add the Knowledge component to the Case Lightning page
7. Configure Article Actions per profile: Publish, Archive, Delete, Submit for Translation
```

Lightning Knowledge uses **record types** as article types. Enabling Lightning Knowledge is
one-way — plan the migration from Classic Knowledge carefully.

### Data Categories

Hierarchical taxonomy for filtering and access control.

```
Category Group: Products
  ├── Platform
  │     ├── Integration
  │     └── Reporting
  └── Mobile

Category Group: Regions
  ├── EMEA
  └── APAC
```

Limits: 5 category groups active at once (3 for Answers), 100 categories per group, 5
levels deep. Visibility can be set to All Categories, None, or Custom per role, permission
set, or profile — this is how internal-only articles stay internal.

### Article Lifecycle

```
Draft → (Approval Process, optional) → Published
                                          ├── Archived
                                          └── New Draft Version → Published
```

Publishing creates a new version; the previous version is retained. Archiving removes it
from search without deleting.

### Case Deflection and Attachment

- **Attach articles to Cases** so resolution knowledge is captured. The Article-Case
  association is reportable — "articles attached per Case" is the deflection proxy metric.
- **Insert article content into emails** from the Case feed.
- **Case deflection** on Experience Cloud and in Web-to-Case forms surfaces articles before
  submission. Reportable via the `CaseDeflection` object.
- **Knowledge-Centered Service (KCS)** — capture articles as a by-product of resolving
  Cases rather than as a separate documentation project. Configure the "Create Article from
  Case" action.

---

## Agent Productivity

### Lightning Service Console

An app with `Console Navigation` set, giving tabs, subtabs, split view, and utility bar.

```
Setup → App Manager → New Lightning App
  Navigation Style:   Console navigation
  Utility Items:      Omni-Channel, History, Notes, Macros, Open CTI Softphone
  Navigation Rules:   how records open (workspace tab vs subtab)
```

Key console configuration:

- **Split View** — list on the left, record on the right. Set per list view
- **Pinned regions** on the record page — highlights panel stays visible while scrolling
- **Keyboard shortcuts** — Setup → Keyboard Shortcuts; agents who use them are materially
  faster
- **Push Notifications** — configure which field changes flash on an open tab

### Macros

Automate repetitive agent actions in the Case feed.

```
Macro: Acknowledge_And_Set_In_Progress
  Instructions:
    1. Select Email Action
    2. Apply Email Template "Acknowledgement"
    3. Set Case Status = "In Progress"
    4. Submit action
```

**Irreversible macros** execute without agent confirmation and can be bulk-run against
multiple Cases from a list view. Powerful and correspondingly risky — restrict the
"Manage Macros" and "Run Macros on Multiple Records" permissions.

Macros only work on actions present on the page layout and feed. A macro referencing a
missing action fails silently.

### Quick Text

Reusable snippets inserted into emails, chats, and text fields with a keyboard shortcut.

```
Quick Text: Greeting_Standard
  Category:  Greeting
  Channel:   Email, Messaging
  Message:   Hi {!Contact.FirstName}, thanks for getting in touch...
```

Merge fields work. Organise by folder and share via folder permissions. Enable "Quick Text
Settings → Let users insert quick text with a keyboard shortcut" or adoption will be near
zero.

### Email Templates for Service

Lightning Email Templates, stored in folders, support merge fields and Enhanced Letterhead.
For Case emails specifically:

- Include `{!Case.CaseNumber}` and thread context
- Set the Org-Wide Address as the From, not the agent
- Use the "Related To" merge context so `{!Case.…}` resolves
- Test rendering in the actual Email action, not the template preview

### CTI and Telephony

**Open CTI** is the framework; the implementation is a partner adapter (Amazon Connect,
Genesys, Five9, Talkdesk, etc.).

**Service Cloud Voice** is Salesforce's native telephony, available with Amazon Connect,
partner telephony, or bring-your-own. Provides in-console call control, real-time
transcription, and Einstein call insights. Separate licence.

Admin responsibilities either way: Call Center definition, softphone layout, permission
assignment, and mapping call outcomes to Case fields.

---

## Case Automation Patterns

### Status-Driven Milestone Pausing

```
Flow: Case_BeforeSave_MilestonePause
Trigger: Record updated
Entry:   ISCHANGED({!$Record.Status})

Decision "Awaiting Customer?"
  Status = "Awaiting Customer"
    → {!$Record.StopMilestoneTime} = {!$Flow.CurrentDateTime}
  Status changed FROM "Awaiting Customer"
    → {!$Record.StopMilestoneTime} = null
```

Before-save, so no extra DML. This is the single most requested Service Cloud automation
and is not available declaratively without it.

### First Response Stamping

```
Flow: EmailMessage_AfterSave_StampFirstResponse
Object:  EmailMessage
Entry:   Incoming = false  AND  ParentId != null

Get Records: Case where Id = {!$Record.ParentId}
Decision: First_Response_Date__c is null?
  → Update Case: First_Response_Date__c = {!$Flow.CurrentDateTime}
  → Complete the First Response milestone
```

### Auto-Close Stale Cases

```
Scheduled Flow: Case_Scheduled_AutoClose
Schedule:  Daily 02:00
Object:    Case
Filter:    Status = "Awaiting Customer"
           AND LastModifiedDate < LAST_N_DAYS:7

Loop → Assignment (Status = "Closed", Closed_Reason__c = "No customer response")
     → add to collection
Update Records (outside loop)
```

Send a warning email at day 5 with a separate scheduled path before closing at day 7.

### Parent-Child Mass Incident

```
Parent Case: "Platform outage 2026-08-23"
  └── Child Cases (customer-reported instances)

Flow on Parent Case:  Status → "Resolved"
  Get Records: Cases where ParentId = {!$Record.Id} AND Status != "Closed"
  Loop → set Status = "Resolved", set Resolution__c from parent
  Update Records (outside loop)
  → Send resolution email to each Contact
```

Bulkify carefully — a major incident can produce thousands of child Cases, which will
exceed Flow limits. Above roughly 2,000 children, escalate to Apex batch.

---

## Service Metrics

Build these before go-live; retrofitting the fields needed is painful.

| Metric | Requires |
|---|---|
| First Response Time | `First_Response_Date__c` stamped by automation |
| Average Resolution Time | `ClosedDate` − `CreatedDate`, business-hours adjusted |
| SLA Compliance | Case Milestone object, `IsCompleted` and `IsViolated` |
| First Contact Resolution | A flag set when closed without reassignment or reopening |
| Reopen Rate | Count of Status transitions back from Closed — needs field history or a counter |
| Case Volume by Origin | `Origin`, protected from agent editing |
| Backlog | Open Cases by age bucket |
| Agent Utilisation | Omni-Channel `AgentWork` object |
| Deflection | `CaseDeflection` object, Knowledge article views |
| CSAT | Survey responses linked to Case |

Business-hours-adjusted duration is not native. Either store a formula using a Business
Hours helper, or compute it in a scheduled Flow into a stored field. A raw
`ClosedDate − CreatedDate` on a Case opened Friday evening reports 60 hours for what was
two hours of work — and destroys the credibility of the whole dashboard.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Cases created manually are not assigned | "Assign using active assignment rules" checkbox missing from the layout or unchecked by default |
| Email-to-Case creates duplicates instead of threading | Customer replied from a different address, or the thread token was stripped. Check Lightning Threading is enabled |
| Milestone never starts | Case has no `EntitlementId` |
| Milestone never completes | No automation sets `MilestoneStatus`; completion is not automatic |
| SLA breached overnight despite business hours | `BusinessHoursId` not populated on the Case, so the default org business hours applied |
| Escalation rule does not fire | Rule inactive, Case created before activation, or the Case matched an earlier rule entry |
| Omni-Channel does not route to an agent | Agent not on a presence status mapped to that service channel, at capacity, or the queue has no routing configuration |
| Agent cannot see the Omni widget | Utility item missing from the Lightning app, or the Omni permission not granted |
| Auto-response fires on auto-replies, causing a loop | No filter on the auto-response rule; add Origin and sender-based exclusions |
| Knowledge article not visible to an agent | Data Category Visibility, or the article is Draft rather than Published |
| Web-to-Case submissions stop | 5,000/day limit reached |
| Macro does nothing | The action it references is not on the layout or feed |
| Queue members cannot see queue Cases | Missing object-level Read, or they are not actually in the queue's group |
| `ClosedDate` blank on a closed Case | The Status value is not flagged as "Closed" in Case Status settings |
| Case Origin values inconsistent | Agents can edit the field — make it read-only via FLS |

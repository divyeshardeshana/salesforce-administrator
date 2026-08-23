# Sales Cloud

---

## Lead Management

### Lead Object Purpose

A Lead is an unqualified, unconverted person-plus-company in a single record. It exists so
that unvetted data never pollutes the Account/Contact/Opportunity model.

Orgs that skip Leads and create Accounts directly end up with thousands of junk Accounts
and no way to distinguish a prospect from a customer. Orgs that never convert Leads end up
with a parallel universe of data invisible to Opportunity reporting. Both failure modes are
common; the fix is a defined qualification threshold and a conversion discipline.

### Lead Assignment Rules

One active rule per org, ordered entries, first match wins — same mechanics as Case
assignment.

```
Rule: Lead_Routing
  Entry 1  Country IN (AE, SA, QA, KW, BH, OM)  → Queue: GCC_Sales
  Entry 2  Annual Revenue > 10,000,000          → Queue: Enterprise_SDR
  Entry 3  Lead Source = "Webinar"              → User: Marketing_Qualifier
  Entry 4  (catch-all)                          → Queue: Inbound_Triage
```

Fires automatically on Web-to-Lead and lead import. For manual creation and Flow-created
Leads, the "Assign using active assignment rules" checkbox must be on the layout or the
Assignment Rule Header set in the Flow.

Always end with a catch-all. Unassigned Leads default to the creating user.

### Lead Conversion

Converting produces up to three records and deactivates the Lead:

```
Lead
 ├── Account    (new, or matched to existing)
 ├── Contact    (new, or matched to existing)
 └── Opportunity (optional — "Do not create an Opportunity" is available)
```

Field mapping: **Setup → Object Manager → Lead → Fields → Map Lead Fields**. Every custom
Lead field that must survive conversion needs an explicit mapping to a field of the same
data type on Account, Contact, or Opportunity. Unmapped custom field data is lost at
conversion — silently, permanently, and this is the single most common Sales Cloud data-loss
complaint.

Audit this before any go-live: list every custom Lead field, confirm each is either mapped
or deliberately not.

Other conversion behaviour:

- The Lead becomes read-only with `IsConverted = true`; it does not appear in standard list
  views
- `ConvertedAccountId`, `ConvertedContactId`, `ConvertedOpportunityId` are stamped — use
  these for closed-loop source reporting
- Validation rules on Account, Contact, and Opportunity **do** fire during conversion. A
  required field with no Lead equivalent blocks conversion for everyone
- Automation on the target objects fires. A Flow creating a welcome Task on Contact insert
  will fire on every conversion
- Duplicate rules apply if configured for the conversion context

### Lead Status and Qualification

```
Lead Status (with a Lead Process per record type)
  New → Working → Nurturing → Qualified (converted)
                            → Unqualified (with a required Disqualification_Reason__c)
```

Mark exactly one status as "Converted" in the status picklist settings. Require a
disqualification reason on the unqualified path — without it, you cannot report on why
pipeline is not forming.

### Web-to-Lead

```
Setup → Web-to-Lead → Enable
  Default Lead Creator
  Default Response Template
  Generate HTML → embed
```

Limit: **500 Leads per day**. Exceeded submissions go to a pending queue and the default
contact is notified; beyond the pending capacity they are discarded.

No native spam protection beyond optional reCAPTCHA. For any real volume, replace with a
Screen Flow on Experience Cloud or a proper API integration — you gain validation, duplicate
matching, and enrichment.

---

## Opportunity Management

### Sales Process and Stages

```
Setup → Sales Processes → New
  Available stages, ordered:
    Qualification → Needs Analysis → Proposal → Negotiation
                  → Closed Won / Closed Lost
```

Each stage carries:

| Attribute | Meaning |
|---|---|
| **Type** | Open, Closed/Won, or Closed/Lost. Drives `IsClosed` and `IsWon` |
| **Probability** | Default forecast weighting; overridable per record |
| **Forecast Category** | Pipeline, Best Case, Commit, Omitted, Closed |

Stage-to-Forecast-Category mapping is set on the stage, not the forecast. Getting this wrong
is why forecast numbers do not match pipeline reports.

Design guidance:

- Six to eight stages maximum. More produces stage-stuffing and unreliable data
- Every stage must have an **exit criterion** a manager can verify. "Proposal" means a
  document was sent, not that someone feels good
- Require a `Closed_Lost_Reason__c` on the lost path. Without it, win/loss analysis is
  impossible
- Enforce stage progression with a validation rule if backwards movement is a problem

### Required Fields by Stage

Page-layout "required" applies at all stages, which is too blunt. Use validation rules:

```
AND(
  ISPICKVAL(StageName, "Proposal"),
  OR(ISBLANK(Decision_Maker__c), ISBLANK(Budget_Confirmed__c))
)
```

Or **Path** with Key Fields, which prompts without blocking — better for adoption, weaker
for data integrity. Use Path for guidance and validation rules for the two or three fields
that genuinely must be present.

### Opportunity Path

```
Setup → Path Settings → New Path
  Object:  Opportunity
  Record Type / Picklist: StageName
  Per stage:
    Key Fields:      up to 5, editable inline
    Guidance for Success:  rich text — what "good" looks like at this stage
```

The single cheapest adoption win in Sales Cloud. Populate the guidance text with the actual
sales methodology rather than leaving it blank.

### Opportunity Teams

```
Setup → Opportunity Team Settings → Enable
  Enable Team Selling
  Default Opportunity Team per user (auto-added on record creation)
```

Each member has a Team Role and an access level (Read Only / Read Write). Splits require
enabling **Opportunity Splits**:

| Split type | Behaviour |
|---|---|
| Revenue split | Must total 100% of Amount |
| Overlay split | Any total, for non-quota-carrying contributors |
| Custom currency/numeric splits | Configurable on custom fields |

Enabling Splits triggers a background job that touches every Opportunity. Do it in a
sandbox first and schedule it in production for a quiet window.

### Contact Roles

`OpportunityContactRole` links Contacts to Opportunities with a role and a primary flag.
Chronically underused; it is the only native way to answer "who was involved in this deal."

Enforce it with a validation rule at the Proposal stage using a roll-up or a Flow-maintained
counter field — Contact Roles cannot be referenced directly in a validation rule formula.

---

## Products, Price Books, and Quotes

### Object Model

```
Product2                  ← the catalogue item (name, code, family, active)
   │
   └── PricebookEntry     ← this product's price in a specific price book
          │                 (Product2Id + Pricebook2Id + CurrencyIsoCode = unique)
          │
Pricebook2 ────────────────┘
   Standard Price Book     ← auto-created; every product needs an entry here first
   Custom Price Books      ← per region, segment, partner, or contract

Opportunity
   └── OpportunityLineItem ← references a PricebookEntry
          Quantity, UnitPrice, TotalPrice, Discount
```

Rules that trip people up:

- A Product **must** have a Standard Price Book entry before it can be added to any custom
  price book
- An Opportunity is locked to **one** price book. Changing it clears existing line items
- `Amount` on the Opportunity becomes read-only and rolls up from line items once products
  are added
- Deactivating a Product does not remove it from existing Opportunities

### Product Schedules

Revenue and quantity schedules split a line item across time periods — the native answer to
subscription and instalment revenue.

```
Setup → Product Schedules Settings → Enable Revenue and Quantity Schedules
Product2 → set "Can Use Revenue Schedule"
OpportunityLineItem → Establish Schedule → Divide evenly / Repeat amount
```

Adequate for straightforward instalments. For genuine subscription management — renewals,
amendments, co-terming, proration — this is not sufficient. Salesforce CPQ or Revenue Cloud
is the honest recommendation, with its licence cost stated up front.

### Quotes

```
Setup → Quote Settings → Enable
Opportunity → Quotes related list → New Quote
  Line items sync from the Opportunity
  Generate a Quote PDF from a Quote Template
  Mark one Quote as "Synced" → changes flow back to the Opportunity
```

Quote Templates are configured in Setup → Quote Templates with a drag-and-drop editor. They
are workable for simple documents and quickly become limiting — no conditional sections, no
complex layout. Beyond basic needs, use a document-generation tool or Salesforce CPQ.

Only one Quote can be synced at a time. Syncing overwrites Opportunity line items.

### Standard Quotes vs CPQ

State the boundary honestly when advising:

| Requirement | Standard Quotes | Needs CPQ |
|---|---|---|
| Simple line items and a PDF | Yes | — |
| Tiered or volume pricing | No | Yes |
| Product bundles and configuration rules | No | Yes |
| Approval-driven discounting matrix | Partially | Yes |
| Subscriptions, renewals, amendments | No | Yes |
| Contracted pricing per customer | Price books only | Yes |

CPQ is a managed package with its own data model, its own learning curve, and a separate
licence. Recommending it casually is a mistake; so is contorting standard Quotes for two
years to avoid it.

---

## Forecasting

### Collaborative Forecasts

```
Setup → Forecasts Settings → Enable
  Forecast Type:      Opportunity Revenue / Quantity / Product Family /
                      Opportunity Splits / Custom Opportunity Currency Field
  Forecast Hierarchy: derived from the Role Hierarchy
  Period:             monthly or quarterly, based on the Fiscal Year setting
  Forecast Categories to display
```

Key mechanics:

- The **forecast hierarchy** is the role hierarchy with forecast managers designated. A role
  without an assigned forecast manager breaks the rollup above it
- **Forecast Categories** derive from the Opportunity Stage mapping. Adjustments are made at
  the manager level and are stored separately from the underlying Opportunities
- **Quotas** are loaded per user per period via the Data Loader against the
  `ForecastingQuota` object; there is no bulk UI
- Up to four forecast types can be active simultaneously

Common failures:

- Numbers do not match the pipeline report → stage-to-category mapping, or the report
  includes categories excluded from the forecast
- A rep's numbers do not roll up → no forecast manager on the intervening role
- Fiscal periods wrong → Setup → Fiscal Year is misconfigured; custom fiscal years cannot be
  reverted to standard

### Custom Fiscal Years

Enabling custom fiscal years (4-4-5, 13-period, etc.) is **irreversible**. It changes report
period grouping, forecast periods, and quota periods. Confirm the requirement in writing
before enabling.

---

## Territory Management

**Enterprise Territory Management** provides an alternative record-access model based on
territory assignment rather than ownership and role hierarchy.

```
Territory Model  (only one Active at a time; others are Planning or Archived)
  └── Territory Type          (Named Account, Geographic, Named Vertical)
       └── Territory          hierarchical
            ├── Assignment Rules   (criteria-based Account assignment)
            ├── Assigned Users     with an access level
            └── Manually assigned Accounts
```

Setup sequence:

```
1. Setup → Territory Settings → Enable Enterprise Territory Management
2. Define Territory Types
3. Create a Territory Model in Planning state
4. Build the territory hierarchy
5. Create assignment rules and run them against the Planning model
6. Preview the results — which Accounts land where
7. Activate the model
```

When to use it: sales coverage is defined by geography, industry, or named-account lists
rather than by who owns the record; and reassignment happens periodically at scale.

When not to: a straightforward owner-plus-role-hierarchy model already works. Territory
Management adds a whole parallel sharing mechanism and a planning cycle. It is not a
lightweight feature.

Interaction with sharing: territory-based sharing is additive to the existing model. Users
assigned to a territory receive access to its Accounts and, depending on settings, the
related Opportunities and Cases.

---

## Campaigns and Attribution

```
Campaign
  ├── CampaignMember     (Lead or Contact, with a Status)
  ├── Parent/Child hierarchy
  └── Campaign Influence  (Opportunity attribution)
```

Configuration worth doing:

- **Member Status values** per campaign type — "Sent / Opened / Clicked" for email, "Invited
  / Registered / Attended" for events. Set a default and a "Responded" flag; the Responded
  flag drives `NumberOfResponses`
- **Campaign Hierarchy** — parent campaigns roll up member and cost statistics from children.
  Set `ParentId` and the rollup fields populate automatically
- **Campaign Influence** — Setup → Campaign Influence Settings. Choose a model:
  - *Primary Campaign Source* — single-touch, the campaign on the Opportunity
  - *Even distribution* — all campaigns touching the Contact Roles share credit equally
  - Custom models via Customizable Campaign Influence

Campaign Influence only works if **Opportunity Contact Roles are populated**. This is the
dependency chain that makes marketing attribution fail in most orgs: no Contact Roles → no
influence records → no attribution → marketing cannot prove ROI.

The `Primary Campaign Source` field on Opportunity is set at Lead conversion from the Lead's
campaign. Verify it is on the layout and being populated.

---

## Activities

Tasks and Events share the `Activity` reporting entity but are separate objects with
separate behaviour.

| | Task | Event |
|---|---|---|
| Time | Due date only | Start and end datetime |
| Calendar | No | Yes |
| Recurrence | Yes | Yes |
| `WhoId` | Contact or Lead | Contact or Lead |
| `WhatId` | Account, Opportunity, Case, custom | Same |

**Shared Activities** (Setup → Activity Settings → "Allow Users to Relate Multiple Contacts
to Tasks and Events") lets one activity relate to up to 50 Contacts. Enabling it is
**irreversible** and changes the reporting model. Enable it — the alternative is duplicate
activity records — but do it deliberately and in a sandbox first.

**Activity capture options:**

| Option | Cost | Data storage |
|---|---|---|
| Manual logging | Free | Stored in Salesforce |
| Einstein Activity Capture | Included at some tiers | Emails and events stored **outside** Salesforce, surfaced in the UI |
| Outlook/Gmail Integration | Free | Manual "log to Salesforce" |
| Third-party (Groove, Salesloft, etc.) | Paid | Usually stored in Salesforce |

The critical caveat with **Einstein Activity Capture**: captured emails and events are held
in a separate store, are **not reportable in standard reports**, and are not accessible via
SOQL. Orgs adopt it for the automation and then discover their activity reporting has
vanished. If activity metrics matter, either supplement EAC or choose a tool that writes
real records.

---

## Sales Automation Patterns

### Stage-Based Field Defaults

```
Flow: Opportunity_BeforeSave_StageDefaults
Entry: ISCHANGED({!$Record.StageName})

Decision
  Stage = "Closed Won"
    → {!$Record.Actual_Close_Date__c} = TODAY()
    → {!$Record.Win_Loss__c} = "Won"
  Stage = "Closed Lost"
    → {!$Record.Win_Loss__c} = "Lost"
  Stage = "Proposal"
    → {!$Record.Proposal_Sent_Date__c} = TODAY()
```

### Renewal Opportunity on Close

```
Flow: Opportunity_AfterSave_CreateRenewal
Entry: ISCHANGED({!$Record.StageName})
       AND ISPICKVAL({!$Record.StageName}, "Closed Won")
       AND {!$Record.Type} = "New Business"

Create Records: Opportunity
  Name        = {!$Record.Name} + " - Renewal"
  AccountId   = {!$Record.AccountId}
  Type        = "Renewal"
  CloseDate   = {!$Record.CloseDate} + 365
  StageName   = "Qualification"
  Amount      = {!$Record.Amount}
  Original_Opportunity__c = {!$Record.Id}
```

### Discount Approval Routing

```
Before-save Flow computes:
  Discount_Percent__c = (List_Price__c - Amount) / List_Price__c * 100

After-save Flow:
  Decision on Discount_Percent__c
    > 30  → Submit for Approval, process "Executive_Discount_Approval"
    > 15  → Submit for Approval, process "Manager_Discount_Approval"
    else  → no action
```

Compute in before-save, act in after-save. Combining both into one after-save Flow costs an
extra DML per record.

### Stale Opportunity Alert

```
Scheduled Flow: Opportunity_Scheduled_StaleAlert
Daily 07:00
Filter: IsClosed = false
        AND LastModifiedDate < LAST_N_DAYS:14
Loop → build a per-owner digest
Send Custom Notification to the owner
Create a Task for the owner
```

Digest per owner, not one notification per Opportunity. A rep with forty stale
Opportunities receiving forty notifications will disable notifications.

---

## Sales Metrics

| Metric | Requires |
|---|---|
| Pipeline coverage | Open pipeline ÷ quota for the period |
| Win rate | Closed Won ÷ (Closed Won + Closed Lost), by segment and by stage entered |
| Average deal size | Amount on Closed Won |
| Sales cycle length | `CloseDate` − `CreatedDate` on Closed Won |
| Stage conversion | Requires stage-entry timestamps — stamp them with a Flow, they are not native |
| Stage duration | Same; `LastStageChangeDate` gives current stage only |
| Slipped deals | Close date pushed — needs field history tracking on `CloseDate` |
| Lead conversion rate | `IsConverted` over Leads created in the period |
| Lead response time | First activity datetime − Lead `CreatedDate` |
| Forecast accuracy | Forecast snapshot vs actual — requires Reporting Snapshots |
| Campaign ROI | Campaign cost vs influenced Opportunity revenue |

Two of these need groundwork most orgs skip:

**Stage-entry timestamps.** Create a datetime field per stage, or a child
`Opportunity_Stage_History__c` object written by a Flow. Without it, "how long do deals sit
in Proposal" is unanswerable.

**Historical pipeline.** Enable Historical Trending on Opportunity (up to three months,
five fields) or build a weekly Reporting Snapshot. Otherwise "what did the pipeline look
like at the start of the quarter" has no answer.

Build both before go-live. Retrofitting historical data is impossible.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Custom Lead field data lost on conversion | No entry in Map Lead Fields for that field |
| Conversion blocked with a validation error | A validation rule on Account, Contact, or Opportunity fires during conversion |
| Cannot add a product to an Opportunity | Product has no Standard Price Book entry, or is inactive, or the price book is inactive |
| Opportunity Amount is read-only | Line items exist; Amount now rolls up from them |
| Changing the price book cleared the line items | Expected behaviour — an Opportunity binds to one price book |
| Forecast does not match the pipeline report | Stage-to-Forecast-Category mapping, or excluded categories |
| A rep's forecast does not roll up | No forecast manager assigned on an intervening role |
| Quota upload fails | Quotas load via Data Loader against `ForecastingQuota`; no bulk UI exists |
| Territory assignment rules did not run | Model is in Planning state, or rules were not executed against the model |
| Campaign influence is empty | No Opportunity Contact Roles populated |
| Campaign hierarchy totals are wrong | `ParentId` not set, or rollup fields not on the layout |
| Activities missing from reports | Einstein Activity Capture stores data outside Salesforce and outside standard reporting |
| Duplicate activities per Contact | Shared Activities not enabled |
| Lead assignment rule skipped on manual creation | "Assign using active assignment rules" checkbox absent from the layout |
| Web-to-Lead submissions stop | 500/day limit reached |
| Cannot report on time-in-stage | Not native — stage-entry timestamps must be stamped by automation |

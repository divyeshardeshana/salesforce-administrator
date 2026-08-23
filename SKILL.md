---
name: salesforce-administrator
description: Designs and maintains Salesforce orgs declaratively — data model and object relationships, permission sets and sharing architecture, record-triggered Flows and validation rules, Sales Cloud lead-to-quote configuration, Service Cloud case routing and SLAs, reports and dashboards, sandbox and release management, data quality, and org health auditing. Use when configuring Salesforce without code, migrating legacy automation to Flow, designing a security model, troubleshooting record visibility, setting up forecasting or entitlements, building analytics, or planning a seasonal release.
license: MIT
metadata:
  author: https://github.com/savvydatacloudconsulting
  version: "1.1.0"
  domain: platform
  triggers: Salesforce, Salesforce admin, Salesforce administrator, Flow Builder, record-triggered flow, screen flow, permission set, permission set group, profile, sharing rule, organization-wide defaults, OWD, role hierarchy, record type, page layout, dynamic forms, validation rule, custom report type, dashboard, sandbox, change set, deployment, Data Loader, duplicate rule, Optimizer, Health Check, Setup Audit Trail, seasonal release, Sales Cloud, Service Cloud, Experience Cloud, lead conversion, lead assignment rule, opportunity stage, sales process, price book, quote, forecasting, territory management, campaign influence, case assignment rule, escalation rule, queue, Omni-Channel, entitlement, milestone, SLA, Knowledge, Email-to-Case, Web-to-Case, macro, quick text, service console
  role: expert
  scope: configuration
  output-format: configuration
  related-skills: salesforce-developer, data-analyst, security-auditor, business-analyst
---

# Salesforce Administrator

Declarative-first Salesforce configuration. Solve with clicks before code; when code is
genuinely required, hand off to `salesforce-developer` with a written justification.

## Core Workflow

1. **Clarify the requirement** — Who is the user, what record do they touch, what outcome
   changes? Reject vague asks ("make it better"); restate as an object, a field, a trigger
   point, and an acceptance test.
2. **Check the org first** — Before building anything, inspect what already exists. Search
   for a field that does the job, a Flow already firing on that object, a report type that
   covers the query. Most "new build" requests are duplicates.
3. **Choose the lowest-power tool** — Follow the automation decision matrix below. A
   formula field beats a Flow; a Flow beats Apex. Every step up costs maintenance forever.
4. **Design the security model before the fields** — Object access, then field-level
   security, then record access. Retrofitting sharing onto a live org is the single most
   expensive rework in Salesforce administration.
5. **Build in a sandbox** — Never configure directly in production beyond reports, list
   views, and user administration. Full/Partial Copy for anything data-dependent.
6. **Document the change** — Every field gets a description. Every Flow gets a description
   and a naming convention. Undocumented config is technical debt the next admin inherits.
7. **Deploy and verify** — Change set or DX. Verify in production against the acceptance
   test from step 1, not against "it looks right."

## Reference Guide

Load detailed guidance based on context:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Data Model & Security | `references/data-model-security.md` | Objects, fields, relationships, record types, profiles, permission sets, OWD, sharing rules, record visibility troubleshooting |
| Flow & Declarative Automation | `references/flow-automation.md` | Record-triggered flows, screen flows, scheduled flows, validation rules, approval processes, order of execution, legacy automation migration |
| Sales Cloud | `references/sales-cloud.md` | Leads and conversion, opportunity stages and paths, products and price books, quotes, forecasting, territories, campaigns, activities |
| Service Cloud | `references/service-cloud.md` | Cases, assignment and escalation rules, queues, Omni-Channel, entitlements and milestones, Knowledge, Email-to-Case, console productivity |
| Reporting & Analytics | `references/reporting-analytics.md` | Report types, summary/matrix/joined reports, formulas, bucketing, dashboards, subscriptions, large-data reporting |
| Org Management & Release | `references/org-management-release.md` | Sandboxes, change sets, deployment, data loading, duplicate management, Health Check, Optimizer, seasonal releases, user administration |

## Constraints

### MUST DO

- **Use permission sets and permission set groups for access; keep profiles minimal.**
  Profiles should carry only login hours, IP ranges, default record types, and page layout
  assignment. Everything else belongs in permission sets.
- **Set organization-wide defaults to the most restrictive level the business tolerates,
  then open access upward.** Sharing rules can only widen access, never narrow it.
- **Give every custom field a description and a help text.** No exceptions. An org with
  400 undescribed fields cannot be audited or safely refactored.
- **Use one record-triggered Flow per object per trigger point** (one before-save, one
  after-save), with internal Decision branching. Multiple flows on the same object and
  context have no guaranteed execution order.
- **Use before-save record-triggered Flows for same-record field updates.** They are
  roughly an order of magnitude faster than after-save and consume no additional DML.
- **Add a Fault path to every Flow element that performs DML or calls an action.** A Flow
  without fault handling surfaces an unhandled exception to the end user.
- **Enforce data quality at entry** — required fields, validation rules, duplicate rules,
  and picklists over free text. Cleaning data after the fact costs 10x preventing it.
- **Build in a sandbox and deploy.** Production configuration is only acceptable for
  reports, dashboards, list views, and user/permission administration.
- **Run Health Check and Optimizer before every major release** and record the score
  trend over time.
- **Name things consistently.** Adopt a convention (for example
  `Object_TriggerPoint_Purpose` for Flows) and apply it to every new component.

### MUST NOT DO

- **Do not grant "Modify All Data" or "View All Data" to solve a sharing problem.** It
  bypasses the entire sharing model, including field-level security in most contexts. If
  you are reaching for it, the record-access design is wrong.
- **Do not build new automation in Workflow Rules or Process Builder.** Creation is
  already disabled, and support ended 31 December 2025.
- **Do not use the Migrate to Flow tool as a migration strategy.** It converts 1:1 — 100
  Workflow Rules become 100 Flows. Consolidate and redesign instead.
- **Do not hard-code record IDs, user IDs, queue IDs, or record type IDs** in Flows,
  formulas, or validation rules. Use Custom Metadata, Custom Labels, or developer names.
- **Do not create a formula field where a rollup or a stored field would serve.** Formula
  fields are computed at query time and degrade report and list-view performance at scale.
- **Do not put a DML element (Create/Update/Delete Records) inside a Flow loop.** Build a
  collection inside the loop, commit once outside it.
- **Do not deactivate a validation rule to make a data load succeed** without a documented
  plan and window to reactivate it. This is how orgs accumulate invalid data silently.
- **Do not deploy to production on a Friday**, and never during a seasonal release
  maintenance window for the org's instance.
- **Do not delete fields.** Deprecate: rename with a `ZZ_` prefix, remove from all layouts,
  revoke field-level security, wait one full release cycle, then delete.

## Automation Decision Matrix

Work down this list. Stop at the first tool that satisfies the requirement.

| Requirement | Tool | Notes |
|---|---|---|
| Display a calculated value, no storage needed | Formula field | Free; recalculates on read. Watch compile size (5,000 chars) |
| Aggregate child records to a parent (master-detail) | Roll-Up Summary field | Stored; no automation cost. Max 40 per object (25 default) |
| Aggregate across lookup relationships | Record-triggered Flow or DLRS | No native rollup across lookups |
| Prevent a bad save | Validation rule | Runs before the record is committed; cheapest guardrail |
| Update fields on the record being saved | Before-save record-triggered Flow | Fastest option; no extra DML, no recursion risk |
| Update related records, send email, post to Chatter | After-save record-triggered Flow | Record ID is available; costs DML |
| Collect input from a user | Screen Flow | Surface on a Lightning page, action, or Experience Cloud site |
| Run on a schedule or in bulk over many records | Scheduled-triggered Flow | Respects Flow bulk limits; check batch size for large volumes |
| Route a record for human approval | Approval Process | Still the right tool; Flow Orchestration for multi-stage |
| Complex multi-object transaction, callouts, >2,000 records per run | Apex | Hand off to `salesforce-developer` with a written reason |

## Configuration Patterns

### Permission Architecture (Correct Pattern)

```
Profile: "Standard Sales User"                  ← minimal, one per user population
  ├─ Login hours / IP ranges
  ├─ Default record types
  └─ Page layout assignment

Permission Set Group: "Sales Rep — EMEA"        ← assigned to the user
  ├─ PS: Object_Opportunity_Edit                ← one concern per permission set
  ├─ PS: Object_Quote_Read
  ├─ PS: App_CPQ_Access
  ├─ PS: Field_Commission_Visibility
  └─ Muting PS: Mute_Opportunity_Delete         ← subtract without unpicking the group
```

Why this works: a new hire gets one permission set group. A leaver loses one assignment.
Auditing "who can delete Opportunities" means reading one permission set, not 40 profiles.

Anti-pattern to avoid:

```
Profile: "Sales Rep EMEA Senior Manager v3"     ← 40 near-identical profiles
Profile: "Sales Rep EMEA Senior Manager v3 - No Delete"
Profile: "Sales Rep EMEA Senior Manager v3 - No Delete (Temp)"
```

### Record Access Layers

Access is cumulative — a user gets the *most permissive* result across all layers. There
is no "deny" in the standard sharing model except Restriction Rules.

```
1. Organization-Wide Defaults   → baseline, start at Private
2. Role Hierarchy               → managers inherit subordinates' records
3. Sharing Rules                → owner-based or criteria-based, widens access
4. Manual Sharing / Teams       → per-record grants
5. Implicit Sharing             → parent-child on standard objects
6. Apex Managed Sharing         → code-controlled, developer territory
7. Restriction Rules            → the only mechanism that NARROWS access
```

Troubleshooting checklist when a user cannot see a record:

1. Object permission — does the profile/permission set grant Read on the object?
2. Field-level security — is the *field* hidden rather than the record?
3. OWD — is the object Private and the user is not the owner?
4. Role hierarchy — is the user above the owner in the hierarchy? "Grant Access Using
   Hierarchies" enabled on the object?
5. Sharing rule — does a criteria-based rule apply, and has the sharing recalculation
   completed?
6. Restriction rule — is one actively filtering this record out?
7. Use **Setup → Sharing Settings → Sharing Button** on the record, or the
   "Why can't this user see this record?" path in Setup → Users → Sharing Access.

### Naming Convention

Adopt and enforce. The exact scheme matters less than its consistency.

```
Flows              Object_TriggerPoint_Purpose
                   Opportunity_BeforeSave_SetCloseDateDefaults
                   Case_AfterSave_NotifyAccountOwner
                   Screen_Onboarding_NewCustomerIntake

Permission Sets    Category_Scope_AccessLevel
                   Object_Opportunity_Edit
                   Field_Commission_Visibility
                   App_CPQ_Access

Validation Rules   Object_Condition
                   Opportunity_CloseDateNotInPast
                   Account_VATNumberRequiredWhenBillable

Custom Fields      Business_Term__c  (no abbreviations, no v2/new/temp)
Report Folders     Dept — Purpose    (Sales — Pipeline Management)
```

### Validation Rule Pattern

Validation rules fire when the formula evaluates to **TRUE**. Write the *error* condition,
not the valid condition — the most common beginner mistake is inverting this.

```
AND(
  ISPICKVAL(StageName, "Closed Won"),
  OR(
    ISBLANK(Contract_Signed_Date__c),
    ISBLANK(PO_Number__c)
  ),
  NOT($Permission.Bypass_Opportunity_Validation)
)
```

Two things worth copying from this example:

- **Bypass hook.** `$Permission.Custom_Permission_API_Name` lets you grant a data-migration
  user or an integration user an exemption without deactivating the rule org-wide.
- **`ISPICKVAL` not `=`.** Picklist comparison with `=` against a string works until
  someone translates the org, then fails silently.

### Before-Save Flow (Correct Pattern)

```
Flow: Opportunity_BeforeSave_SetDefaults
Trigger: A record is created or updated
Object: Opportunity
Optimize for: Fast Field Updates          ← before-save
Entry Criteria (formula):
    ISNEW() || ISCHANGED({!$Record.Amount})

Elements:
  Decision "Has Amount?"
    → Assignment: {!$Record.Discount_Tier__c} = "Enterprise"
    → Assignment: {!$Record.Requires_Approval__c} = true
```

No Update Records element is needed — assigning to `$Record` in a before-save flow writes
the value as part of the same transaction. Adding an Update Records element here is a
common and expensive mistake.

### Bulkified Flow Loop (Correct Pattern)

```
CORRECT
  Get Records → collection of Contacts
  Loop over collection
    Assignment: set field on loop variable
    Assignment: add loop variable to updateCollection
  End Loop
  Update Records ← updateCollection        (ONE DML, outside the loop)

WRONG
  Loop over collection
    Update Records ← loop variable         (DML inside loop → governor limit)
  End Loop
```

## Quality Gates

Before marking any configuration change complete:

- [ ] Acceptance test from step 1 passes in the target org
- [ ] Field descriptions and help text populated
- [ ] Flow has a description, an API-name matching the convention, and fault paths
- [ ] Tested as a *non-admin* user (login-as, or a dedicated test user per persona)
- [ ] Tested with a bulk operation — 200 records minimum where automation is involved
- [ ] Permission changes reviewed for over-provisioning
- [ ] Change documented in the org's change log
- [ ] Rollback plan written for anything touching data or sharing

## When to Escalate to a Developer

Hand off to `salesforce-developer` when the requirement genuinely needs code. Say so
explicitly and give the reason:

- Complex multi-object transactional logic with rollback semantics
- Outbound callouts requiring authentication flows, retry, or bulk payload assembly
- Processing volumes above Flow's practical ceiling (roughly 2,000 records per transaction)
- Custom UI beyond Screen Flow and Dynamic Forms capability
- Apex Managed Sharing for record access that declarative sharing cannot express
- Anything requiring `Database.Stateful` batch processing or custom queueable chaining

Do not escalate to avoid learning Flow. Do not build a 60-element Flow to avoid escalating.

---

[Documentation](https://github.com/savvydatacloudconsulting/salesforce-administrator) ·
[Report an issue](https://github.com/savvydatacloudconsulting/salesforce-administrator/issues)

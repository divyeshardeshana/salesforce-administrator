# Data Model & Security

---

## Part 1 — Data Model Design

### Object Selection

Before creating a custom object, exhaust the standard objects. Salesforce ships reporting,
sharing, mobile layouts, API behaviour, and third-party integration compatibility with
standard objects that you would otherwise rebuild.

| Business concept | Use this standard object | Do not create |
|---|---|---|
| A company you sell to | Account | `Company__c`, `Client__c` |
| A person at that company | Contact | `Customer_Contact__c` |
| A deal in progress | Opportunity | `Deal__c`, `Pipeline_Item__c` |
| An unqualified inbound enquiry | Lead | `Enquiry__c` |
| A support request | Case | `Ticket__c`, `Issue__c` |
| A price-list entry | Product2 + PricebookEntry | `Item_Price__c` |
| A scheduled activity | Task / Event | `Follow_Up__c` |
| A signed agreement | Contract | `Agreement__c` (unless genuinely different) |
| A project with phases | Custom object — legitimately absent from core CRM | — |

Legitimate reasons for a custom object:

- The concept has its own lifecycle, ownership, and sharing requirements
- It has more than a handful of fields of its own
- Users need to report on it independently
- It participates in relationships with two or more other objects

Illegitimate reasons:

- "Account has too many fields already" — use record types and page layouts
- "We call it something different" — rename the tab and object label instead
- "The standard object has fields we don't use" — remove them from the layout

### Renaming Standard Objects

Setup → Rename Tabs and Labels. Changes the label everywhere in the UI while the API name
stays `Account`. This is almost always better than a custom object. Caveats:

- Does not change the API name, so reports built by formula and integrations are unaffected
- Some Salesforce Help text and error messages still use the original term
- Translation Workbench needed for multi-language orgs

### Relationship Types

| Type | Child ownership | Sharing | Roll-up summary | Reparenting | Required on child |
|---|---|---|---|---|---|
| **Lookup** | Independent owner | Independent | No (native) | Yes | Optional |
| **Master-Detail** | Inherited from parent | Inherited from parent | Yes | Optional setting | Yes — always |
| **Hierarchical** | Self-lookup on User only | — | No | Yes | Optional |
| **External Lookup** | Points to external object | — | No | Yes | Optional |
| **Many-to-Many** | Junction object with 2 M-D | Inherited from primary | Yes on both | — | Yes |

Decision guidance:

- **Master-Detail when** the child is meaningless without the parent (an Invoice Line
  without an Invoice), you need roll-up summaries, and inherited sharing is desirable.
- **Lookup when** the child can exist independently, needs its own owner, or needs its own
  sharing model. Also when you might later need to reparent freely.
- **Convert M-D to Lookup** is possible only if no roll-up summaries exist and all child
  records have a parent. **Lookup to M-D** requires every child record to be populated.

Hard limits worth knowing:

- Two master-detail relationships maximum per object
- Three levels of master-detail nesting maximum
- 40 roll-up summary fields per object (default 25, increasable by support)
- 40 lookup relationships per object

### Junction Objects (Many-to-Many)

```
Contact ──┐
          ├── Contact_Event_Registration__c ──┐
Event__c ─┘   (2 master-detail relationships)  │
                                                │
First M-D defined = "primary" ──────────────────┘
  • controls the junction record's look and feel
  • controls sharing inheritance
  • controls what happens on parent delete
```

Set the primary relationship deliberately. Deleting either master deletes the junction
record. If you need the junction to survive one parent's deletion, use a lookup for that
side and lose the roll-up on it.

### Field Type Selection

| Need | Field type | Watch out for |
|---|---|---|
| Fixed set of options | Picklist | Use a Global Value Set if reused across objects |
| Options that vary by another field | Dependent Picklist | Controlling field must be a picklist or checkbox |
| Money | Currency | Enable multi-currency *before* you have data, not after |
| Percentage | Percent | Stored as the displayed number; 50% is `50`, not `0.5` |
| A yes/no | Checkbox | Never nullable — defaults to false. Use a picklist if "unknown" matters |
| Free text under 255 chars | Text | Set a realistic length; 255 by default wastes nothing but signals nothing |
| Long free text | Text Area (Long) — 131,072 chars | Not filterable in reports; not indexed |
| Rich formatted text | Text Area (Rich) | Not searchable the same way; avoid for data you'll report on |
| Unique business key | Text with Unique + External ID | Enables upsert via API and Data Loader |
| Calculated, not stored | Formula | 5,000 char compile limit; performance cost at scale |
| Aggregated from children | Roll-Up Summary | Master-detail only |
| Encrypted sensitive data | Classic Encrypted Text or Shield Platform Encryption | Classic is not reportable or filterable |

### Formula Field Discipline

Formula fields are computed at query time. A report on 500,000 records with six formula
fields in the columns computes three million values before rendering.

Rules of thumb:

- Formulas referencing fields on the *same* record are cheap
- Formulas that traverse relationships (`Account.Owner.Manager.Email`) are expensive; each
  hop is a join
- Cross-object formulas cannot be indexed, so filtering a report by one forces a full scan
- Nested formulas referencing other formulas compound the cost — check the compile size
- If a value changes rarely and is read constantly, store it with a Flow instead

Compile size check: Setup → the field detail page shows "Compile size" against the 5,000
byte limit. If you are near it, the formula is too complex to maintain regardless.

### Record Types

Record types control:

- Which **picklist values** are available
- Which **page layout** a user sees (in combination with profile)
- Which **business process** applies (Sales Process, Support Process, Lead Process)

Record types do **not** control field-level security, record access, or required fields.

Create a record type when the same object serves genuinely different business processes —
a "New Business" Opportunity with a nine-stage sales process versus a "Renewal"
Opportunity with three stages. Do not create record types to hide fields; use Dynamic
Forms or page layouts.

```
Object: Opportunity
├── Record Type: New_Business
│     Sales Process: NB_Process (9 stages)
│     Layout: Opportunity_NewBusiness_Layout
│     Picklist Lead_Source__c: Web, Event, Partner, Outbound
│
└── Record Type: Renewal
      Sales Process: Renewal_Process (3 stages)
      Layout: Opportunity_Renewal_Layout
      Picklist Lead_Source__c: Existing Customer
```

Assign record types via permission set, not profile, wherever possible. Default record
type still comes from the profile, which is one of the few things profiles must retain.

### Page Layouts vs Dynamic Forms

Dynamic Forms supersede page layouts for field placement on Lightning record pages. They
allow field-level visibility rules without a record type or a separate layout.

| Capability | Page Layout | Dynamic Forms |
|---|---|---|
| Field placement | Yes | Yes, with component-level control |
| Conditional field visibility | No | Yes, filter-based |
| Required field enforcement | Yes | Yes |
| Related lists | Yes | Related lists still come from the layout |
| Buttons and actions | Yes | Still from the layout |
| Mobile | Yes | Supported |

Practical approach: keep a minimal page layout for related lists and actions, and drive
field placement through Dynamic Forms. This eliminates most of the layout proliferation
that makes older orgs unmaintainable.

Conditional visibility example:

```
Field: Cancellation_Reason__c
Set Component Visibility:
    Filter 1:  Record → Status  equals  "Cancelled"
```

### Compact Layouts

Determine the highlights panel on the record page and the fields shown in mobile cards and
lookup hover. Frequently forgotten; a compact layout showing "Created Date" instead of
"Amount" and "Stage" is a daily friction point for users.

---

## Part 2 — Security Architecture

Salesforce security operates in three independent layers. Diagnose in this order.

```
LAYER 1 — Object access:     Can the user touch this TYPE of record at all?
LAYER 2 — Field access:      Can the user see this FIELD on records they can access?
LAYER 3 — Record access:     WHICH records of that type can the user see?
```

A user who can see the Account object, has FLS on Annual Revenue, but is not in the
sharing scope of a specific Account, sees nothing. All three must align.

### Layer 1 — Object Permissions

CRUD plus two dangerous extras:

| Permission | Grants |
|---|---|
| Read | View records the user has record-level access to |
| Create | Create new records |
| Edit | Modify records the user has edit access to |
| Delete | Delete records the user has edit access to |
| **View All** | See *every* record of this object, bypassing sharing |
| **Modify All** | See, edit, delete, and transfer every record, bypassing sharing |

`View All Data` and `Modify All Data` (system permissions, not per-object) are broader
still and bypass sharing on every object.

Audit these regularly:

```
Setup → Permission Sets → [each set] → System Permissions
Setup → Profiles → [each profile] → Administrative Permissions
Look for: View All Data, Modify All Data, Manage Users, Customize Application,
          Author Apex, Manage Sharing, Data Export, Password Never Expires
```

In a healthy org, `Modify All Data` exists on the System Administrator profile and nowhere
else. Integration users are the most common place it leaks in — an integration that needs
to write to all Accounts should get a permission set with `View All`/`Modify All` on
*Account only*, not the org-wide system permission.

### Layer 2 — Field-Level Security

FLS is enforced in the UI, in reports, in list views, in search results, and — critically —
in the API for most contexts. It is set per field, per profile or permission set.

Two settings per field: **Visible** and **Read-Only**.

Common failures:

- A field is on the page layout but invisible via FLS → the user sees nothing and reports a
  "missing field". Check FLS before touching the layout.
- A field is required on the page layout but hidden by FLS → the record cannot be saved.
- Formula fields inherit nothing; set FLS on the formula field itself.
- FLS does **not** apply to Apex running in system mode, so a Flow calling Apex may expose
  a field the user cannot otherwise see.

Bulk FLS management is far faster from Setup → Object Manager → Fields → Set Field-Level
Security (all profiles for one field) than from the profile side.

### Layer 3 — Record Access

Access is additive. Every mechanism below *grants*; only Restriction Rules remove.

#### Organization-Wide Defaults (OWD)

The baseline. Set the most restrictive value the business can tolerate.

| OWD | Meaning |
|---|---|
| Private | Only the owner and users above them in the role hierarchy |
| Public Read Only | Everyone can view; only owner and hierarchy can edit |
| Public Read/Write | Everyone can view and edit |
| Public Read/Write/Transfer | Lead and Case only; adds ownership change |
| Controlled by Parent | Detail side of master-detail; inherits from the master |

Rule: **you can widen with sharing rules, you cannot narrow.** Starting Public and trying
to restrict later means rebuilding the model. Start Private.

Also on this screen: **Grant Access Using Hierarchies**. Enabled by default and not
editable for standard objects. On custom objects, disabling it means managers do *not*
automatically see subordinates' records — occasionally exactly what a compliance
requirement demands.

#### Role Hierarchy

Roles are about **record access rollup**, not reporting lines and not job titles. A user
in a role sees records owned by users in roles beneath them.

Design guidance:

- Keep it shallow. Depth costs sharing recalculation time on every ownership change.
- Model it on who needs to *see* whose data, not the HR org chart.
- Fewer than ten levels in almost all cases; performance degrades noticeably beyond that.
- Roles are also what territory-less sharing rules key off, so plan them before writing rules.

#### Sharing Rules

Two kinds:

**Owner-based** — records owned by members of a role, role-and-subordinates, or public
group are shared with another role or group.

**Criteria-based** — records matching a field condition are shared, regardless of owner.

```
Rule: Share EMEA Opportunities with EMEA Support
  Type:        Criteria-based
  Criteria:    Region__c equals "EMEA"
  Share with:  Public Group "EMEA Support Team"
  Access:      Read Only
```

Limits: 300 sharing rules per object (50 criteria-based). Criteria-based rules trigger
sharing recalculation on every matching record change — a criteria-based rule on a
high-volume object with a frequently-changing field is a known performance trap.

#### Public Groups

Containers for users, roles, roles-and-subordinates, territories, and other public groups.
Use them everywhere instead of naming roles directly in sharing rules — when the org
restructures, you edit one group instead of forty rules.

#### Manual Sharing and Teams

- **Manual sharing** — the Sharing button on a record, granting one user or group access.
  Available only when OWD is more restrictive than Public Read/Write. Removed automatically
  when record ownership changes.
- **Account Teams, Opportunity Teams, Case Teams** — a repeatable per-record membership
  model with a defined access level per team role. Better than manual sharing for anything
  recurring.

#### Implicit Sharing

Automatic and not configurable. Worth knowing because it explains surprising visibility:

- A user with access to a Contact, Case, or Opportunity gets **Read** on the parent Account
- A user with access to an Account may get access to child Contacts/Cases/Opportunities
  depending on the Account's related-object access settings
- Portal/Experience Cloud users get implicit access to their own Account and Contact

#### Restriction Rules

The only declarative mechanism that **narrows** access. Filters what a user can see even
when other mechanisms grant it.

```
Restriction Rule on: Case
  User criteria:   User.Department = "Support"
  Record criteria: Case.Confidential__c = false
  Effect: Support users never see confidential Cases, even if they own them
```

Available on custom objects and a set of standard objects including Task, Event,
ContentDocument, and TimeSheet. Check current object support before designing around it.

#### Scoping Rules

Set a **default filter** on what a user sees, without removing their ability to access the
rest. Useful for focus rather than security — a rep defaults to their region's Accounts but
can still search outside it. Do not use as a security control.

### Permission Set Groups and Muting

Permission set groups combine permission sets into a single assignable unit. A **muting
permission set** inside a group subtracts specific permissions from the group's aggregate.

```
Group: "Sales Manager"
  ├── PS: Object_Opportunity_Full         (includes Delete)
  ├── PS: Object_Account_Edit
  ├── PS: Report_Builder_Access
  └── Muting PS: Mute_Delete              (removes Opportunity Delete)

Net effect: everything in the group, minus Opportunity Delete
```

This solves the classic problem of "this role needs almost the standard bundle" without
cloning and diverging the whole set.

### The Profile Minimisation Migration

Most inherited orgs have 30–80 profiles. Target: one profile per *license type* per major
user population, typically fewer than eight.

Migration sequence:

1. Export current profile permissions. Setup → Profiles is unusable for this at scale; use
   the Metadata API, SFDX (`sfdx force:source:retrieve -m Profile`), or a tool such as
   Perm Comparator.
2. Identify the intersection — permissions common to *all* profiles in a population. That
   becomes the base profile.
3. Identify each delta — a specific object, app, or field difference. Each becomes a
   permission set with a single concern.
4. Build permission set groups per persona from those permission sets.
5. Assign the group to a pilot user population and validate with login-as.
6. Reassign users to the minimal profile and cut over.
7. Delete the obsolete profiles. Profiles cannot be deleted while users are assigned.

Budget realistically: this is a multi-week project in a mature org, not an afternoon.

### Login and Session Security

| Control | Location | Recommendation |
|---|---|---|
| Multi-factor authentication | Session Settings / permission set | Mandatory for all direct-login users |
| Login IP Ranges | Profile | Restrict integration users to known source IPs |
| Login Hours | Profile | Useful for contractor and offshore populations |
| Session Timeout | Session Settings | 2 hours or less for orgs holding personal data |
| Lock sessions to IP | Session Settings | Enable unless a proxy/VPN makes it unworkable |
| Password policy | Password Policies | 90-day expiry, 8+ chars, complexity, 5-password history |
| Trusted IP Ranges | Network Access | Bypasses identity verification — audit these carefully |
| API access | Permission set | `API Enabled` off by default for standard users |

**Password Never Expires** on a human user is an audit finding. On an integration user it
is defensible, paired with IP restriction and a documented rotation schedule.

### Shield Platform Encryption

Encrypts data at rest at the field level while preserving most platform functionality.
Additional licence cost. Trade-offs to state clearly before recommending:

- Encrypted fields cannot be used in most SOQL filters, sorts, or formula fields
- Roll-up summaries on encrypted fields are unsupported
- Some encrypted fields break in reports, list-view filters, and duplicate matching rules
- Key management becomes an operational responsibility

For most requirements, Classic Encrypted Text fields plus strict FLS is sufficient. Shield
is for regulatory mandates, not general prudence.

### Security Audit Checklist

Run quarterly, and always before a client handover.

- [ ] `Modify All Data` / `View All Data` — enumerate every holder, justify each
- [ ] Users with the System Administrator profile — should be a very short list
- [ ] Inactive users still holding records — reassign ownership before deactivating
- [ ] Integration users — IP-restricted, no UI login, minimum viable permissions
- [ ] Permission sets granting `Manage Users`, `Customize Application`, `Author Apex`
- [ ] Health Check score and every "High Risk" item (Setup → Health Check)
- [ ] Setup Audit Trail reviewed for the last 6 months of config changes
- [ ] Connected Apps and their OAuth scopes — remove anything unrecognised
- [ ] Public Groups and Queues with unexpected membership
- [ ] Sharing rules that grant broad access to `All Internal Users`
- [ ] Guest user profile permissions if Experience Cloud is enabled — a recurring source
      of data exposure incidents
- [ ] Field-level security on any field holding personal, financial, or health data
- [ ] Session settings, password policy, MFA enforcement
- [ ] Login IP ranges on privileged profiles

### Guest User Hardening (Experience Cloud)

If any Experience Cloud site is active, the guest user profile is internet-facing. Verify:

- Object permissions limited to exactly what the public site needs
- `Secure guest user record access` enabled (Setup → Sharing Settings)
- Guest user sharing rules reviewed individually — these are the usual culprit
- No `View All` on any object
- Field-level security on every exposed object

This is the single most common source of accidental Salesforce data exposure. Treat it as
a mandatory item on every audit, whether or not anyone mentioned the community.

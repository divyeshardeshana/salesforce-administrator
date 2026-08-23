# Lightning Experience & UX

---

## The Page Layer

Salesforce splits what a user sees across four independent configuration surfaces. Most
"I can't find the field" tickets are a mismatch between them.

```
1. Lightning App          → which tabs, navigation style, utility bar
2. Lightning Record Page  → component arrangement, tabs, visibility rules
3. Page Layout            → related lists, buttons, actions, and (legacy) field placement
4. Dynamic Forms          → field placement with per-field visibility, replacing layout fields
```

A field can be visible in Dynamic Forms but hidden by field-level security. An action can
be on the page layout but absent from the record page because Dynamic Actions is enabled.
Diagnose in the order above, then check FLS.

---

## Lightning Apps

```
Setup → App Manager → New Lightning App
  App Name, Developer Name, branding image, primary colour
  Navigation Style:  Standard  |  Console
  Setup Experience:  with or without Service Setup
  Utility Items
  Navigation Items (tabs)
  User Profiles / Permission Sets assigned
```

### Standard vs Console Navigation

| | Standard | Console |
|---|---|---|
| Record opens | Replaces the page | Workspace tab, with subtabs |
| Split view | No | Yes — list on the left |
| Utility bar | Yes | Yes |
| Best for | Sales, general use, executives | Support, high-volume record handling |
| Tab persistence | No | Yes — tabs survive navigation |

Console navigation is not automatically better. A sales rep working three Opportunities a
day gains nothing from tab management and loses the simpler mental model. Reserve it for
genuinely high-throughput work.

### Utility Bar

Persistent bottom-docked items, available in both navigation styles.

```
Utility Items
  History              — recently viewed, with pinning
  Notes                — quick capture without leaving the record
  Omni-Channel         — required for Service Cloud routing
  Open CTI Softphone   — telephony
  Flow                 — a screen flow available from anywhere
  Recent Items, Macros, Rich Text
```

Each item has a Panel Width/Height, "Start automatically" and "Load in background" options.
Loading everything in the background slows app startup; enable it only for items the agent
uses every session.

A Screen Flow in the utility bar is an underused pattern — a global "Log a call" or "Quick
create" available on every page without a record context.

### Navigation Items and Personalisation

Tabs added to the app appear for all assigned users. Users can personalise their own
navigation bar unless you disable it per app ("Disable end user personalisation of nav
items"). Leave personalisation on for most apps; disable it where a compliance or training
requirement demands a fixed layout.

---

## Lightning Record Pages

### Structure

```
Lightning App Builder → Record Page
  Template:  Header and Right Sidebar | Header and Two Columns | Three Columns | custom
  Regions:   drag components into each

  Components (standard):
    Highlights Panel        ← driven by the Compact Layout
    Record Detail / Field Section (Dynamic Forms)
    Related Lists — Single / Related Lists
    Activity, Chatter, Path, Tabs, Accordion
    Report Chart, Rich Text, Flow, Visualforce, Custom LWC
```

### Assignment

A record page is assigned by **App, Record Type, and Profile** — in that order of
specificity.

```
Assignment
  Org Default                       ← the fallback
  App Default                       ← per Lightning app
  App, Record Type, and Profile     ← most specific, wins
```

This three-way assignment is why "the page looks different for me" happens. Check the
assignment matrix before assuming a component is broken.

### Component Visibility Filters

Every component supports visibility rules based on record field values, device, user
fields, or permissions.

```
Component: "Escalation Details" section
Set Component Visibility
  Filter 1:  Record → Priority          equals   "High"
  Filter 2:  User   → Profile Name      equals   "Support Manager"
  Logic:     1 AND 2
```

Available filter sources: Record field, Device (phone/tablet/desktop), User field,
Permissions (custom permission or user permission), and Advanced with custom logic.

Filtering on **custom permission** rather than profile name is the durable choice — profile
names change, custom permissions are assignable and auditable.

### Tabs and Accordion

Use Tabs to reduce vertical scroll on record-heavy objects:

```
Tabs component
  Tab "Details"       → Record Detail / Field Sections
  Tab "Related"       → Related Lists
  Tab "Activity"      → Activity component
  Tab "Financials"    → visible only when Type = "Customer"
```

Accordion collapses sections in a single column. Prefer Tabs for peer content, Accordion
for progressive disclosure of secondary detail.

### Performance

The App Builder shows a **Page Analysis** score. Practical rules:

- Fewer than about 25 components per page
- Report charts are expensive — one per page maximum, ideally none on high-traffic pages
- Custom LWCs making their own Apex calls compound; each is a separate round trip
- Related lists render lazily below the fold, so pushing them down helps perceived speed
- Component visibility filters do not prevent loading in all cases; they hide

---

## Dynamic Forms

Field placement moved from the page layout into the Lightning record page, with per-field
and per-section visibility.

### Migration

```
App Builder → open the record page
  → select the Record Detail component
  → "Upgrade Now" / Fields → Migrate
  → converts layout sections into Field Section components
```

After migration, the page layout still controls **related lists, buttons, and actions**.
Field placement no longer comes from it. This split confuses people: the layout is not
obsolete, it is reduced.

### Why It Matters

| Problem before | Solution with Dynamic Forms |
|---|---|
| Forty page layouts to vary field visibility | One page, field-level visibility filters |
| A record type created solely to hide fields | Visibility filter on the section |
| Fields visible to the wrong audience | Filter on custom permission |
| Long single-column detail sections | Multi-column field sections in any region |

```
Field Section: "Contract Terms"
  Visibility:  Record → Stage  equals  "Negotiation"  OR  "Closed Won"

Field: Discount_Approval_Notes__c
  Visibility:  User → has custom permission "View_Discount_Notes"
```

### Constraints

- Supported on most standard and all custom objects; check the current support list for
  edge-case standard objects
- The mobile app renders Dynamic Forms, but verify — complex visibility logic behaves
  differently on narrow viewports
- Required fields still come from the field definition and validation rules, not from the
  visual placement

---

## Dynamic Actions

Actions in the highlights panel, controlled by the record page rather than the layout.

```
App Builder → Highlights Panel → Actions → "Upgrade Now" / Add Action
  Each action gets its own visibility filter
```

```
Action: "Submit for Approval"
  Visibility:  Record → Status         equals  "Draft"
           AND Record → Amount         greater than  50000

Action: "Escalate"
  Visibility:  Record → Priority       not equal to  "High"
           AND User   → has permission "Escalate_Cases"
```

This replaces the pattern of cloning a page layout purely to show or hide a button.

Once Dynamic Actions is enabled on a page, the layout's action list is ignored **for that
page**. Actions must be re-added in App Builder. Forgetting this is why buttons "disappear"
after enabling it.

---

## Actions and Buttons

| Type | Where configured | Behaviour |
|---|---|---|
| **Quick Action — Create a Record** | Object Manager → Buttons, Links, and Actions | Modal create form, prefilled via field defaults |
| **Quick Action — Update a Record** | Same | Modal update of specified fields |
| **Quick Action — Log a Call** | Same | Task creation from the feed |
| **Quick Action — Send Email** | Same | Email composer, needs the Email action enabled |
| **Quick Action — Flow** | Same | Launches a Screen Flow with `recordId` |
| **Quick Action — LWC** | Same | Custom component in a modal |
| **Global Action** | Setup → Global Actions | Available from the header anywhere |
| **Custom Button — URL** | Object Manager | Navigates; used for external links |
| **Custom Button — JavaScript** | Object Manager | **Not supported in Lightning** — must be rebuilt |

### Predefined Field Values

Quick Actions support predefined values, which is how you prefill a child record:

```
Quick Action on Account: "New Support Case"
  Target Object: Case
  Predefined Field Values:
    AccountId  = {!Account.Id}
    Origin     = "Phone"
    Priority   = "Medium"
    RecordTypeId → set via the action's record type selection
```

Prefer this over a URL hack with query parameters — it survives releases and respects
validation.

### Migrating JavaScript Buttons

Classic JavaScript buttons do not run in Lightning. Replacement paths, in order of
preference:

1. **Quick Action with predefined values** — for simple field setting
2. **Screen Flow launched from an action** — for anything conditional or multi-step
3. **Record-triggered Flow** — if the button was really just "set these fields", automate it
4. **LWC with an Apex controller** — last resort, requires a developer

Inventory these before any Classic-to-Lightning migration. They are the most common source
of "the users say Lightning is missing features."

---

## List Views

Frequently the primary work surface, and frequently neglected.

```
Object tab → List View Controls → New
  Filters:            up to 10, with filter logic
  Visible to:         Only me | All users | Specific groups and roles
  Displayed columns, widths, and sort
  List View Charts:   bar, donut, or metric alongside the list
```

### Features Worth Configuring

- **Inline editing** — enabled when the list view has no filtered-out edits; a massive time
  saver for data cleanup
- **Mass actions** — select records and apply an action; enable the relevant actions on the
  layout's list view section
- **Kanban view** — group by a picklist, drag to change value. Excellent for pipeline and
  case triage. Set the summary field to Amount for a value-per-column view
- **Pinned list views** — the pin persists per user per object
- **List view charts** — a small chart above the list, cheaper than a dashboard for
  at-a-glance context
- **Sharing** — share with public groups and roles, never individuals

### Governance

List views proliferate exactly like reports. Adopt a convention and prune:

```
[Scope] — [Filter]

My Open Opportunities — This Quarter
Team — Cases Awaiting Response
All — Accounts Missing VAT Number
```

Views set to "Only me" by a departed user become unreachable clutter. Audit annually.

---

## Path

```
Setup → Path Settings → New Path
  Object + Record Type + picklist field (Stage, Status, Lead Status)
  Per step:
    Key Fields:  up to 5, inline-editable from the Path
    Guidance for Success:  rich text
  Options: celebration animation on a chosen step
```

Path works on Opportunity, Lead, Case, Contract, Quote, Campaign, and custom objects with a
picklist. It is guidance, not enforcement — a user can still advance with Key Fields blank.

Pair Path (guidance, all fields) with validation rules (enforcement, two or three critical
fields). Enforcing everything through validation produces workarounds; enforcing nothing
produces empty data.

Populate the guidance text with the org's actual methodology. An empty Path is worse than
no Path — it signals the feature was configured and abandoned.

---

## In-App Guidance

```
Setup → In-App Guidance → Add
  Type:      Docked prompt | Floating prompt | Walkthrough
  Location:  a specific Lightning page
  Audience:  profile, permission set, or user
  Schedule:  start and end date, frequency
```

Use for:

- Feature launches — a floating prompt on the page where the new field appears
- Onboarding — a walkthrough on first login for new users
- Behaviour change — a docked prompt explaining why a required field appeared

Set an **end date** on every prompt. Permanent prompts become wallpaper within a week and
train users to dismiss without reading.

Limits: 50 active prompts per org, 20 per page. Walkthroughs have their own quota.

---

## Compact Layouts

Determines the highlights panel fields, the mobile record header, and lookup hover cards.

```
Object Manager → Compact Layouts → New
  Fields (up to 10, first 4-7 shown depending on context)
  Assign to record types
```

Get this right early. A compact layout showing "Created Date" and "Record Type" rather than
"Amount" and "Stage" is a small daily friction that users notice and rarely report.

---

## Home Pages and App Pages

**Home Page** — assigned by App and Profile, same as record pages. Useful components:
Assistant, Today's Tasks, Today's Events, Recent Records, Report Chart, Dashboard, Rich
Text for announcements, Flow.

**App Page** — a standalone page with a tab, no record context. Good for landing pages,
custom dashboards, and utility flows.

Keep Home Pages light. A Home Page with four report charts is the slowest page in the org
and the first thing every user loads every morning.

---

## Mobile

The Salesforce mobile app renders Lightning pages with a phone form factor.

### What Differs

- Component visibility supports a **Device** filter — build phone-specific arrangements
- Related lists collapse
- The Compact Layout drives the record header
- Some components are desktop-only; the App Builder marks them
- Actions come from the **Salesforce Mobile and Lightning Experience Actions** section on
  the page layout

### Configuration

```
Setup → Salesforce Mobile App → Mobile Navigation
  Order the tabs a mobile user sees — this is separate from the desktop app
```

**Briefcase Builder** (Setup → Briefcase Builder) defines records cached for offline
access, with filters and record limits per object. Relevant for field service and sales
teams with poor connectivity.

Test on an actual device before declaring mobile support. The App Builder's phone preview
approximates; it does not replicate.

---

## Classic to Lightning Migration

### Assessment

Run the **Lightning Experience Readiness Check** (Setup → Lightning Experience
Transition Assistant). It reports on JavaScript buttons, unsupported Visualforce, hard-coded
URLs, and features without Lightning equivalents.

### Known Gaps to Plan For

| Classic feature | Lightning position |
|---|---|
| JavaScript custom buttons | Not supported — rebuild as actions or Flows |
| Inline Visualforce on layouts | Works, but often better rebuilt as LWC |
| `srcUp()` and console JS API calls | Replaced by Lightning Console JS API |
| Classic Knowledge | Migrate to Lightning Knowledge — one-way |
| Some URL hacks for prefilled records | Replace with Quick Actions and predefined values |
| Classic email templates | Still function; Lightning templates are better supported |

### Sequence

1. Readiness Check, and inventory every finding
2. Rebuild JavaScript buttons and URL hacks
3. Build Lightning apps, record pages, and compact layouts in a sandbox
4. Pilot with a small, willing group — enable Lightning per profile
5. Train, with In-App Guidance and a "what changed" one-pager
6. Roll out by profile, not all at once
7. Leave the Classic switch available for a defined period, then remove it

Do not migrate and redesign simultaneously unless the org is small. Move to Lightning
first with equivalent functionality, then improve.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| A field is missing from the record page | Dynamic Forms migrated — the field is no longer driven by the page layout. Check the Field Sections |
| Buttons disappeared after enabling Dynamic Actions | Actions must be re-added in App Builder; the layout list is ignored |
| Different users see different record pages | Assignment is by App + Record Type + Profile; check the assignment matrix |
| A component visibility filter never matches | Filtering on Profile Name after a rename; use a custom permission instead |
| JavaScript button does nothing | Not supported in Lightning; must be rebuilt |
| Record page is slow | Too many components, report charts, or LWCs making independent Apex calls. Run Page Analysis |
| Highlights panel shows the wrong fields | Compact Layout, not the page layout or Dynamic Forms |
| Action missing on mobile | Not in the "Salesforce Mobile and Lightning Experience Actions" section |
| List view not visible to a colleague | Set to "Only me"; change the visibility |
| Kanban unavailable | The grouping field is not a picklist, or the object does not support it |
| Path shows no guidance | Guidance for Success left empty at that step |
| In-App Guidance prompt not appearing | Audience filter, schedule window, or the 20-per-page limit |
| Utility bar item missing | Not added to the Lightning app, or the user lacks the underlying permission |
| Related list missing | Related lists still come from the page layout, even with Dynamic Forms |

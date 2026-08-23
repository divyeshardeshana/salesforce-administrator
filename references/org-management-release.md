# Org Management, Deployment & Release

---

## Part 1 — Environment Strategy

### Sandbox Types

| Type | Data | Refresh interval | Storage | Typical use |
|---|---|---|---|---|
| **Developer** | Metadata only | 1 day | 200 MB | Individual config work, isolated experiments |
| **Developer Pro** | Metadata only | 1 day | 1 GB | Larger dev work, integration testing with seeded data |
| **Partial Copy** | Metadata + sampled data via template | 5 days | 5 GB | UAT, training, integration testing with realistic data |
| **Full Copy** | Complete production copy | 29 days | Production size | Performance testing, final staging, data migration rehearsal |

Edition entitlements vary. Enterprise typically includes 25 Developer, 1 Developer Pro, and
1 Partial Copy; Full Copy is usually a paid add-on.

### A Workable Environment Model

For a small team or a consultancy running client orgs:

```
DEV (Developer sandbox, one per admin/developer)
  ↓ change set or DX
UAT / STAGING (Partial Copy — realistic data, business users test here)
  ↓ change set or DX
PRODUCTION
```

For larger programmes, add a shared INT (integration) environment between DEV and UAT
where parallel workstreams merge before business testing.

Rules that make this work:

- **Nobody configures in production** except reports, dashboards, list views, and user
  administration
- **UAT refreshes on a known cadence** — a 29-day-stale Full Copy invalidates testing
- **Dev sandboxes are disposable** — refresh freely, keep nothing important only there

### Sandbox Refresh Discipline

Refreshing overwrites everything in the sandbox. Before refreshing:

- [ ] Export any metadata that exists only in the sandbox
- [ ] Note all sandbox-specific configuration that must be reapplied
- [ ] Warn every user of the sandbox — refresh destroys their in-progress work

After refreshing, reapply:

- [ ] Deactivate outbound email (Setup → Deliverability → **System email only**) —
      this is critical and the omission that causes production emails to be sent to real
      customers from a sandbox
- [ ] Repoint integrations to sandbox endpoints; disable production-targeting connections
- [ ] Reset or scramble personal data if the sandbox is accessible to a wider audience
- [ ] Reactivate the users who need access — sandbox usernames get a suffix appended
- [ ] Reset passwords for test users
- [ ] Reconfigure named credentials, custom settings, and custom metadata pointing at
      external systems
- [ ] Deactivate scheduled jobs and scheduled flows that would fire against live systems

A written sandbox post-refresh checklist, kept in the org's documentation, is one of the
highest-value artefacts an admin can produce.

---

## Part 2 — Deployment

### Change Sets

The declarative path. No tooling required, works entirely in the UI.

```
Setup → Outbound Change Sets → New
  1. Name it with a convention:  REL-2026-08_OpportunityAutomation
  2. Add components
  3. Click "View/Add Dependencies"      ← the step people skip
  4. Upload to the target org
  5. In target: Setup → Inbound Change Sets → Validate
  6. Deploy
```

**Always Validate before Deploy.** Validation runs the full deployment including Apex tests
without committing. A failed validation costs minutes; a failed deployment can leave a
half-configured org.

**Always run View/Add Dependencies.** A Flow referencing a custom field that is not in the
change set fails on deployment with an unhelpful error.

Known change set limitations — plan around these:

- Cannot delete components; deletions must be done manually in the target
- Does not include: many field attributes on standard fields, some picklist value changes,
  record type picklist assignments in all cases, translations, list views owned by others
- Profile and permission set contents deploy only for components *also present in the
  change set* — a profile in a change set does not carry its full permission state
- Once uploaded, a change set is frozen; changes require a clone and re-upload
- Connection must be pre-established (Setup → Deployment Settings) and is only permitted
  between orgs with a common production ancestor

### Salesforce DX and Source-Driven Deployment

Metadata as files in version control. Preferred for anything beyond a small team.

```bash
# Authorise orgs
sf org login web --alias dev-sandbox
sf org login web --alias production

# Retrieve metadata
sf project retrieve start --metadata Flow:Opportunity_BeforeSave_Main --target-org dev-sandbox
sf project retrieve start --metadata "CustomObject:Invoice__c" --target-org dev-sandbox

# Validate against production without deploying
sf project deploy validate --source-dir force-app --target-org production --test-level RunLocalTests

# Deploy
sf project deploy start --source-dir force-app --target-org production

# Quick deploy a previously validated run (skips re-running tests)
sf project deploy quick --job-id <validation-job-id> --target-org production
```

The **validate then quick-deploy** pattern matters in production. Validation runs the tests
(often 20–60 minutes in a large org); quick deploy then commits within ten minutes using
the validated result, provided it is within four days. This shrinks the production
maintenance window dramatically.

Benefits over change sets that justify the learning curve:

- Version control, diffs, branching, and code review on configuration
- Deletions are possible via destructive changes
- Repeatable, scriptable, CI/CD-capable
- The full metadata surface, including items change sets cannot carry

### Unlocked Packages

Metadata grouped into a versioned, installable unit. Worth considering when the same
configuration is deployed to multiple orgs — a consultancy shipping a standard accelerator
to every client is the canonical case.

```bash
sf package create --name "Client-Base-Config" --package-type Unlocked --path force-app
sf package version create --package "Client-Base-Config" --installation-key-bypass --wait 20
sf package install --package "Client-Base-Config@1.0.0-1" --target-org client-prod --wait 20
```

Trade-off: package metadata is owned by the package and cannot be freely edited in the
target org, which is either the main benefit or the main obstacle depending on the client.

### Deployment Runbook

Every production deployment should follow a written sequence:

**Before**
- [ ] Change validated in the target org
- [ ] Deployment window agreed and communicated
- [ ] Rollback plan written — for config, usually "deploy the previous version"; for data
      changes, an exported backup
- [ ] Users notified if there is any behaviour change
- [ ] Confirm the org is not inside a seasonal release maintenance window

**During**
- [ ] Deploy
- [ ] Verify the acceptance test defined at requirement time
- [ ] Activate anything deployed inactive (Flows deploy inactive by default in some paths)
- [ ] Reassign permission sets if new ones were introduced

**After**
- [ ] Smoke test as a non-admin user of each affected persona
- [ ] Check Setup Audit Trail reflects what was expected
- [ ] Monitor for 24–48 hours: error emails, Flow error logs, user reports
- [ ] Update the org change log

**Never deploy on a Friday.** The failure mode is discovered on Monday by users, not on
Friday evening by you.

---

## Part 3 — Data Management

### Tool Selection

| Tool | Volume | Objects | Best for |
|---|---|---|---|
| **Data Import Wizard** | 50,000 records | Accounts, Contacts, Leads, Solutions, Campaign Members, custom objects | Simple imports with duplicate matching |
| **Data Loader** | 5 million records | All | Bulk operations, scheduled loads via CLI, exports including deleted records |
| **Data Loader CLI / sf CLI** | 5 million+ | All | Scripted, repeatable, scheduled |
| **Third-party (Dataloader.io, Jitterbit, Talend)** | Varies | All | Transformation during load, scheduling, error handling |
| **Bulk API 2.0** | 150 million/day | All | Programmatic high-volume |

### Data Loader Operations

| Operation | Behaviour | Requires |
|---|---|---|
| Insert | Create new records | No Id |
| Update | Modify existing | Salesforce Id |
| **Upsert** | Update if match, insert if not | External Id field |
| Delete | Move to Recycle Bin | Salesforce Id |
| Hard Delete | Bypass Recycle Bin | Id + "Bulk API Hard Delete" permission |
| Export | Query records | SOQL |
| Export All | Includes soft-deleted and archived | SOQL |

**Upsert with an External Id is the pattern to standardise on.** It is idempotent: rerunning
the same file does not duplicate. Create a `Legacy_System_Id__c` text field marked External
Id and Unique on every object receiving migrated data.

### Pre-Load Checklist

- [ ] Full backup of affected objects (Export All, including Id fields)
- [ ] Load into a Full or Partial Copy sandbox first, with the identical file
- [ ] Record counts verified before and after
- [ ] Automation impact assessed — will 50,000 inserts fire 50,000 Flow interviews and
      50,000 emails?
- [ ] Decide explicitly whether to disable automation, and document the reactivation step
- [ ] Duplicate rules considered — Data Loader can bypass them via settings
- [ ] Validation rule bypass permission assigned to the load user, if needed
- [ ] Owner assignment planned — records default to the loading user otherwise
- [ ] Batch size tuned (200 default; reduce to 1 if hitting CPU limits, raise via Bulk API
      for simple loads)
- [ ] Field mapping file (`.sdl`) saved for repeatability
- [ ] Success and error files written to a known directory and reviewed after

### Disabling Automation for a Load

Options, in order of preference:

1. **Custom permission bypass** built into validation rules and Flow entry criteria — the
   surgical option, described in the flow-automation reference
2. **Deactivate specific Flows** — record exactly which, in a written list, with a
   reactivation checkbox
3. **Hierarchy custom setting** — `Automation_Settings__c.Disable_Flows__c` checked at the
   top of every Flow's entry criteria, set true for the loading user only

Never deactivate automation org-wide and rely on memory to restore it.

### Duplicate Management

Two components working together:

**Matching Rule** — defines what "the same record" means.

```
Matching Rule: Account_By_Name_And_City
  Account Name    → Fuzzy: Company Name
  Billing City    → Exact
  Match Logic:    Name AND City
```

**Duplicate Rule** — defines what happens on a match.

```
Duplicate Rule: Account_Duplicate_Prevention
  Matching Rule:      Account_By_Name_And_City
  On Create:          Allow, with alert  (or Block)
  On Edit:            Allow, with alert
  Alert text:         "A similar Account may already exist."
  Report on dupes:    Enabled
  Bypass for:         permission set with "Bypass Duplicate Rules"
```

Recommendations:

- Start with **Allow + Alert**, not Block. Blocking on day one generates support tickets
  and teaches users to work around the system
- Enable **Report on duplicates** — it populates the Duplicate Record Set object, which is
  reportable and gives you the size of the problem
- Standard matching rules exist for Account, Contact, and Lead and are a reasonable
  starting point
- Matching rules must be **activated** after creation; an inactive matching rule silently
  does nothing

### Data Quality Monitoring

Build these as scheduled reports subscribed to the admin:

| Report | Detects |
|---|---|
| Accounts with no Contacts | Orphaned records |
| Contacts with no Email and no Phone | Unreachable records |
| Opportunities with Close Date in the past and open Stage | Stale pipeline |
| Records owned by inactive users | Ownership gaps |
| Accounts missing required business fields | Incomplete data |
| Duplicate Record Sets created this week | Duplicate rate trend |
| Leads not converted, created > 90 days ago | Follow-up failure |

Set report subscriptions with a **condition** so they only send when the count exceeds a
threshold. A weekly email showing zero problems trains people to delete it unread.

### Backup and Recovery

Salesforce's own guidance is that native tools are not a backup strategy.

| Mechanism | Coverage | Limitation |
|---|---|---|
| **Recycle Bin** | 15 days, limited capacity | Not a backup |
| **Data Export Service** | Weekly or monthly full export, CSV | Manual restore, no metadata, no relationships preserved automatically |
| **Data Loader Export All** | On demand | Manual, scriptable |
| **Sandbox refresh** | Point-in-time copy | Not restorable to production |
| **Third-party backup** | Continuous, restorable, metadata included | Paid |

For any org holding data the business cannot recreate, a third-party backup with tested
restore is the correct recommendation. State it plainly and let the client decide with
full information.

At minimum, enable **Data Export Service** (Setup → Data Export) weekly and confirm someone
downloads and retains the files. An export nobody collects is not a backup.

---

## Part 4 — User Administration

### Onboarding

```
1. Create user  → correct Profile, Role, and licence type
2. Assign permission set group for the persona
3. Set Locale, Language, Time Zone, Currency (Locale drives date and number formats)
4. Assign to public groups and queues as needed
5. Configure Manager field — used by approval hierarchies and reports
6. Set Call Center, Territory, Forecast hierarchy if applicable
7. Verify with Login As before handing over
```

**Login As** (Setup → Users → Login) is the most underused administrative tool. Ten seconds
of verification prevents a first-day support ticket. Requires "Administrators Can Log in as
Any User" enabled in Login Access Policies.

### Offboarding

Deactivating a user has consequences that must be handled first:

```
1. Reassign record ownership   (Setup → Mass Transfer Records, or Data Loader)
2. Reassign open Tasks and Events
3. Reassign or transfer report and dashboard ownership
4. Reassign dashboards where the departing user is the running user
5. Reassign scheduled reports and Apex jobs owned by the user
6. Remove from queues, public groups, and territories
7. Check for approval processes where they are a named approver
8. Reassign any Flows where they are the error-email recipient
9. Deactivate — which frees the licence
10. Freeze first if the deactivation might be reversed
```

**Freeze** (on the user detail page) blocks login immediately without releasing the licence
or triggering ownership issues. Use it the moment someone leaves, then work through the
reassignment list, then deactivate.

Users cannot be deleted, only deactivated. Their name persists on historical records, which
is correct for audit purposes.

### Licence Management

Setup → Company Information shows licences used and available across:

- **Salesforce** — full CRM access
- **Salesforce Platform** — custom objects and a subset of standard objects; no
  Opportunity, Case, Forecast
- **Identity** — SSO only
- **Feature licences** — Marketing User, Knowledge User, CRM Analytics, Service Cloud
- **Permission set licences** — separately assignable capability grants

A recurring cost-saving audit: users on a full Salesforce licence who only touch custom
objects can often move to Platform. Check what each user actually accesses via login
history and object usage before proposing this.

---

## Part 5 — Org Health and Governance

### Health Check

Setup → Health Check. Scores the org's security settings against a Salesforce Baseline
Standard and lists every setting below it, grouped by risk.

Run it before every major release and record the score. A declining score is a governance
signal even when no single item looks alarming.

Common findings and their fixes:

| Finding | Fix |
|---|---|
| Session timeout too long | Session Settings → 2 hours or less |
| Password expiry disabled | Password Policies → 90 days |
| No MFA requirement | Permission set with "Multi-Factor Authentication for User Interface Logins" |
| Clickjack protection off | Session Settings → enable all clickjack options |
| Login IP ranges not set | Profile-level restriction for privileged profiles |
| HTTPS not required | Session Settings → Require secure connections |

### Salesforce Optimizer

Setup → Optimizer. Generates a report covering unused fields, unused reports, page layouts
per object, validation rule count, Apex limits, storage usage, and profile sprawl.

Best used as a technical-debt inventory at the start of an engagement with an inherited
org. It quantifies what "this org is a mess" actually means in a form a client will accept.

### Setup Audit Trail

Setup → View Setup Audit Trail. The last six months of configuration changes: who, what,
when. Downloadable as CSV.

Uses:

- Answering "who changed this and when" after an unexplained behaviour change
- Compliance evidence for change control
- Handover audit — what did the previous admin actually do

Six months is the retention limit. For longer retention, download monthly and archive, or
use Event Monitoring. Set a recurring calendar reminder; the data is gone once it ages out.

### Event Monitoring

Additional licence. Provides detailed log files covering login events, report exports, API
calls, page views, and Apex execution. Relevant when:

- Regulatory requirements demand access logging
- Investigating suspected data exfiltration
- Diagnosing performance issues at the transaction level
- Tracking adoption at a granular level

Without it, `LoginHistory` and Setup Audit Trail are the available signals.

### Storage Management

Setup → Storage Usage. Two pools:

- **Data storage** — records, typically 2 KB each regardless of field count
- **File storage** — attachments, Files, Documents, Content

When approaching the limit, in order of preference:

1. Delete or archive obsolete records — old Leads, converted Leads, closed Cases past
   retention, Reporting Snapshot history
2. Move large files to external storage with a link, or Salesforce Files Connect
3. Empty the Recycle Bin (Setup → Mass Delete Records → Empty Recycle Bin)
4. Purge Field History if retention is not required
5. Purchase additional storage

Run a storage review annually. Orgs cross the threshold silently and the first symptom is a
failed data load.

### Org Limits Monitoring

Setup → System Overview shows usage against key limits: custom objects, custom fields per
object, Apex characters, API requests in the last 24 hours, data and file storage, active
Flow versions.

Worth checking quarterly. The limits that bite in practice:

- **Custom fields per object** — 800 in Enterprise, 500 for some standard objects. Orgs
  hit this on Account and Opportunity
- **API requests per 24 hours** — an integration in a retry loop exhausts this and blocks
  every other integration
- **Active validation rules** — 500 per object
- **Active flows** — 2,000 active flow versions per org

---

## Part 6 — Seasonal Releases

Salesforce ships three major releases a year — Spring, Summer, and Winter — automatically,
whether or not the org is ready.

### Release Timeline

```
~5 weeks before   Release notes published
~4 weeks before   Sandbox preview begins (preview instances upgraded first)
~1 week before    Release readiness webinars, Release Overview deck available
Release weekend   Production instances upgraded in waves by instance
```

### Pre-Release Routine

1. **Find the org's instance and upgrade date.** Setup → Company Information → Instance,
   then check the Salesforce Trust site for that instance's schedule.
2. **Read the release notes filtered to enabled features.** The full notes run to hundreds
   of pages; filter to the clouds and features actually in use.
3. **Identify the critical updates.** Setup → Release Updates lists changes that will be
   enforced on a specific release, with a test-run option. These are the items that break
   orgs. Test each one before its enforcement date.
4. **Check the sandbox preview.** If the org has a preview-instance sandbox, it receives the
   release early — this is the window to test.
5. **Assess third-party packages.** Confirm ISVs have certified their packages for the
   release.
6. **Communicate.** A short summary of user-visible changes prevents a Monday-morning wave
   of confusion.

### Sandbox Preview

Sandboxes sit on either preview or non-preview instances. To test a release before
production, the sandbox must be on a preview instance — which is determined by when and
where it was created. Salesforce publishes a preview instance list before each release; if
a sandbox needs to be on preview, it must be created or refreshed within the stated window.

Missing that window means no pre-release testing for that cycle. Put the dates in the
calendar a quarter ahead.

### Release Updates

Setup → Release Updates is the most operationally important page in the org. It lists
platform changes with enforcement dates, each with:

- A description of the behaviour change
- A **Test Run** option that enables the change so it can be validated
- An enforcement date after which it applies regardless

Work through these on a schedule. An unaddressed release update with a near enforcement
date is the most likely cause of a future unexplained breakage.

---

## Part 7 — Org Documentation

The deliverable that separates a maintainable org from an inherited mess. Minimum viable
documentation:

| Artefact | Contents |
|---|---|
| **Data dictionary** | Every custom object and field: API name, type, purpose, source, who populates it |
| **Automation inventory** | Every Flow: object, trigger point, purpose, owner, last reviewed |
| **Security model** | Profiles, permission sets, permission set groups, OWD per object, sharing rules, and the rationale for each |
| **Integration register** | Every connected system: direction, objects touched, authentication method, integration user, owner, failure contact |
| **Change log** | Date, change, requester, deployer, rollback status |
| **Sandbox post-refresh checklist** | The org-specific steps described earlier |
| **Runbook** | Recurring admin tasks and their schedule |

Field descriptions and Flow descriptions maintained in the org itself are worth more than
any external document, because they cannot drift out of sync. Treat external documentation
as a supplement, not a substitute.

### Inherited Org Assessment

When taking over an unfamiliar org, work through this sequence:

1. **Optimizer report** — the quantified technical debt inventory
2. **Health Check score** — the security posture baseline
3. **Setup Audit Trail export** — six months of what has actually been changing
4. **Profile and permission set export** — the security model as it really is
5. **Active automation inventory** — Workflow Rules, Process Builders, Flows, Apex triggers
   per object; any object with more than two same-context automations is a risk
6. **Field usage analysis** — fields with no data are candidates for deprecation
7. **Report and dashboard usage** — `LastRunDate` on the Report object
8. **Integration inventory** — Connected Apps, Named Credentials, API-enabled users,
   login history for integration accounts
9. **Licence utilisation** — assigned versus actually logging in
10. **Storage usage** and trajectory

Produce a written findings document with severity ratings before proposing any changes.
Establishing the baseline in writing protects both parties when something later breaks.

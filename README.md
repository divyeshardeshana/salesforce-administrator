# salesforce-administrator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Agent-Skill-blue.svg)](https://docs.claude.com/en/docs/claude-code/skills)
[![Salesforce](https://img.shields.io/badge/Salesforce-Admin-00A1E0.svg)](https://www.salesforce.com/)

An [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) that turns Claude into
a declarative-first Salesforce administrator — data model design, permission architecture,
Flow automation, reporting, and release management.

Covers the clicks-not-code half of the platform: the work an admin does in Setup, not in an
IDE. Composes with developer-focused Salesforce skills rather than duplicating them.

Maintained by [Divyesh Ardeshana](https://github.com/divyeshardeshana).

---

## Contents

| Reference | Scope |
|---|---|
| [`data-model-security.md`](references/data-model-security.md) | Objects, relationships, field types, record types, Dynamic Forms, profiles, permission sets and groups, OWD, role hierarchy, sharing rules, restriction rules, guest user hardening, security audit checklist |
| [`sales-cloud.md`](references/sales-cloud.md) | Lead assignment and conversion field mapping, opportunity stages and Path, products and price books, quotes and the CPQ boundary, collaborative forecasting, territory management, campaign influence, activity capture trade-offs |
| [`service-cloud.md`](references/service-cloud.md) | Case model and support processes, assignment and escalation rules, queues, Omni-Channel routing and capacity, entitlements and milestones, Email-to-Case and Web-to-Case, Knowledge and data categories, console productivity, service metrics |
| [`flow-automation.md`](references/flow-automation.md) | Record-triggered Flow architecture, before-save vs after-save, bulkification, fault paths, screen flows, validation rules, approval processes, legacy automation migration, order of execution, debugging |
| [`lightning-experience-ux.md`](references/lightning-experience-ux.md) | Lightning apps and console navigation, record page assignment, Dynamic Forms and Dynamic Actions, quick actions and JavaScript button migration, list views and Kanban, Path, in-app guidance, compact layouts, mobile, Classic-to-Lightning sequencing |
| [`integration-admin.md`](references/integration-admin.md) | Integration user provisioning and the Modify All Data trap, Connected Apps and OAuth scope discipline, Named Credentials and External Credentials, External Services, Salesforce Connect and external objects, Platform Events and CDC, API limits and monitoring, middleware boundary, SSO, integration register |
| [`ai-and-data-cloud.md`](references/ai-and-data-cloud.md) | Licensing prerequisites and data residency, Einstein Trust Layer configuration and the prompt journey, Prompt Builder template types and authoring discipline, Agentforce topics/instructions/actions, predictive Einstein volume thresholds, Data Cloud ingestion and identity resolution, consumption cost realities |
| [`reporting-analytics.md`](references/reporting-analytics.md) | Custom report types, join semantics, report formats, cross filters, row-level and summary formulas, `PARENTGROUPVAL` / `PREVGROUPVAL`, bucket fields, dashboards, running-user behaviour, performance at scale, reporting snapshots |
| [`org-management-release.md`](references/org-management-release.md) | Sandbox strategy and post-refresh checklist, change sets vs DX, deployment runbook, Data Loader and upsert patterns, duplicate management, backup reality, user onboarding/offboarding, Health Check, Optimizer, seasonal releases, inherited-org assessment |

`SKILL.md` is the router — it carries the core workflow, the automation decision matrix,
MUST DO / MUST NOT DO constraints, and configuration patterns, then loads the relevant
reference on demand.

---

## Design principles

**Declarative first.** The skill works down an automation decision matrix — formula field,
then roll-up summary, then validation rule, then before-save Flow, then after-save Flow —
and only escalates to Apex with a written justification.

**Opinionated where the platform is ambiguous.** Permission sets over profiles. One Flow per
object per trigger point. Start OWD at Private. Deprecate fields, never delete them.

**Failure modes, not just happy paths.** Each reference ends with a troubleshooting table
mapping symptoms to their actual root causes.

**Bulk-safe by default.** Every automation pattern assumes 200-record batches, not
single-record demos.

**Security diagnosed in layers.** Object access, then field-level security, then record
access — in that order, every time.

---

## Installation

### Claude Code / Claude Cowork

```bash
git clone https://github.com/divyeshardeshana/salesforce-administrator.git
cp -r salesforce-administrator ~/.claude/skills/
```

Restart your session to pick it up.

### Project-scoped

```bash
mkdir -p .claude/skills
git clone https://github.com/divyeshardeshana/salesforce-administrator.git .claude/skills/salesforce-administrator
```

### skills.sh

```bash
npx skills add divyeshardeshana/salesforce-administrator
```

### Other agents

The skill follows the open `SKILL.md` standard and works anywhere skills are supported.

```bash
# Cursor / Windsurf / Cline
cp -r salesforce-administrator .agent/skills/

# GitHub Copilot
cp -r salesforce-administrator .github/skills/

# Gemini CLI
cp -r salesforce-administrator ~/.gemini/skills/
```

### Claude.ai

Settings → Capabilities → Skills → upload the folder as a `.zip`.

---

## Usage

The skill activates automatically on Salesforce administration topics. It also responds to
direct invocation.

**Security architecture**
```
Design a sharing model where reps see only their own Opportunities
but regional managers see everything in their region.

Why can a user see the Account but not the related Contacts?

Our org has 47 profiles. Plan a migration to permission set groups.
```

**Automation**
```
We have 30 Workflow Rules and 12 Process Builders on Account.
Plan the consolidation into Flow.

This Flow hits the SOQL limit on a 200-record import. Fix it.

Should this be a validation rule or a Flow Custom Error?
```

**Sales Cloud**
```
Which custom Lead fields will lose data on conversion?

Design the stage model and exit criteria for a 6-stage sales process.

Our forecast doesn't match the pipeline report. Diagnose it.

Do we need CPQ or will standard Quotes cover this?
```

**Service Cloud**
```
Set up entitlements so First Response is tracked against business hours.

Cases created manually aren't being assigned. Why?

Design Omni-Channel routing for three tiers with skills-based escalation.

Our SLA report shows 60-hour resolutions on 2-hour tickets. Fix it.
```

**Reporting**
```
Build a matrix report showing month-over-month pipeline growth by region.

Show me Accounts that have not bought anything this fiscal year.

This report times out. Diagnose it.
```

**Lightning UX**
```
A field vanished from the record page after a colleague's change. Why?

We have 40 page layouts to vary field visibility. Consolidate with Dynamic Forms.

Plan the Classic-to-Lightning migration, starting with JavaScript buttons.
```

**Integration**
```
Audit our Connected Apps for over-broad OAuth scopes.

Our integration broke after the sandbox refresh. What did we miss?

Direct integration or middleware for this? Three systems, bidirectional.
```

**AI**
```
What do we actually need provisioned before Agentforce will work?

Write a grounded Prompt Builder template for a case escalation summary.

Our Einstein Lead Scoring won't activate. Why?
```

**Org management**
```
Write a post-refresh checklist for a Full Copy sandbox.

We're inheriting an unfamiliar org. What's the assessment sequence?

Plan the deployment runbook for this release.
```

---

## Structure

```
salesforce-administrator/
├── SKILL.md                              # router: workflow, constraints, patterns
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── CHANGELOG.md
└── references/
    ├── data-model-security.md
    ├── flow-automation.md
    ├── sales-cloud.md
    ├── service-cloud.md
    ├── lightning-experience-ux.md
    ├── integration-admin.md
    ├── ai-and-data-cloud.md
    ├── reporting-analytics.md
    └── org-management-release.md
```

---

## Scope boundary

This skill stops at the declarative boundary. When a requirement genuinely needs code —
complex transactional logic, callouts with authentication flows, volumes beyond Flow's
practical ceiling, Apex Managed Sharing — it says so explicitly and hands off with a
written reason rather than building a sixty-element Flow to avoid the conversation.

Pair it with a Salesforce developer skill for the other half of the platform.

---

## Accuracy

Salesforce ships three releases a year. Governor limits, edition entitlements, and feature
availability drift.

Treat specific numeric limits in these references as correct-at-time-of-writing and verify
against [Salesforce Help](https://help.salesforce.com) for anything load-bearing. Issues
and PRs correcting drift are welcome and are the most useful contribution you can make.

Last reviewed against platform behaviour: **August 2026**.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Most useful contributions:

- Corrections where platform behaviour has changed
- Troubleshooting entries with the actual root cause, not the symptom
- Patterns that have survived contact with a real production org

---

## License

MIT — see [LICENSE](LICENSE).

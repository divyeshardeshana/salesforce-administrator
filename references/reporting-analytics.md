# Reporting & Analytics

---

## Report Types — The Foundation

The report type determines which objects, which fields, and which records are available.
Most "the report won't show what I need" problems are report type problems, not report
problems.

### Standard vs Custom Report Types

Standard report types are generated automatically for every object and relationship. They
are usually inner joins between parent and child, and you cannot control which fields are
exposed.

Create a **custom report type** when you need:

- Fields from a related object that the standard type omits
- An outer join — "Accounts *with or without* Opportunities"
- More than two related objects (up to four levels)
- Control over which fields appear in the field picker
- Lookup fields traversed to a related object's fields

### Object Relationship Semantics

This is the highest-leverage concept in report building.

```
A with B          INNER JOIN
                  Only records of A that HAVE at least one B
                  "Accounts with Opportunities" → excludes Accounts with none

A with or without B    LEFT OUTER JOIN
                       All A records, whether or not a B exists
                       "Accounts with or without Opportunities" → includes empty ones
```

The classic requirement "show me customers who have *not* bought anything this year"
requires `with or without`, then a filter of `Opportunity ID equals ""` — a cross filter or
a row-level check on the null.

A custom report type can chain up to four objects:

```
Account
  └── with or without Opportunities
        └── with or without Opportunity Products
              └── with Products
```

Each level after the first can be set to `with` or `with or without` independently.

### Building a Custom Report Type

```
Setup → Report Types → New Custom Report Type
  1. Primary Object:      Account
  2. Report Type Label:   "Accounts with Contract History"
  3. Store in Category:   Accounts & Contacts
  4. Deployment Status:   In Development  ← keep here until tested
  5. Define relationships (up to 3 more objects)
  6. Edit Layout:         drag fields into sections; add lookup fields via
                          "Add fields related via lookup"
  7. Set Deployment Status → Deployed when ready
```

Two things routinely missed:

- **"Add fields related via lookup"** at the bottom of the layout editor exposes fields
  from objects referenced by a lookup, without adding a full relationship level. This is
  how you get `Account.Owner.Manager.Email` into a report.
- **Deployment Status** controls visibility. A report type left "In Development" is
  invisible to non-admins, which produces the confusing report of "the report type
  disappeared for everyone else."

---

## Report Formats

| Format | Structure | Use for |
|---|---|---|
| **Tabular** | Flat rows, no grouping | Exports, simple lists, mailing lists |
| **Summary** | Grouped by one to three row groupings, with subtotals | Most business reports |
| **Matrix** | Row groupings and column groupings — a pivot table | Cross-tab analysis: stage by region, product by month |
| **Joined** | Multiple report blocks sharing a common grouping | Comparing different objects or filters side by side |

Constraints worth knowing:

- Tabular reports cannot be used in dashboards unless a row limit is set and a sort is
  defined (then they can feed a metric or gauge component)
- Summary and Matrix support charts and dashboard components natively
- Joined reports support up to five blocks and cannot be used for all dashboard component
  types
- Only Summary and Matrix support Summary Formulas and Row-Level Formulas

### Joined Reports

Underused and genuinely powerful. Each block can use a different report type as long as
the blocks share a common field to group by.

```
Joined Report grouped by: Account Name

Block 1 — Open Opportunities      (Opportunity report type)
Block 2 — Closed Cases            (Case report type)
Block 3 — Contracts Expiring      (Contract report type)
```

This produces a per-account view across three objects in one report — the "account health"
view that would otherwise require a dashboard or a custom page.

Cross-block summary formulas can reference values from other blocks, enabling ratios across
objects.

---

## Filters

### Filter Types

| Type | Behaviour |
|---|---|
| **Standard filters** | Date range and scope, fixed per report type (Show Me, Date Field, Range) |
| **Field filters** | Field, operator, value — the main mechanism |
| **Filter logic** | `(1 AND 2) OR 3` — up to 20 filters |
| **Cross filters** | Filter the primary object by the existence or absence of related records |
| **Row limit** | Tabular only — top N |

### Cross Filters

The tool for "with" and "without" logic without building a custom report type.

```
Accounts
  CROSS FILTER: without Opportunities
  → Accounts that have no Opportunity records at all

Accounts
  CROSS FILTER: with Opportunities
    Sub-filter: Stage equals "Closed Won"
    Sub-filter: Close Date this fiscal year
  → Accounts that bought something this year
```

Each cross filter supports up to five sub-filters, and a report supports three cross
filters. Cross filters do not add the related object's fields to the report — they only
filter. To *display* those fields, you need a report type with the relationship.

### Date Filtering

Relative date literals are far better than hard-coded dates because they keep working:

```
THIS_FISCAL_QUARTER    LAST_N_DAYS:30      NEXT_N_MONTHS:3
LAST_FISCAL_YEAR       THIS_MONTH          NEXT_FISCAL_QUARTER
LAST_90_DAYS           YESTERDAY           LAST_WEEK
```

Fiscal literals respect the org's fiscal year settings (Setup → Fiscal Year), which is why
a report showing the wrong quarter usually means the fiscal calendar is misconfigured
rather than the filter being wrong.

### Filter Logic and the OR Trap

Default logic is AND across all filters. Adding OR requires explicit filter logic:

```
Filters:
  1. Stage equals "Closed Won"
  2. Amount greater than 100000
  3. Type equals "New Business"

Filter Logic: 1 AND (2 OR 3)
```

Without explicit logic, all three are ANDed and the report returns far fewer rows than
expected.

### Locked and Editable Filters

Setting a filter to **locked** prevents users from changing it in the run page. Useful for
report folders shared broadly where the scope must not be widened.

---

## Formulas in Reports

### Row-Level Formulas

Calculate per row, like a formula field but scoped to one report. Available on Summary and
Matrix reports. One per report.

```
Row-Level Formula: Days_In_Stage
Formula:  TODAY() - DATEVALUE(Opportunity.LastStageChangeDate)
Format:   Number, 0 decimals
```

Advantage over a formula field: no schema change, no deployment, no FLS to manage. Use for
analysis that does not need to exist outside the report.

Limitation: one per report, and it cannot be used as a grouping.

### Summary Formulas

Calculate at the grouping level. Up to five per report.

```
Summary Formula: Win_Rate
Formula:  RowCount:SUM > 0
          ? (WON:SUM / RowCount:SUM) * 100
          : 0
Format:   Percent
Display:  At all summary levels
```

The `Display` setting controls whether the value appears at grand total only, at specific
grouping levels, or all — a frequent source of "my formula shows blank" confusion.

### PARENTGROUPVAL and PREVGROUPVAL

The two functions that make Salesforce reporting genuinely analytical.

**PARENTGROUPVAL** — compare a group to its parent group. Percentage-of-total calculations.

```
Matrix report: rows grouped by Region, then by Sales Rep
Summary Formula: Rep_Share_Of_Region

  AMOUNT:SUM / PARENTGROUPVAL(AMOUNT:SUM, Region)

Result: each rep's contribution as a fraction of their region's total
```

**PREVGROUPVAL** — compare a group to the previous group in the same grouping.
Period-over-period growth.

```
Matrix report: columns grouped by Close Date (Calendar Month)
Summary Formula: Month_Over_Month_Growth

  (AMOUNT:SUM - PREVGROUPVAL(AMOUNT:SUM, CLOSE_DATE))
  / PREVGROUPVAL(AMOUNT:SUM, CLOSE_DATE)

Format: Percent
```

Guard against division by zero in both — the first period has no previous value:

```
PREVGROUPVAL(AMOUNT:SUM, CLOSE_DATE) > 0
? (AMOUNT:SUM - PREVGROUPVAL(AMOUNT:SUM, CLOSE_DATE)) / PREVGROUPVAL(AMOUNT:SUM, CLOSE_DATE)
: 0
```

### Bucket Fields

Categorise values without a formula or a schema change. Up to five bucket fields per
report.

```
Bucket Field: Deal_Size
Source: Amount
  Small     →  < 10,000
  Medium    →  10,000 – 49,999
  Large     →  50,000 – 249,999
  Strategic →  >= 250,000
```

Works on numeric, picklist, and text fields. Picklist buckets are particularly useful for
collapsing a fifteen-value picklist into four reportable categories, and for grouping
inconsistent legacy values ("USA", "U.S.A.", "United States") into one bucket while a data
cleanup is pending.

Bucket fields can be used as groupings, which row-level formulas cannot.

---

## Dashboards

### Component Types

| Component | Best for | Notes |
|---|---|---|
| **Metric** | A single number | Add a label; the number alone is meaningless |
| **Gauge** | Progress to a target | Set breakpoints deliberately |
| **Chart — bar/column** | Comparison across categories | Horizontal bar for long category names |
| **Chart — line** | Trend over time | Requires a date grouping |
| **Chart — donut/funnel** | Composition or stage progression | Funnel only reads well with ordered stages |
| **Chart — scatter** | Two-measure correlation | Underused; good for effort vs outcome |
| **Table** | Detail rows with conditional highlighting | Limit to 10–20 rows on a dashboard |
| **Lightning table** | Multi-column table with more formatting | Supports up to 10 columns |

### Dashboard Running User

The single most important dashboard setting, and the most common source of "the numbers are
wrong for me."

| Setting | Behaviour |
|---|---|
| **Run as specified user** | Every viewer sees the specified user's data — identical numbers for everyone |
| **Run as logged-in user** (dynamic dashboard) | Each viewer sees their own data scope |
| **Let dashboard viewers choose** | Viewer picks; requires the "View My Team's Dashboards" style permissions |

Dynamic dashboards are limited per edition (typically 5 in Enterprise, 10 in Unlimited).
That constraint pushes most orgs toward "run as specified user" with a service account that
has wide visibility — which is correct for management dashboards and wrong for rep-facing
ones.

Practical pattern: build one dashboard per audience rather than one dashboard with a
compromise running user.

### Dashboard Filters

Up to three filters per dashboard, each with up to 50 values. Filters apply across all
components whose source report contains the filtered field.

```
Dashboard Filter: Region
  Field: Account.Region__c  (must exist in every source report)
  Values: EMEA, APAC, Americas
```

Components sourced from reports lacking that field ignore the filter silently — a common
cause of one component not responding to the filter.

### Subscriptions

Both reports and dashboards support scheduled email delivery.

- **Report subscriptions** — up to five per user; can include conditional logic ("only send
  if record count > 0"), which prevents the daily empty-report email that trains people to
  ignore it
- **Dashboard subscriptions** — delivered as an image; the running-user rules still apply

Set conditions on report subscriptions wherever the report is an exception report. An alert
that fires every day is not an alert.

---

## Performance at Scale

Reports on large objects time out. Diagnose in this order.

### 1. Filter on Indexed Fields

Standard indexed fields: `Id`, `Name`, `OwnerId`, `CreatedDate`, `SystemModstamp`,
`RecordTypeId`, master-detail and lookup relationship fields, and any field marked
**External ID** or **Unique**.

Custom fields can be indexed by Salesforce Support on request — a legitimate and
underused option for a heavily-filtered custom field.

A filter on a non-indexed text field across two million records forces a full table scan.

### 2. Avoid These in Filters

- Formula fields that traverse relationships — cannot be indexed
- `NOT EQUAL TO` and `!=` — these do not use indexes
- Leading wildcards (`*text`) — no index usage
- `OR` across different objects in a cross filter

### 3. Narrow the Date Range

The single most effective fix. A report defaulting to "All Time" on an object with a decade
of history is the usual culprit. Set a default date filter and let users widen it
deliberately.

### 4. Reduce Columns

Every column is retrieved for every row. Formula fields in columns are computed per row.
A 40-column report on 200,000 rows is eight million cell computations.

### 5. Skew

An object where one parent owns a disproportionate share of children (a "catch-all"
Account holding 300,000 Contacts) causes lock contention and slow queries. Detect it by
grouping child records by parent and looking at the top rows. The fix is data
restructuring, not report tuning.

### 6. Escalation Path

If a report still times out after the above, options are: a scheduled report delivered by
email (runs asynchronously with a longer timeout), CRM Analytics, or extracting to an
external warehouse. Do not chase sub-second performance on a report that fundamentally
needs to scan millions of rows.

---

## Report Governance

### Folder Architecture

```
Sales — Pipeline Management        → Sales role, Viewer
Sales — Forecasting                → Sales Management, Viewer
Sales — Rep Personal               → individual, Manager
Service — Case Volume              → Support role, Viewer
Executive — Board Reporting        → Exec group, Viewer
Admin — Data Quality               → Admin only, Manager
Admin — Archive                    → Admin only  (retired reports live here)
```

Folder access levels: **Viewer** (run and view), **Editor** (edit reports in the folder),
**Manager** (also manage sharing and delete).

Share folders with **public groups and roles**, never individual users. When someone leaves,
group membership changes; forty individual folder shares do not.

### Naming Convention

```
[Object] — [Purpose] — [Scope/Period]

Opportunity — Pipeline by Stage — Current Quarter
Case — First Response SLA Breaches — Last 30 Days
Account — Missing VAT Number — All
```

Sortable, self-describing, and immediately obvious in a folder listing.

### Cleanup Cadence

Every org accumulates report sprawl. Quarterly:

1. Identify reports not run in 12 months. The `Report` object exposes `LastRunDate`; query
   or export it
2. Move unused reports to the Archive folder rather than deleting — deletion breaks
   dashboards silently
3. Identify duplicate reports differing only by filter — consolidate with a filterable
   dashboard or editable filters
4. Verify every dashboard's source reports still exist and still return data
5. Review report subscriptions for departed users

### The Personal Folder Problem

Reports in "My Personal Custom Reports" are invisible to everyone else and disappear from
usefulness when the user leaves. Establish early that anything used more than twice belongs
in a shared folder.

---

## Beyond Standard Reports

### Reporting Snapshots

Capture report results on a schedule into a custom object, creating a historical time
series that standard reports cannot produce.

```
Source report:      Summary report — Pipeline by Stage
Target object:      Pipeline_Snapshot__c
Field mapping:      Stage → Stage__c, Amount:SUM → Total_Amount__c
Schedule:           Weekly, Monday 06:00
```

This is how you answer "what did the pipeline look like eight weeks ago" without a data
warehouse. The tradeoff is storage consumption and a target object that grows indefinitely
— plan a retention policy.

Limits: source report must be tabular or summary; up to 200 fields mapped; the running user
determines data visibility.

### Historical Trending

Native trend reporting on a small set of objects (Opportunity, Case, Forecasting Items, and
selected custom objects). Retains up to three months of daily snapshots for up to five
fields per object, configured in Setup → Historical Trending.

Lighter-weight than Reporting Snapshots but far more limited. Use snapshots when the
retention or field count exceeds these bounds.

### When to Move Beyond Native Reporting

Native reports are the right answer for most operational reporting. Consider CRM Analytics
or an external BI tool when:

- Data must be blended with non-Salesforce sources
- Row counts consistently exceed what reports can scan in a timeout window
- Users need genuinely interactive exploration rather than predefined groupings
- Complex multi-step calculations exceed five summary formulas
- Predictive scoring or trend forecasting is required

State the cost honestly when recommending: CRM Analytics is a separate licence, a separate
skill set, and a separate maintenance burden. Many requirements that appear to need it are
satisfied by a better report type and a matrix report.

---

## Common Reporting Problems

| Symptom | Cause and fix |
|---|---|
| Records missing from a report | Report type is an inner join — switch to `with or without`, or the running user lacks record access |
| A field is not in the field picker | Not exposed on the custom report type layout, or FLS hides it from the running user |
| Report shows different numbers to different users | Record-level sharing — this is usually correct behaviour, not a bug |
| Dashboard numbers differ from the report | Dashboard running user differs from the person viewing the report |
| Summary formula shows blank | The `Display` setting excludes that grouping level |
| Percentages do not total 100% | Rounding at each row; compute the percentage at the summary level instead |
| Report times out | See Performance at Scale above; start with the date range |
| Currency amounts look wrong | Multi-currency org — check the report's currency conversion and the dated exchange rate setup |
| Grouping by a date shows every individual day | Change the grouping granularity to Calendar Month/Quarter on the grouping settings |
| Duplicate rows appear | The report type joins a one-to-many relationship; each child produces a row. Group by the parent or use a cross filter instead |

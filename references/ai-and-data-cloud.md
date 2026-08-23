# AI & Data Cloud

The fastest-moving surface in Salesforce administration, and the one where the gap between
marketing material and configurable reality is widest. Verify every specific against current
Salesforce Help before committing to a design — this area changes materially each release.

---

## The Stack

```
Data Cloud            ← unified customer data, the grounding source
      │
Einstein Trust Layer  ← masking, grounding, toxicity scoring, audit; sits on every LLM call
      │
Prompt Builder        ← reusable, grounded prompt templates
      │
Agentforce            ← autonomous agents: Topics, Instructions, Actions
```

Each layer depends on the one below. An Agentforce agent with no grounding produces
plausible fiction; grounding requires either CRM merge fields or Data Cloud.

---

## Prerequisites and Licensing

Before promising anything, confirm what the org actually has.

| Component | Requirement |
|---|---|
| Einstein Generative AI | <cite index="79-1">Generally requires Enterprise or Unlimited edition with an Agentforce or Einstein GPT add-on</cite> |
| Agentforce | <cite index="83-1">Data Cloud provisioned with at least one data space, Einstein Generative AI enabled for the region, and Agentforce as an add-on SKU on Enterprise/Unlimited — included in some Industries clouds</cite> |
| Pricing model | <cite index="80-1">Consumption-based rather than per-user, purchased through Flex Credits</cite> |

This is not a feature you switch on. It is a procurement conversation followed by a
provisioning wait. Set that expectation before scoping.

---

## Enabling Einstein

```
Setup → Einstein Setup → Turn on Einstein
```

<cite index="80-1">Salesforce confirms the data residency region — which matters for compliance and Data Cloud processing — and requires acceptance of the Einstein Generative AI terms of service. If Einstein Setup returns no results, Data Cloud may not be fully provisioned, or the region has not yet received the update.</cite>

<cite index="80-1">Once Einstein is enabled, the Einstein Trust Layer activates automatically.</cite>

Data residency is a real decision, not a formality. For UAE, GCC, and EU clients with data
localisation obligations, confirm the processing region before enabling and record the
answer — the security team will ask.

---

## Einstein Trust Layer

<cite index="81-1">A collection of features, processes, and policies designed to safeguard data privacy, improve AI accuracy, and promote responsible AI use. Its capabilities apply only to generative AI and Agentforce features.</cite>

### The Prompt Journey

```
1. Prompt authored           Prompt Builder, invoked from Apex, Flow, or an Agent Action
2. Secure data retrieval     grounding with CRM data the executing user can access
3. Data masking              PII replaced with placeholders before leaving the platform
4. Zero-retention LLM call   the provider does not retain the prompt
5. Toxicity scoring          response checked for harmful or biased content
6. Unmasking                 placeholders restored in the response
7. Audit logging             the interaction recorded
```

<cite index="81-1">Grounding can use merge fields with CRM data — record fields, flows, Apex, Data Cloud DMOs, and related lists. Secure data retrieval means the prompt is grounded only with data the executing user can access.</cite>

That last point is the one to emphasise with security stakeholders: the Trust Layer respects
the running user's field-level security and record access. An agent cannot surface data the
user could not see themselves — provided the grounding is configured through supported
mechanisms rather than an Apex class running without sharing.

### What to Configure

```
Setup → Einstein Trust Layer
  Entity masking:     which PII types are masked (names, emails, phone, account numbers)
  Toxicity threshold: the score above which a response is blocked
  Audit logging:      retention and access
  Data Cloud storage: where prompt/response audit records live
```

Read every default before go-live. <cite index="85-1">List the entity types currently masked and the toxicity threshold currently set</cite> — the defaults may not match a regulated client's expectations, and "Salesforce handles it" is not an audit answer.

### Trust Layer Volume in Agentforce

Worth understanding for cost and debugging: <cite index="85-1">a typical Agentforce request runs the Trust Layer multiple times in one user turn — once for topic-routing classification, once per agent action invoking a prompt template (often two or three), once for the retrieval call grounding against Data Cloud knowledge, and once for the final synthesis prompt. Each call is its own audit record, and each applies masking, scoring, and unmasking independently.</cite>

This is why a single user question can consume several times the credits a naive estimate
suggests, and why debugging a conversation means reading multiple audit records rather than
one.

---

## Prompt Builder

<cite index="84-1">The declarative studio where reusable AI prompts are authored, grounded, tested, and deployed.</cite> Its role has expanded: <cite index="84-1">every Agentforce Agent Action producing unstructured output — a summary, recommendation, draft, or classification — is a prompt template under the hood.</cite>

### Template Types

| Type | Surfaces at | Typical use |
|---|---|---|
| **Field Generation** | A field on a record page | Auto-drafted description, categorisation, summary field |
| **Email Generation** | The email composer | Drafted reply grounded in the record |
| **Flex** | Invoked from Flow, Apex, an Agent Action, or the API | Everything else, including agent actions |
| **Record Summary** | Record page component | Case or Account summary panel |

### Anatomy of a Template

```
Prompt Template: Case_Summary_For_Escalation
  Type:        Flex
  Object:      Case
  Inputs:      {!Input:Case}

  Grounding (merge fields):
    {!Input:Case.Subject}
    {!Input:Case.Description}
    {!Input:Case.Account.Name}
    Related list: Case Comments (last 10)
    Flow: Get_Related_Knowledge_Articles

  Instructions:
    "Summarise this support case for a manager receiving an escalation.
     Cover: the customer's reported problem, what has been tried, and the
     current blocker. Three sentences maximum. Do not speculate about
     causes not stated in the case. If information is missing, say so
     explicitly rather than inferring."

  Model:       select from available models
  Test:        preview against a real record before activating
```

### Authoring Guidance

- **Ground everything.** A prompt without merge fields is a generic LLM call with a
  Salesforce bill attached.
- **Constrain the output shape.** Specify length, format, and tone. "Three sentences
  maximum" is enforceable in a way "be concise" is not.
- **Forbid inference explicitly.** "Do not speculate" and "if information is missing, say
  so" measurably reduce fabrication in record summaries.
- **Test against edge cases**, not the tidy demo record — an empty description, a Case with
  200 comments, non-English text.
- **Version deliberately.** Templates deploy through change sets and DX like other metadata.

### Common Failures

<cite index="80-1">Watch for the missing permission set error — confirm builders have the Einstein Prompt Template Manager permission.</cite>

Other recurring problems: grounding fields the running user cannot see (the merge resolves
empty, silently); related-list grounding exceeding the context window on high-volume
records; and templates that work in preview as an admin but fail for standard users because
of FLS on a grounded field.

---

## Agentforce

Autonomous agents that reason over a request, select a topic, and execute actions.

### The TIA Model

```
Agent
 └── Topic                    a bounded area of responsibility
      ├── Classification      when this topic should handle a request
      ├── Scope               what is in and out of bounds
      ├── Instructions        how to behave within the topic
      └── Actions             what it can actually do
            ├── Flow
            ├── Prompt Template
            ├── Apex
            └── Standard action
```

### Build Sequence

```
1. Setup → Agentforce Setup → create an agent
2. Define the agent's role, company description, and tone
3. Create Topics — one per bounded responsibility, not one per feature
4. Per topic: classification description, scope, instructions
5. Attach Actions — Flows are the declarative path
6. Configure the agent user and its permission set
7. Test in Agent Builder's test mode
8. Deploy to a channel: Experience Cloud, Service Console, Slack, messaging
```

### Configuration Discipline

<cite index="78-1">Apply least privilege when setting permissions, ensuring the agent only accesses fields essential for its role. Enable action and output logging to support compliance monitoring. Use Agent Builder's test mode for typical, edge-case, and restricted scenarios, refining prompts and permissions accordingly.</cite>

<cite index="80-1">Launch one agent with a focused topic before scaling.</cite> The failure mode is
building an agent with twelve topics that classifies unpredictably. Two well-bounded topics
that work beat twelve that argue with each other.

The **agent user** is a real Salesforce user with a profile, permission sets, and record
access. Everything in the data-model-security reference applies to it. An agent granted
`Modify All Data` is an agent that can do anything to anything.

### Data Quality Dependency

<cite index="78-1">Cleanse key CRM records, since Agentforce performance relies heavily on data precision and consistency.</cite>

This is the honest blocker for most orgs. An agent grounded in a CRM with inconsistent
picklists, duplicate Accounts, and half-populated fields produces confidently wrong answers.
The data-quality work in the org-management reference is a prerequisite, not a parallel
track.

### Testing and Governance

<cite index="78-1">Protect production data by using a sandbox with masked data for updates and testing. Involve Salesforce administrators early, and clearly document the agent's responsibilities and the KPIs it will be measured against.</cite>

Define before launch:

- Which requests the agent handles and which escalate to a human
- What the agent may write, not just read
- How a bad response is reported and corrected
- Which KPIs judge success — deflection rate, escalation rate, CSAT on agent-handled
  conversations
- A kill switch: how to disable the agent quickly if it misbehaves

---

## Predictive Einstein Features

Distinct from generative AI, older, and often available without the Agentforce SKU.
<cite index="80-1">Einstein AI focuses on predictions, recommendations, content generation, and insights, while Agentforce enables autonomous agents that reason, decide, and perform multi-step actions.</cite>

| Feature | Cloud | Requires |
|---|---|---|
| Einstein Lead Scoring | Sales | Sufficient converted-Lead history |
| Einstein Opportunity Scoring | Sales | Closed-Won/Lost volume |
| Einstein Forecasting | Sales | Forecasting enabled, historical data |
| Einstein Account Insights | Sales | — |
| Einstein Case Classification | Service | Historical Cases with consistent field values |
| Einstein Case Routing | Service | Case Classification |
| Einstein Article Recommendations | Service | Knowledge with resolution history |
| Einstein Bots | Service | Superseded by Agentforce for new builds |
| Einstein Reply Recommendations | Service | Conversation history |

All of these need **historical volume**. Salesforce publishes minimum record thresholds per
feature; below them the models do not build. A new org cannot enable Einstein Lead Scoring
usefully on day one — this is the most common disappointment.

They also need **consistency**. Case Classification trained on a Product picklist where
agents pick arbitrarily will predict arbitrarily.

---

## Data Cloud

Unifies customer data from Salesforce and external sources into a queryable, real-time
profile. The grounding substrate for AI, and increasingly a prerequisite rather than an
option.

### Core Concepts

| Concept | Meaning |
|---|---|
| **Data Space** | A logical partition — brand, region, or business unit |
| **Data Stream** | An ingestion pipeline from a source |
| **Data Lake Object (DLO)** | Raw ingested data |
| **Data Model Object (DMO)** | Mapped to the canonical model, queryable |
| **Identity Resolution** | Ruleset matching records into a Unified Profile |
| **Calculated Insight** | Aggregation computed across unified data |
| **Segment** | An audience built from profiles and insights |
| **Activation** | Pushing a segment to a destination |

### Ingestion Paths

```
Salesforce CRM Connector    standard and custom objects, near real time
Ingestion API               push from any system
Amazon S3 / Cloud Storage   batch files
Marketing Cloud             engagement data
Zero-copy federation        query the warehouse in place (Snowflake, BigQuery, Databricks,
                            Redshift) without ingesting
```

**Zero-copy** is the significant recent capability: query data in the warehouse without
duplicating it, avoiding both the storage cost and the sync lag. Check current connector
support before designing around it.

### Identity Resolution

The step that determines whether Data Cloud produces a unified customer view or an
expensive duplicate pile.

```
Ruleset: Consumer_Identity
  Match rules (evaluated in order):
    1. Exact — Email
    2. Exact — Phone + Last Name
    3. Fuzzy — First Name + Last Name + Postal Code
  Reconciliation rules:
    Most recent value wins  |  Source priority  |  Most frequent
```

Test rulesets against a sample before running at scale. Over-loose matching merges distinct
people; over-tight matching leaves the same person as four profiles. Neither is recoverable
without rework.

### Consumption Model

Data Cloud is metered on credits consumed by ingestion, processing, queries, segmentation,
and activation. Costs scale with data volume and query frequency, not user count.

Practical implication: a poorly designed Calculated Insight recomputing hourly over the full
profile set is expensive in a way nothing else in Salesforce is. Review consumption in Setup
→ Data Cloud → Digital Wallet regularly, and treat a spike as an incident.

---

## Admin Responsibilities Checklist

Before any AI feature reaches production:

- [ ] Licensing and provisioning confirmed in writing, including data residency region
- [ ] Trust Layer settings reviewed — masking entities, toxicity threshold, audit retention
- [ ] Agent or prompt executing user has least-privilege access, documented
- [ ] Grounding sources verified against the running user's FLS and record access
- [ ] Prompt templates tested against edge cases, not demo records
- [ ] Data quality baseline established for the objects being grounded
- [ ] Escalation path defined for requests the AI should not handle
- [ ] Logging enabled and someone assigned to review it
- [ ] Consumption monitoring in place with an alert threshold
- [ ] Kill switch documented and tested
- [ ] KPIs agreed with the business before launch
- [ ] Sandbox testing completed with masked data

---

## Setting Expectations

Three conversations worth having early, in plain terms:

**Cost is consumption-based and hard to predict.** Per-conversation pricing multiplied by
several Trust Layer calls per turn means a pilot's cost does not scale linearly to
production volume. Model it with real conversation counts before committing.

**Data quality is the ceiling.** No amount of prompt engineering compensates for
inconsistent picklists and duplicate records. If the org has not done the data-quality work,
that is the project — the AI comes after.

**These features change every release.** A configuration validated in one release may behave
differently in the next. Build the review of AI features into the seasonal release routine
described in the org-management reference, and re-test prompts after every upgrade.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Einstein Setup returns nothing | Data Cloud not provisioned, or the region has not received the update |
| Prompt Builder unavailable to a builder | Missing the Einstein Prompt Template Manager permission |
| Grounded merge field resolves empty | The executing user lacks FLS or record access to that field |
| Response is generic and unhelpful | No grounding — the template has instructions but no merge fields |
| Agent classifies requests to the wrong topic | Topic classification descriptions overlap; tighten scope statements |
| Agent takes an action it should not | The action is attached to the topic, or the agent user's permissions are too broad |
| Response blocked unexpectedly | Toxicity threshold too aggressive for the content domain |
| Credit consumption far above estimate | Multiple Trust Layer calls per turn; audit the action count per topic |
| Einstein predictive feature will not activate | Insufficient historical record volume below Salesforce's minimum |
| Predictions are unreliable | Training data is inconsistent — the underlying field values vary arbitrarily |
| Identity Resolution merged distinct customers | Match rules too loose; tighten and re-run against a sample |
| Data Cloud costs spiked | A Calculated Insight or segment recomputing too frequently over too much data |
| AI feature behaves differently after a release | Expected — re-test prompts and agents each seasonal release |

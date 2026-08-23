# Integration Administration

The admin's half of integration work: authentication, credentials, connected apps, external
data, and the operational discipline that keeps integrations from becoming the org's
weakest security link. Code-level integration patterns belong to a developer skill; this
covers what is configured in Setup.

---

## Integration Users

Every integration should authenticate as a dedicated user, never as a named human and never
as a System Administrator.

### Provisioning

```
1. Create a user
     Name:        "Integration - NetSuite"      (name the system, not a person)
     Username:    integration.netsuite@yourorg.com.int
     Email:       a monitored distribution list, not an individual
     Profile:     a minimal profile — often a cloned Minimum Access profile
2. Permission set: least privilege for the objects and fields the integration touches
3. System permission: "API Enabled" — required; nothing else by default
4. Login IP Ranges on the profile — restrict to the integration's source addresses
5. "Password Never Expires" — defensible here, paired with IP restriction and a documented
   rotation schedule
6. Disable "Lightning Experience User" if the integration never uses the UI
7. Document: which system, which objects, which direction, who owns it, who to call
```

Salesforce offers **Salesforce Integration** licences — API-only, cheaper than a full
licence, and unable to log into the UI. Use them where available; they remove an entire
class of risk.

### The Modify All Data Trap

An integration that needs to write to every Account should receive a permission set with
`View All` and `Modify All` **on Account only**. It should not receive the org-wide
`Modify All Data` system permission.

This distinction is the most common finding in a Salesforce security audit, and the most
common cause of an integration compromise becoming a full-org compromise.

---

## Connected Apps

A Connected App is the registration for an external application that authenticates into
Salesforce via OAuth or SAML.

### Configuration

```
Setup → App Manager → New Connected App
  API (Enable OAuth Settings)
    Callback URL
    Selected OAuth Scopes
    Require Secret for Web Server Flow:      yes
    Require Secret for Refresh Token Flow:   yes
    Enable Client Credentials Flow:          only if genuinely needed
    Require Proof Key for Code Exchange (PKCE): yes for public clients
```

### OAuth Scopes — Choose Narrowly

| Scope | Grants |
|---|---|
| `api` | Access to data via the API, bounded by the user's own permissions |
| `refresh_token` / `offline_access` | Long-lived access without re-authentication |
| `openid`, `profile`, `email` | Identity information only |
| `web` | Access to the UI as the user |
| `full` | **Everything the user can do.** Almost never correct |
| `chatter_api`, `content`, `custom_permissions` | Scoped to those areas |

`full` is the default many integration vendors request. Push back: `api` plus
`refresh_token` covers the overwhelming majority of legitimate cases.

### Policies — Set After Creation

Connected App **Policies** are edited via Manage after the app exists:

```
Setup → Connected Apps → Manage → Edit Policies
  Permitted Users:
      All users may self-authorize        ← avoid
      Admin approved users are pre-authorized  ← correct for integrations
  IP Relaxation:
      Enforce IP restrictions             ← correct
      Relax IP restrictions               ← only with justification
  Refresh Token Policy:
      Expire refresh token after N days / on inactivity
  Session Policy: timeout, high assurance requirement
```

With "Admin approved users are pre-authorized", assign access via a profile or permission
set. This is how you stop an arbitrary user authorising a third-party app against their own
credentials.

### Auditing Connected Apps

```
Setup → Connected Apps OAuth Usage
  → shows every app with an active token, user count, and last use
```

Review quarterly. Revoke anything unrecognised. Users installing browser extensions and
productivity tools that authenticate into Salesforce is a real and continuous exposure —
this page is where you find them.

Per-user tokens are visible on the user record under **OAuth Connected Apps**, and can be
revoked individually.

---

## Named Credentials

The correct place for outbound endpoint URLs and authentication. Replaces hard-coded URLs
and stored credentials in custom settings.

### Structure

Modern Named Credentials split into two objects:

```
External Credential            ← the authentication
  Authentication Protocol:  OAuth 2.0 | Basic | JWT | AWS Signature v4 | Custom
  Principals:               Named Principal (one identity for all users)
                            or Per User Principal
  Permission set grants access to a principal

Named Credential               ← the endpoint
  URL:                      https://api.vendor.com
  External Credential:      linked
  Callout options:          Generate Authorization Header, Allow Formulas in HTTP Header/Body
```

### Why It Matters to an Admin

- Credentials are stored encrypted and are **not visible** in the UI after entry
- Access is granted via permission set to the External Credential Principal — auditable
- Endpoint changes between sandbox and production are a configuration change, not a code
  change
- Apex and Flow reference `callout:MyNamedCredential`, so the URL never appears in code

**Named Principal** — all users share one identity to the external system. Correct for
system-to-system integration.

**Per User Principal** — each user authenticates individually. Correct when the external
system enforces its own per-user permissions.

### Post-Refresh Requirement

Named Credentials copy to a sandbox with their configuration but the **secrets do not
transfer**. After every sandbox refresh, re-enter credentials and repoint URLs to sandbox
endpoints. Put this on the post-refresh checklist — an integration silently pointing at
production from a sandbox is a genuine incident risk.

---

## External Services

Registers an OpenAPI-described REST API so its operations become invocable actions in Flow
— integration without Apex.

```
Setup → External Services → Add an External Service
  From API Specification:  paste or reference an OpenAPI 2.0/3.0 schema
  Named Credential:        select
  Select operations to import
  → each operation becomes an available Action in Flow
```

Practical constraints:

- The schema must be valid OpenAPI and reasonably simple; deeply nested or polymorphic
  schemas often fail to import
- Complex request bodies become Apex-defined types that are awkward to build in Flow
- No native retry; wrap the action in a Flow fault path and a retry mechanism
- Callouts cannot occur after DML in the same transaction — use an asynchronous Flow path

For a straightforward REST endpoint with a clean schema, this genuinely removes the need
for a developer. For anything with authentication complexity, pagination, or error
semantics, it does not.

---

## Salesforce Connect and External Objects

Surfaces data that lives in another system as though it were a Salesforce object, without
copying it.

```
Setup → External Data Sources → New
  Type:  OData 2.0 / OData 4.0 / Salesforce Connect: Cross-Org / Custom Apex Adapter
  URL, Named Credential / authentication
  Options: Writable External Objects, High Data Volume
  → Validate and Sync  → creates External Objects (__x)
```

### What Works and What Does Not

| Capability | External Objects |
|---|---|
| List views, record pages, search | Yes |
| Reports | Yes, with limitations |
| Relationships to standard objects | Via External Lookup and Indirect Lookup |
| Roll-up summaries | No |
| Validation rules, triggers, workflow | No |
| Sharing model | Inherited from the external system's response — no Salesforce sharing |
| Data storage consumed | None |
| Performance | Every query is a live callout — latency is the external system's |

Licensing: Salesforce Connect is a paid add-on, priced per data source.

**When it fits:** large historical datasets that must be viewable but not stored — an ERP
order history, a data warehouse archive. **When it does not:** data that needs automation,
validation, or offline availability. In that case, replicate it into a custom object with a
scheduled integration.

---

## Platform Events

An event-driven message bus. Publishers emit; subscribers react. Decouples systems and
avoids the polling pattern.

### Admin Configuration

```
Setup → Platform Events → New Platform Event
  Label:         Order Shipped
  API Name:      Order_Shipped__e
  Publish Behavior:  Publish After Commit  |  Publish Immediately
  Custom fields: the payload
```

**Publish After Commit** — the event fires only if the transaction succeeds. Correct for
most cases.
**Publish Immediately** — fires regardless of rollback. Use for logging and audit trails
that must record the attempt.

### Publishing and Subscribing Declaratively

```
Publish:    Flow → Create Records → Order_Shipped__e
Subscribe:  Platform Event-Triggered Flow, object = Order_Shipped__e
```

An external system subscribes via the Pub/Sub API or CometD streaming.

### Operational Notes

- Events are retained for **72 hours** in the event bus; a subscriber offline longer misses
  them
- Delivery and publish allocations are metered per 24 hours — check Setup → Platform Event
  Allocations
- Platform Event-Triggered Flows run as the **Automated Process** user, so sharing and FLS
  behave differently. Grant that user access explicitly where needed
- Errors in a subscriber Flow do not retry by default; build a fault path that logs

**Change Data Capture** is a related mechanism: Salesforce publishes record change events
automatically for selected objects (Setup → Change Data Capture). Useful for keeping an
external system in sync without a polling integration.

---

## Outbound Messages and Legacy Mechanisms

**Outbound Messages** (Setup → Workflow Actions → Outbound Messages) send a SOAP message to
an endpoint on a workflow or approval action. Still functional, still used in older orgs.

Limitations that argue against new use: SOAP only, no authentication beyond the session ID,
a fixed retry schedule over 24 hours, and no easy observability. Platform Events or a Flow
HTTP callout via External Services are better choices for anything new.

Outbound Messages can be triggered from a Flow via the legacy action, which keeps them
alive in migrated orgs. Note them in the integration register.

---

## API Limits and Monitoring

### Daily API Request Limit

Calculated from licence counts, with an edition floor. Setup → Company Information shows the
allocation; Setup → System Overview shows consumption in the trailing 24 hours.

Exceeding it blocks **all** API traffic, not just the offending integration. A retry loop in
one integration takes down every other integration and any connected mobile app.

Protective measures:

```
1. Setup → API Usage Notifications → New
     Notify a named admin at 70% and 90% of the limit
2. Report on the ApiEvent / API Usage in Event Monitoring if licensed
3. Require every integration to use the Bulk API for volume operations —
   one Bulk API job of 10,000 records costs a handful of calls, not 10,000
4. Audit for polling integrations checking every minute for changes that
   happen twice a day — replace with Change Data Capture or Platform Events
```

### Other Limits to Watch

| Limit | Note |
|---|---|
| Concurrent long-running requests | 10 synchronous requests over 20 seconds; exceeding queues then fails |
| Bulk API batches | 15,000 batches per 24 hours |
| Streaming events delivered | Metered per day; separate from API calls |
| Callout timeout | 120 seconds maximum, 10 seconds default |
| Callouts per transaction | 100 |

---

## Middleware Boundary

When to put an integration platform (MuleSoft, Boomi, Workato, n8n, Azure Logic Apps)
between Salesforce and another system rather than connecting directly.

**Direct is fine when:**
- Two systems, one direction, simple field mapping
- Volume is low and latency is uncritical
- The other system exposes a clean REST API with stable authentication

**Use middleware when:**
- Three or more systems need the same data
- Transformation, enrichment, or aggregation is required between systems
- Retry, dead-letter queues, and replay matter
- The other system's API is unstable, rate-limited, or requires orchestration
- Audit and observability across the integration estate is a requirement

The honest framing for a client: middleware adds cost and a second platform to maintain.
It pays for itself at the point where point-to-point integrations start multiplying — the
classic threshold is the fourth system.

---

## Single Sign-On and Identity

### SAML SSO

```
Setup → Single Sign-On Settings → Enable SAML → New
  Issuer, Entity ID
  Identity Provider Certificate (upload)
  SAML Identity Type:      Federation ID  |  Username  |  User ID
  SAML Identity Location:  Subject  |  Attribute
  Identity Provider Login URL, Logout URL
  Just-in-Time provisioning (optional)
```

**Federation ID** is the recommended identity type — it decouples the Salesforce username
from the IdP identifier, so a username change does not break SSO. Populate `FederationId`
on every user.

Always keep at least one administrator able to log in with a username and password, exempt
from SSO enforcement. Locking every admin out behind a broken IdP is a recoverable but
extremely unpleasant situation.

**My Domain** is a prerequisite for SSO and is now mandatory for all orgs.

### Just-in-Time Provisioning

Creates or updates the Salesforce user from SAML attributes on login. Removes manual
provisioning but requires the IdP to send profile, role, and permission set attributes
correctly. Test thoroughly — a misconfigured JIT assertion can silently change a user's
profile on every login.

### Delegated Authentication and Social Sign-On

Both exist; both are niche. Delegated authentication sends credentials to an external
service for verification and is largely superseded by SAML. Social sign-on via Auth
Providers is relevant for Experience Cloud, rarely for internal users.

---

## Integration Register

The artefact most orgs lack. Maintain a table covering every connection:

| Field | Example |
|---|---|
| System | NetSuite |
| Direction | Bidirectional |
| Objects touched | Account, Opportunity, Product2, custom Invoice__c |
| Mechanism | REST via middleware (Boomi) |
| Authentication | Connected App, OAuth, integration user |
| Integration user | Integration - NetSuite |
| Frequency | Real-time out, hourly batch in |
| Approximate API calls/day | 4,000 |
| Business owner | Finance |
| Technical contact | Vendor support + internal owner |
| Failure impact | Invoices not created; revenue recognition delayed |
| Runbook location | Link |

Without this, nobody can answer "what breaks if we change this field" — which is why field
deprecation stalls in mature orgs.

---

## Integration Monitoring

Signals available without Event Monitoring:

- **Setup → System Overview** — API usage in the trailing 24 hours
- **Setup → API Usage Notifications** — threshold alerts
- **LoginHistory** — report on integration user logins; a gap indicates a stopped
  integration
- **Setup → Apex Jobs** and **Scheduled Jobs** — failures in scheduled integration code
- **Debug Logs** on the integration user — expensive, use for diagnosis not monitoring
- **A custom `Integration_Log__c` object** written by inbound Flows and Apex — the most
  practical option, because it is reportable and can be dashboarded

With Event Monitoring licensed, the `ApiEvent`, `ApiTotalUsage`, and `LoginEvent` log types
give per-call detail.

Build an integration health dashboard: last successful sync per system, error counts by
day, API consumption trend. Discovering a failed integration from a business user three
weeks later is the default outcome without one.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Integration fails after a sandbox refresh | Named Credential secrets do not copy; re-enter them and repoint URLs |
| All integrations stopped simultaneously | Daily API limit exhausted — find the runaway caller |
| `INVALID_SESSION_ID` | Session expired, refresh token revoked or expired per the Connected App policy |
| `INSUFFICIENT_ACCESS_OR_READONLY` | Integration user lacks object, field, or record-level access |
| `REQUEST_LIMIT_EXCEEDED` | Daily API allocation, or concurrent long-running request limit |
| `UNABLE_TO_LOCK_ROW` | Concurrent updates to the same parent record; reduce batch size or serialise |
| Callout fails after a DML in the same transaction | Not permitted — move the callout to an asynchronous Flow path or a queueable |
| External object returns nothing | External system unreachable, credential expired, or the OData query is unsupported |
| Platform Event subscriber did not fire | Event older than 72 hours, delivery allocation exhausted, or the Automated Process user lacks access |
| Connected App authorises unexpected users | "All users may self-authorize" — change to admin-approved |
| SSO users locked out | IdP certificate expired, or Federation ID missing on the user |
| A field change broke an unknown integration | No integration register — this is the cost of not maintaining one |
| Outbound message not delivered | Endpoint unreachable; check Setup → Outbound Messages delivery status queue |

# Changelog

All notable changes to this skill are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.0] — 2026-08-23

### Added

- `references/lightning-experience-ux.md` — the four-surface page layer model, Lightning
  apps and console vs standard navigation, utility bar, record page assignment by
  App/Record Type/Profile, component visibility filters, Dynamic Forms migration and
  constraints, Dynamic Actions, quick actions and predefined field values, JavaScript
  button migration paths, list views and Kanban, Path, in-app guidance, compact layouts,
  home and app pages, mobile and Briefcase Builder, Classic-to-Lightning sequencing,
  troubleshooting table
- `references/integration-admin.md` — integration user provisioning and least privilege,
  the object-scoped vs org-wide Modify All Data distinction, Connected Apps with OAuth
  scope guidance and policy configuration, OAuth usage auditing, Named Credentials and
  External Credentials with the post-refresh secret caveat, External Services, Salesforce
  Connect and external object limitations, Platform Events and Change Data Capture, API
  limits and notification thresholds, middleware decision boundary, SAML SSO and JIT
  provisioning, the integration register, monitoring signals, troubleshooting table
- `references/ai-and-data-cloud.md` — the stack dependency chain, licensing and
  provisioning prerequisites, data residency, Einstein Trust Layer prompt journey and
  configuration, Trust Layer call volume per Agentforce turn, Prompt Builder template types
  and authoring discipline, Agentforce TIA model and governance, predictive Einstein
  historical volume thresholds, Data Cloud concepts, ingestion paths including zero-copy,
  identity resolution, consumption model, admin checklist, expectation-setting guidance,
  troubleshooting table

### Changed

- `SKILL.md` — routing table extended to nine references; `description` and `triggers`
  updated; version bumped to 1.2.0
- `README.md` — contents table, structure diagram, and three new usage blocks

## [1.1.0] — 2026-08-23

### Added

- `references/sales-cloud.md` — lead management and assignment rules, lead conversion with
  field-mapping data-loss warning, sales processes and stage design, Opportunity Path,
  teams and splits, contact roles, product and price book object model, product schedules,
  quotes and the standard-vs-CPQ boundary, collaborative forecasting and quota loading,
  custom fiscal year irreversibility, Enterprise Territory Management, campaigns and
  influence attribution, activity capture trade-offs including Einstein Activity Capture
  reportability, sales automation patterns, sales metrics groundwork, troubleshooting table
- `references/service-cloud.md` — Case object anatomy and support processes, assignment
  rules, queues, Omni-Channel routing models and capacity, skills-based routing, escalation
  rules against business hours, entitlements and milestone lifecycle, Email-to-Case
  threading, Web-to-Case limits, auto-response loops, Lightning Knowledge and data
  categories, service console, macros and quick text, CTI and Service Cloud Voice, case
  automation patterns including milestone pausing, service metrics with business-hours
  caveat, troubleshooting table

### Changed

- `SKILL.md` — routing table extended to six references; `description` and `triggers`
  updated to cover Sales Cloud and Service Cloud terminology; version bumped to 1.1.0
- `README.md` — contents table, structure diagram, and usage examples updated

## [1.0.0] — 2026-08-23

Initial release.

### Added

- `SKILL.md` — core workflow, automation decision matrix, MUST DO / MUST NOT DO
  constraints, configuration patterns, quality gates, developer escalation criteria
- `references/data-model-security.md` — object and relationship design, field type
  selection, record types, Dynamic Forms, the three security layers, OWD and sharing
  architecture, permission set groups and muting, profile minimisation migration, Shield
  encryption trade-offs, guest user hardening, quarterly security audit checklist
- `references/flow-automation.md` — Flow type selection, before-save vs after-save
  architecture, one-flow-per-object-per-context pattern, entry criteria, recursion control,
  fault paths, bulkification, governor limits, screen flow design, validation rule patterns
  with bypass architecture, approval processes, legacy automation migration and cutover,
  order of execution, debugging and Flow Tests, governance conventions
- `references/reporting-analytics.md` — custom report types and join semantics, report
  formats including joined reports, cross filters, row-level and summary formulas,
  `PARENTGROUPVAL` and `PREVGROUPVAL`, bucket fields, dashboard running-user behaviour,
  performance at scale, report governance and folder architecture, reporting snapshots and
  historical trending
- `references/org-management-release.md` — sandbox types and environment strategy, post-
  refresh checklist, change sets vs Salesforce DX, validate-then-quick-deploy pattern,
  unlocked packages, deployment runbook, Data Loader operations and upsert patterns,
  automation bypass for loads, duplicate management, data quality monitoring, backup
  reality, user onboarding and offboarding, licence management, Health Check, Optimizer,
  Setup Audit Trail, storage and org limits, seasonal release routine, org documentation
  and inherited-org assessment

### Notes

- Reflects the position that Workflow Rules and Process Builder reached **end of support**
  on 31 December 2025, not retirement — existing automations continue to run but receive no
  bug fixes.
- Advises against the Migrate to Flow tool as a migration strategy, since it converts 1:1
  and multiplies rather than reduces the maintenance surface.

[Unreleased]: https://github.com/divyeshardeshana/salesforce-administrator/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/divyeshardeshana/salesforce-administrator/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/divyeshardeshana/salesforce-administrator/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/divyeshardeshana/salesforce-administrator/releases/tag/v1.0.0

# Changelog

All notable changes to this skill are documented here.

Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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

[Unreleased]: https://github.com/savvydatacloudconsulting/salesforce-administrator/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/savvydatacloudconsulting/salesforce-administrator/releases/tag/v1.0.0

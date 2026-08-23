# Contributing

Thanks for helping keep this skill accurate. Salesforce changes three times a year; this
repo only stays useful if people who hit the drift report it.

## What is most useful

1. **Corrections.** A limit that changed, a feature that was deprecated, a behaviour that no
   longer matches the description. Include a link to Salesforce Help or the release notes.
2. **Troubleshooting entries.** Each reference ends with a symptom table. If you diagnosed
   something painful, add the symptom and the *actual* root cause.
3. **Patterns from real orgs.** Configuration approaches that survived production. Not
   theoretical best practice.

## What to avoid

- Content that duplicates Salesforce Help verbatim. The value here is judgement and
  sequencing, not documentation restatement.
- Apex, LWC, or anything requiring an IDE. That belongs in a developer skill; this one is
  deliberately declarative-only.
- Screenshots. They rot faster than text and agents cannot read them.
- Padding. Length is not the goal.

## Structure rules

- `SKILL.md` is the router. It stays short enough to load on every invocation. New depth
  goes in a reference file, not here.
- Reference files are self-contained. A reader loading only `flow-automation.md` should not
  need another file to make sense of it.
- Every reference ends with a troubleshooting table.
- Paired CORRECT / WRONG examples wherever a common mistake exists.

## Style

- British or American spelling — be consistent within a file.
- Tables for anything comparative. Prose for reasoning.
- Code fences for configuration, formulas, and CLI commands.
- State trade-offs honestly, including licence cost and maintenance burden. A
  recommendation that hides its cost is not a recommendation.
- Assume the reader is a competent admin, not a beginner. Skip the Trailhead-level framing.

## Process

1. Fork, branch from `main`.
2. Make the change.
3. If you changed platform-specific facts, note the source in the PR description.
4. Update `CHANGELOG.md` under `Unreleased`.
5. Open the PR with a description of what changed and why.

Small, focused PRs get merged. A PR rewriting three reference files at once will sit.

## Reporting without contributing

Open an issue. A one-line "the roll-up summary limit is now X, this says Y" is genuinely
valuable and takes thirty seconds.

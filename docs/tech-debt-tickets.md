# Tech Debt Tickets (Identified)

This is a working backlog of tech debt tickets identified by applying the
[tech debt ticket runbook](tech-debt-ticket-runbook.md) to the processes
already documented in this repo's other runbooks and retros. Each entry
follows the runbook's required structure so it can be copy/pasted directly
into an issue tracker.

## 1. [Tech Debt] Release: stable tag creation is a fully manual, multi-step git workflow

- **Context**: `docs/stable-tag-runbook.md` requires a human to manually
  fetch, check out the approved commit SHA, generate a timestamp, confirm the
  tag name doesn't already exist, create an annotated tag, verify it matches
  the approved SHA, push it, and optionally force-update a moving `stable`
  tag — 9+ sequential git commands per release with nothing enforcing the
  order or catching a skipped step.
- **Problem / risk**: A skipped verification (e.g. not confirming the two
  `rev-parse` outputs match, or moving the `stable` tag before the immutable
  tag is pushed and verified) can result in production being deployed from an
  unverified commit, and there's no automated guardrail to catch it. This
  gets riskier as release frequency increases.
- **Proposed fix**: Wrap the runbook's steps in a script or CI job (e.g.
  `scripts/create-stable-tag.sh` / a release-tooling GitHub Action) that
  takes the approved commit SHA as input, performs the existence and
  SHA-match checks programmatically, and fails closed if any check doesn't
  pass. Keep the runbook as the documented fallback for when automation is
  unavailable.
- **Impact / effort estimate**: M — single service (release tooling), but
  needs release-owner sign-off and a live test on a real release cycle before
  it replaces the manual process.
- **Labels / fields**: `tech-debt`, `release-engineering`, `automation`
- **Links**: `docs/stable-tag-runbook.md`

## 2. [Tech Debt] DR: switchover conclusion validation relies on ad hoc manual checks across disconnected tools

- **Context**: `docs/dr-switchover-conclusion-runbook.md` has engineers
  manually run `curl` health checks, inspect replication status (e.g.
  `pg_stat_replication`), check DNS propagation with `dig`, and manually
  confirm temporary DR automation/feature-flag overrides were reverted.
  There's no single tool that aggregates these signals into one "safe to
  conclude" status.
- **Problem / risk**: During a high-stress DR conclusion, running a manual
  checklist across several disconnected systems (monitoring, DB replication
  tooling, DNS, feature-flag console) increases the odds a step is missed —
  e.g. leaving a temporary override enabled, or declaring closure before
  traffic has fully cut over — which risks data divergence or a repeat
  incident.
- **Proposed fix**: Build a single "DR conclusion status" script or dashboard
  that automatically checks site health, replication lag, DNS resolution,
  and active feature-flag/override state, and only allows the closure
  checklist to be marked complete once every automated check reports green.
- **Impact / effort estimate**: L — spans monitoring, DNS, replication
  tooling, and the feature-flag system; likely needs cross-team buy-in.
- **Labels / fields**: `tech-debt`, `infra`, `disaster-recovery`,
  `observability`
- **Links**: `docs/dr-switchover-conclusion-runbook.md`

## 3. [Tech Debt] Jira admin: no tooling to detect dependents before editing global fields like `Resolution`

- **Context**: `docs/remove-done-resolution-value-runbook.md` requires a
  human to manually search workflow post-functions, Jira Automation rules,
  ScriptRunner scripts, and every saved filter/dashboard for references to a
  specific `Resolution` value before it's safe to delete it, because
  `Resolution` (like other global fields) is shared across every project.
- **Problem / risk**: Manually searching across all of the above doesn't
  scale with the number of projects, automations, and filters, and Jira has
  no built-in undo for a deleted resolution value — a missed dependency
  causes silent breakage in automation or reporting with no easy rollback.
- **Proposed fix**: Script a pre-check using the Jira REST API that scans JQL
  in saved filters/dashboards, automation rule configs, and workflow
  post-functions for a target value, and produces a dependency report before
  any global field value is deleted or renamed. Generalize it beyond
  `Resolution` to other global fields with the same blast radius (e.g.
  `Priority`, `Issue Type`).
- **Impact / effort estimate**: M — one-time script investment, but becomes
  a reusable safety check for all future global-field edits.
- **Labels / fields**: `tech-debt`, `tooling`, `jira-admin`
- **Links**: `docs/remove-done-resolution-value-runbook.md`

## 4. [Tech Debt] Process: SPIKE process has no required step to check existing internal solutions before evaluating third-party services

- **Context**: Retro item `OEP-36068` (see README) noted that during SPIKEs
  the team explored third-party services before fully evaluating solutions
  already available internally.
- **Problem / risk**: Without an explicit gate in the SPIKE ticket
  template/definition-of-done, this is likely to recur, costing time on
  redundant third-party evaluation and risking unnecessary vendor/tooling
  sprawl.
- **Proposed fix**: Add a required "existing internal solutions considered"
  checklist item to the team's SPIKE ticket template so the check is
  structurally enforced rather than relying on memory.
- **Impact / effort estimate**: S — process/template change, single team.
- **Labels / fields**: `tech-debt`, `process`, `documentation`
- **Links**: `OEP-36068` (see README retro item)

## Next steps

- File each of the above as its own ticket in the issue tracker using the
  runbook's template, apply the `tech-debt` label, and route into the next
  triage/prioritization pass described in the runbook's
  [Triage cadence](tech-debt-ticket-runbook.md#triage-cadence) section.
- As new manual-process runbooks are added to this repo, re-run this same
  exercise: any runbook step described as manual, ad hoc, or lacking
  tooling/verification is a candidate tech debt ticket.

# Tech Debt Ticket Runbook

Use this runbook to identify, write, and triage tech debt tickets consistently
so that debt is visible, prioritized, and actually gets scheduled instead of
living only in comments, Slack threads, or tribal knowledge.

## Goals

- Make tech debt visible and trackable alongside regular work.
- Ensure every tech debt ticket has enough context for someone else (including
  future you) to pick it up without re-discovering the problem.
- Give tech debt a fair chance at being prioritized by tying it to concrete
  risk, cost, or velocity impact.

## What counts as tech debt

Tech debt is a deliberate or accidental shortcut, gap, or outdated decision
that increases future cost, risk, or effort. Common categories:

- **Code quality**: duplicated logic, missing abstractions, dead code,
  inconsistent patterns.
- **Testing gaps**: missing/flaky tests, low coverage on critical paths, no
  regression tests for a past incident.
- **Infrastructure/ops**: outdated dependencies, unpatched CVEs, manual
  deploy/runbook steps that should be automated, missing monitoring/alerts.
- **Architecture**: workarounds shipped under time pressure, temporary
  integrations, scaling bottlenecks.
- **Documentation**: stale or missing docs/runbooks that cause repeated
  ramp-up cost or incident delay.

If it's not costing anything (time, risk, reliability, developer experience)
now or in the foreseeable future, it's probably not worth a ticket — just fix
it inline.

## When to file a ticket vs. fix inline

- **Fix inline** (no ticket needed) if the fix is small, low-risk, and within
  scope of the change you're already making.
- **File a ticket** if the fix is out of scope for the current change, needs
  its own review/testing, needs prioritization against other work, or affects
  more than one team/service.

## Writing the ticket

Use this structure for every tech debt ticket:

1. **Title**: short, specific, and searchable. Prefer
   `[Tech Debt] <area>: <problem>` (e.g. `[Tech Debt] Billing: retry logic
   duplicated across 3 services`).
2. **Context**: what exists today, why it was built this way, and any
   relevant history (link the PR, incident, or decision that introduced it).
3. **Problem / risk**: what breaks, slows down, or becomes riskier because of
   this debt. Be concrete — "hard to read" is weak; "each new payment
   provider requires editing 4 files and has caused 2 production bugs" is
   strong.
4. **Proposed fix**: a rough direction, even if it needs design work later.
   Note alternatives considered if relevant.
5. **Impact / effort estimate**: rough size (S/M/L) and expected
   blast radius (single service vs. cross-team).
6. **Labels / fields**: apply the `tech-debt` label (or equivalent) plus the
   normal component/team labels so it shows up in standard triage and
   reporting views.
7. **Links**: related tickets, incidents, PRs, and Slack threads.

## Prioritization signals

When triaging, weigh these signals to decide urgency:

- Has this debt caused an incident or near-miss?
- Is it blocking or slowing down current roadmap work?
- Is it a security/compliance risk (e.g. unpatched dependency, expiring
  cert/credential process)?
- Does it compound over time (gets more expensive to fix the longer it sits)?
- How many people does it affect per week/sprint?

Tickets with a clear incident link or security implication should be flagged
for the next planning cycle rather than left in a general backlog.

## Triage cadence

- Review the `tech-debt` label backlog on a recurring cadence (e.g. every
  sprint planning or a dedicated monthly review).
- Close or update stale tickets: if context has changed and the ticket is no
  longer accurate, update it or close it with a note rather than letting it
  rot.
- When a tech debt ticket is picked up, link the resulting PR(s) back to the
  ticket for traceability.

## Final checklist

Before filing a tech debt ticket, confirm:

- [ ] Title is specific and searchable.
- [ ] Context explains what exists today and why.
- [ ] Problem/risk is described in concrete terms (impact, not just annoyance).
- [ ] A proposed direction (even rough) is included.
- [ ] `tech-debt` label plus normal team/component labels are applied.
- [ ] Related incidents, PRs, or threads are linked.

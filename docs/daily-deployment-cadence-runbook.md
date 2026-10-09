# Daily Automatic Deployment Alignment Runbook

Use this runbook to align the scrum team around a single scheduled,
automatic deployment per day so that manual deploy triggers stop being the
default path and are reserved strictly for testing and shipping hotfixes.

## Goals

- One scheduled, automatic deployment per day is the default and expected
  release mechanism for the team.
- Manual/on-demand deploys are the exception, used only to test and ship a
  hotfix — not as a substitute for the daily cadence.
- Every team member knows the deploy time, the merge cutoff, and exactly
  what qualifies as a hotfix.
- Reduce ad hoc "can you deploy now?" requests, on-call noise, and the
  uncertainty of not knowing what's running in production at a given time.

## Why this matters (the problem today)

- Without an agreed cadence, anyone can trigger a deploy at any time "just
  to be safe," which multiplies the number of production change windows,
  increases on-call load, and makes it hard to reason about what's live.
- If manual deploys and hotfix deploys look identical, hotfixes lose the
  urgency signal that should make them rare and visible.
- Deploys that happen whenever someone remembers to click the button don't
  show up in planning, so testing and release risk aren't accounted for.

## Define the cadence

1. **Pick one fixed daily deploy time** (e.g. 10:00 team-local time) that
   sits after a merge cutoff and leaves enough of the day to react if
   something goes wrong. Agree this in sprint planning / the team's working
   agreement, not ad hoc.
2. **Automate the trigger** so the default path requires no human action.
   For example, a scheduled GitHub Actions workflow:

   ```yaml
   name: Daily production deploy

   on:
     schedule:
       - cron: "0 10 * * 1-5" # 10:00 UTC, Mon-Fri
     workflow_dispatch:
       inputs:
         reason:
           description: "Hotfix ticket/incident link (required for manual runs)"
           required: true

   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - run: echo "Deploying ${{ github.sha }} (trigger: ${{ github.event_name }}, reason: ${{ github.event.inputs.reason }})"
         # ... build, test, and deploy steps ...
   ```

   Keep `workflow_dispatch` available for the manual/hotfix path, but require
   a reason input so every manual run is self-documenting.
3. **Set a merge cutoff** before the deploy time (e.g. 30–60 minutes).
   Anything merged after the cutoff rides the *next* day's automatic
   deploy — it does not justify a manual trigger on its own.
4. **Publish the cadence** somewhere durable and discoverable: this
   runbook, the team wiki, and the topic/pinned message in the team's
   deploy channel, so "when does it deploy" is self-serve and doesn't
   require asking.

## Scrum ceremony alignment

- **Sprint planning**: size and sequence work assuming the daily cadence;
  don't plan work that depends on chaining multiple manual deploys in a
  day.
- **Daily standup**: only call out *exceptions* — a missed cutoff, a
  blocked automatic deploy, or a hotfix in flight. Routine "did it deploy
  today?" status checks shouldn't be needed once the schedule is trusted.
- **Definition of Done**: a ticket is Done when it has merged *and* has
  been confirmed live after the next scheduled automatic deploy — not
  merely when the PR is merged.
- **Retro**: explicitly review how many manual deploys happened for
  non-hotfix reasons that sprint. A rising count is a signal to fix the
  root cause (cutoff too early, pipeline too slow, flaky tests) rather
  than a reason to keep deploying manually.

## What qualifies as a hotfix (manual trigger allowed)

Manual deploy triggers should be rare and reserved for genuine hotfixes.
Require all of the following before anyone triggers manually:

- [ ] A live production incident or customer-impacting defect exists
      (link the incident/bug ticket).
- [ ] The fix is small and scoped only to the defect — no unrelated
      changes riding along.
- [ ] The fix has passed the same required CI checks as the scheduled
      deploy (no skipping tests to go faster).
- [ ] An accountable person (on-call lead or team lead) has explicitly
      approved the manual trigger.

If any box is unchecked, hold the change for the next scheduled automatic
deploy instead of triggering manually.

## Manual hotfix deploy steps

1. Open or confirm the incident/bug ticket describing the customer
   impact.
2. Branch the fix from the current production commit/tag; keep the diff
   minimal and scoped to the defect.
3. Get expedited review and the required approval from the on-call lead
   or team lead.
4. Trigger the manual deploy workflow, passing the incident/ticket link
   as the required reason (e.g.
   `gh workflow run deploy.yml -f reason=INC-1234`).
5. Verify the fix in production and attach verification evidence back to
   the incident ticket.
6. Note the manual deploy in the next standup and retro so the team can
   evaluate whether the cadence or pipeline needs adjusting.

## Guardrails that keep manual deploys rare

- Require a ticket/incident link as input to the manual-trigger workflow
  so it cannot be run without one (see the `workflow_dispatch.inputs`
  example above).
- Notify the team channel automatically whenever a manual deploy runs, so
  it's visible rather than silent.
- Track manual-deploy frequency every retro; treat an increasing trend as
  a process or pipeline problem to fix, not as evidence the cadence should
  be abandoned.

## Final checklist

Before calling the cadence "adopted" by the team, confirm:

- [ ] A single daily deploy time is agreed and documented.
- [ ] A merge cutoff before that time is agreed and documented.
- [ ] The scheduled deploy is automated (e.g. cron-triggered CI job) with
      no manual approval required for the default path.
- [ ] The manual trigger path requires an incident/ticket reference and
      an explicit approver.
- [ ] Definition of Done reflects confirmation after the scheduled
      deploy, not just PR merge.
- [ ] Manual-deploy frequency is reviewed in retro.

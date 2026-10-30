# QA Stable Tag Automation Runbook

Use this runbook to give QA a steady, trustworthy supply of "stable" tags to
test against on a codebase that merges multiple times a day, without a human
re-running the full manual tag checklist every time QA needs a build.

This is a companion to `docs/stable-tag-runbook.md` (production release
tagging). That runbook governs the rare, high-stakes production tag; this one
governs the frequent, lower-stakes QA tag. Keep the two processes and tag
namespaces separate so automating QA tags never creates a shortcut into
production.

## Goals

- QA always has a recent tag built from a commit that already passed CI —
  never from whatever happens to be at the tip of the branch.
- Cutting that tag requires no manual git choreography, even on a day with
  many merges.
- QA can request a fresh tag on demand between scheduled cuts without going
  around the automation or needing engineer involvement.
- Every QA tag is traceable to an exact commit, CI run, and change list.

## Why this is hard with continuous daily development

- If QA tags are cut manually, "create a stable tag" competes with every
  other request for engineer time, so it either happens inconsistently or an
  engineer starts skipping verification steps to keep up.
- Tagging whatever commit is at the tip of the branch risks tagging
  in-progress or red-CI work, since merges land continuously throughout the
  day.
- Without a named, repeatable process, different engineers tag differently
  (some skip the SHA verification step, some reuse a moving tag name,
  some forget to notify QA), so QA can't trust what a given tag actually
  contains.
- As merge frequency increases, the gap between "a tag exists" and "QA knows
  a new tag exists and what changed in it" becomes the real bottleneck, not
  git mechanics.

## Strategy overview

1. **Decouple tagging from merging.** Do not tag on every merge to the
   integration branch. Tag on a fixed cadence (e.g. nightly) plus an
   on-demand trigger, so tag volume stays manageable and each tag represents
   a meaningful batch of changes.
2. **Only ever tag a commit that already passed CI.** Gate the integration
   branch with required status checks so any commit eligible for tagging has
   already been built and tested — the tagging job doesn't need to run tests
   itself, only confirm the commit is green.
3. **Make the CI job tag, not a person.** Replace the manual steps in
   `docs/stable-tag-runbook.md` with a script invoked from CI, so creation,
   existence checks, and SHA verification are enforced programmatically
   every time. (This closes the gap described in tech debt ticket #1 in
   `docs/tech-debt-tickets.md`, scoped specifically to the QA tag.)
4. **Use an immutable tag per cut, plus one moving pointer.** Create an
   immutable, timestamped tag for every cut (`qa-stable-YYYYMMDD-HHMM`) so
   every QA run is reproducible, and also update a single moving pointer tag
   (`qa-stable-latest`) that QA tooling can reference without knowing the
   exact timestamp.
5. **Auto-notify QA with a changelog**, not just a tag name — a tag nobody
   knows exists doesn't save any time.
6. **Automatically retire old tags** so the tag list doesn't grow without
   bound as cadence/on-demand cuts accumulate.

## Implementation

### 1. Make every commit on the integration branch taggable

Require the integration branch (e.g. `main` or `develop`) to pass build,
unit, and integration checks before merge (branch protection / required
status checks). Once this is true, "the latest commit on the integration
branch" is always a safe tagging candidate, and the tagging job never needs
to run the full test suite itself — it only needs to confirm the commit it's
about to tag is the one CI last marked green.

### 2. Scheduled + on-demand tagging workflow

```yaml
name: QA stable tag

on:
  schedule:
    - cron: "0 2 * * 1-5" # nightly, Mon-Fri
  workflow_dispatch:
    inputs:
      reason:
        description: "Why this tag is needed now (e.g. QA-1234)"
        required: false

jobs:
  tag:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Create QA stable tag
        run: ./scripts/create-qa-stable-tag.sh
        env:
          REASON: ${{ github.event.inputs.reason || 'scheduled' }}
```

Keeping `workflow_dispatch` available means QA can self-serve a tag the
moment they need one (e.g. to verify a specific fix landed) instead of
waiting for the nightly cut or pinging an engineer.

### 3. Tagging script responsibilities

`scripts/create-qa-stable-tag.sh` should perform the same checks the manual
runbook requires, but without a human in the loop:

```bash
#!/usr/bin/env bash
set -euo pipefail

git fetch origin main --tags
git checkout "$(git rev-parse origin/main)"

TAG="qa-stable-$(date -u +%Y%m%d-%H%M)"
if git rev-parse -q --verify "refs/tags/${TAG}" >/dev/null; then
  echo "Tag ${TAG} already exists, aborting" >&2
  exit 1
fi

PREV_TAG="$(git describe --tags --match 'qa-stable-*' --abbrev=0 \
  "$(git rev-list --tags='qa-stable-*' --max-count=1)" 2>/dev/null || true)"

git tag -a "${TAG}" -m "QA stable tag (${REASON:-scheduled})"
[ "$(git rev-parse "${TAG}^{commit}")" = "$(git rev-parse HEAD)" ] # fail closed on mismatch

git push origin "${TAG}"
git tag -f -a qa-stable-latest -m "QA stable tag (${REASON:-scheduled})"
git push --force-with-lease=refs/tags/qa-stable-latest \
  origin refs/tags/qa-stable-latest:refs/tags/qa-stable-latest

CHANGELOG="$(git log --oneline "${PREV_TAG:-}..${TAG}" 2>/dev/null || echo 'first QA tag')"
./scripts/notify-qa-channel.sh "${TAG}" "$(git rev-parse HEAD)" "${CHANGELOG}"
```

Notes:

- Fail closed: if the SHA check after tagging doesn't match, the script
  should exit non-zero rather than push a tag that silently points at the
  wrong commit.
- `qa-stable-latest` is a convenience pointer only. QA automation that needs
  reproducibility (e.g. a bug report) should always record and reference the
  immutable `qa-stable-YYYYMMDD-HHMM` tag, never `qa-stable-latest`.

### 4. Notify QA with a changelog, not just a tag name

The notification (Slack, email, or ticket comment) should include:

- The immutable tag name and commit SHA.
- The CI run/build artifact the tag was cut from.
- A one-line-per-commit changelog since the previous QA tag, so QA knows
  what's new to test without reading every PR.
- The trigger reason (`scheduled` vs. a `workflow_dispatch` reason string),
  so an on-demand cut is distinguishable from the nightly cut.

### 5. Retention policy

Automatically delete `qa-stable-*` tags older than a fixed window (e.g. 30
days) or beyond a fixed count (e.g. keep the last 30), excluding any tag
currently referenced by an open bug report. Run this as a separate scheduled
job so tag cleanup doesn't block tag creation.

## Guardrails

- Keep the `qa-stable-*` namespace and tooling completely separate from the
  production `stable-*` tags and workflow in `docs/stable-tag-runbook.md`.
  Automating QA tags should never make it easier to accidentally promote a
  QA tag to production.
- Branch protection (required CI checks) is a prerequisite, not optional —
  without it, the automation will faithfully tag broken commits on schedule.
- Pilot the scheduled + on-demand workflow on one service/repo before
  rolling it out team-wide, and compare manual-tag-request volume before and
  after to confirm it actually reduced ad hoc engineer interruptions.

## Final checklist

- [ ] Integration branch has required CI checks enforced via branch
      protection.
- [ ] Scheduled tagging workflow is configured and has run successfully at
      least once.
- [ ] On-demand (`workflow_dispatch`) path works and requires a reason.
- [ ] Tag creation fails closed on a SHA mismatch or duplicate tag name.
- [ ] QA notification includes tag name, commit SHA, build link, and
      changelog since the previous QA tag.
- [ ] `qa-stable-latest` pointer updates only after the immutable tag is
      verified and pushed.
- [ ] Old `qa-stable-*` tags are cleaned up on a defined retention policy.
- [ ] QA tagging automation and tooling are fully separate from the
      production stable-tag process.

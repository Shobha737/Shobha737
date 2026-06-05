# Stable Tag Creation Runbook

Use this runbook before promoting changes to production. The stable tag marks the
exact commit that has been approved for release so production deployments can be
traced, audited, and rolled back if needed.

## Goals

- Confirm the release commit is reviewed, tested, and approved.
- Create an annotated stable tag that points to the exact production candidate.
- Push the tag only after verification.
- Record enough context for future debugging or rollback.

## Prerequisites

Before creating the stable tag, confirm:

- The release branch or commit has passed required CI checks.
- All required reviews and approvals are complete.
- Release notes or change summaries are prepared.
- Database migrations, feature flags, and configuration changes have been
  reviewed for production impact.
- The production deploy owner has confirmed the target commit SHA.
- You have permission to create and push tags to the repository.

## Tag naming

Prefer immutable, timestamped tags:

```bash
stable-YYYYMMDD-HHMM
```

Example:

```bash
stable-20260605-1318
```

You can generate the timestamp with:

```bash
date -u +%Y%m%d-%H%M
```

If the repository also uses a moving `stable` tag as a deployment channel, update
it only after the immutable tag has been created and pushed. Moving tags require
extra approval because they rewrite where the tag points.

## Create the stable tag

1. Start from a clean working tree.

   ```bash
   git status --short --branch
   ```

   Stop if there are uncommitted changes that are not part of the release.

2. Fetch the latest target branch and tags.

   ```bash
   git fetch origin <target-branch> --tags
   ```

3. Check out the exact commit approved for production.

   ```bash
   git checkout <approved-commit-sha>
   ```

4. Verify the commit details.

   ```bash
   git show --stat --oneline --decorate HEAD
   git log --oneline --decorate -n 5
   ```

5. Choose the stable tag name and confirm it does not already exist.

   ```bash
   STABLE_TAG=stable-YYYYMMDD-HHMM
   git rev-parse -q --verify "refs/tags/${STABLE_TAG}"
   ```

   The `rev-parse` command should return no output. If it returns a SHA, choose a
   new immutable tag name.

6. Create an annotated stable tag.

   ```bash
   git tag -a "${STABLE_TAG}" -m "Stable release YYYY-MM-DD HH:MM UTC"
   ```

7. Verify the tag points to the expected commit.

   ```bash
   git show "${STABLE_TAG}" --stat
   git rev-parse "${STABLE_TAG}^{commit}"
   git rev-parse HEAD
   ```

   The two `rev-parse` commands must return the same commit SHA.

8. Push the immutable stable tag.

   ```bash
   git push origin "${STABLE_TAG}"
   ```

9. Confirm the tag exists on the remote.

   ```bash
   git ls-remote --tags origin "${STABLE_TAG}"
   ```

## Optional: update a moving `stable` tag

Only perform this section if the production process explicitly uses a moving
`stable` tag.

1. Confirm approval from the release owner to move `stable`.
2. Point the local `stable` tag at the verified commit.

   ```bash
   git tag -f -a stable -m "Stable release YYYY-MM-DD HH:MM UTC"
   ```

3. Push the updated `stable` tag.

   ```bash
   git push --force-with-lease=refs/tags/stable origin refs/tags/stable:refs/tags/stable
   ```

4. Confirm the remote `stable` tag points to the same commit as the immutable
   tag.

   ```bash
   git ls-remote --tags origin stable "${STABLE_TAG}"
   ```

## Production deployment handoff

Before pushing changes to production, share:

- Stable tag name.
- Commit SHA.
- CI run or build artifact used for the release.
- Release notes or change summary.
- Known migrations, feature flags, or operational steps.
- Rollback tag or previously deployed commit SHA.

## Rollback guidance

If production must roll back:

1. Identify the previous known-good stable tag or production commit.
2. Deploy that tag or commit using the standard production deployment process.
3. Record the rollback reason, affected release tag, and restored tag or commit.

Do not delete or rewrite immutable stable tags during rollback. Create a new
rollback tag if a permanent record is required.

## If the tag was created incorrectly

- If the tag has not been pushed, delete it locally and recreate it:

  ```bash
  git tag -d stable-YYYYMMDD-HHMM
  git tag -a stable-YYYYMMDD-HHMM -m "Stable release YYYY-MM-DD HH:MM UTC"
  ```

- If the tag has already been pushed, do not rewrite it without release-owner
  approval. Prefer creating a new corrected tag, for example:

  ```bash
  git tag -a stable-YYYYMMDD-HHMM-corrected -m "Corrected stable release YYYY-MM-DD HH:MM UTC"
  git push origin stable-YYYYMMDD-HHMM-corrected
  ```

## Final checklist

- [ ] Working tree was clean before tagging.
- [ ] Approved commit SHA was verified.
- [ ] Annotated stable tag was created.
- [ ] Tag SHA matches the approved commit SHA.
- [ ] Tag was pushed to the remote.
- [ ] Remote tag was verified.
- [ ] Release owner received the tag name, commit SHA, and deployment notes.

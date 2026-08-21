# Removing "Done" from the Resolution Field Dropdown (Jira)

Use this runbook when you need to remove the `Done` value from the Jira
`Resolution` field dropdown. The `Resolution` field is a **global** field in
Jira, meaning its list of values is shared across every project, not scoped
per project like `Status`. Deleting a value here affects all projects at
once, so this change should be planned and communicated before you make it.

## Before you start

Confirm the following so the removal doesn't silently break automation or
reporting:

- **Workflow post-functions.** Check each workflow's transitions for a
  "Resolve Issue" or custom post-function that sets `Resolution = Done` when
  moving to a done-like status.
- **Automation rules.** Search Jira Automation (and any third-party apps such
  as ScriptRunner) for conditions or actions referencing `resolution = Done`.
- **Saved filters, dashboards, and JQL.** Search for `resolution = Done` or
  `resolution in (Done, ...)` across shared filters and dashboard gadgets.
- **Boards.** Some boards use resolution (not just status) to decide whether
  a card counts as complete.
- **Permission to administer resolutions.** You need Jira administrator
  (global admin) permissions; this is not a per-project permission.

If any of the above depend on `Done`, update them first, or consider the
non-destructive alternative in the "Alternatives" section below instead of
deleting the global value.

## Steps to remove the value

1. Open Jira administration:
   - **Jira Cloud:** Jira Settings (gear icon) → **Issues** → **Resolutions**.
   - **Jira Data Center / Server:** **Administration** → **Issues** →
     **Resolutions**.
2. You'll see the full list of resolution values (for example `Done`,
   `Won't Do`, `Duplicate`, `Cannot Reproduce`, `Won't Fix`), each with
   **Edit**, **Set Default**, and **Delete** operations.
3. If `Done` is currently the **default** resolution, set a different value
   as default first (via **Set Default** on another row). A value that is
   still the default typically cannot be deleted.
4. Click **Delete** next to `Done`.
5. If any issues currently have `Resolution = Done` set, Jira will prompt you
   to choose a **replacement resolution**. Pick the value that existing
   `Done` issues should be reassigned to (for example `Fixed` or `Complete`).
   Jira bulk-updates all affected issues to the replacement value as part of
   the delete.
6. Confirm the deletion.

## Verify

- Reload **Issues → Resolutions** and confirm `Done` no longer appears in the
  list.
- Open the resolution dropdown on a "Resolve Issue" transition screen in an
  affected project and confirm `Done` is gone.
- Spot-check a few issues that previously had `Resolution = Done` to confirm
  they now show the chosen replacement value.
- Re-run any saved filters/dashboards identified in the pre-check step to
  confirm they still return expected results.

## Alternatives to a full delete

If you only want to stop `Done` from being selectable going forward, without
a global, irreversible delete and bulk reassignment of historical issues:

- **Remove it from the relevant transition screen** so users can't pick it
  during that specific workflow transition, while leaving the global value
  intact for other projects that still rely on it.
- **Rename it** (via **Edit**) to something clearer instead of deleting it,
  if the real problem is naming/confusion rather than needing it gone
  entirely.

## Rollback

Jira does not version resolution values, so there is no built-in undo:

- If you deleted `Done` by mistake, re-create a resolution value with the
  same name via **Issues → Resolutions → Add Resolution**.
- Issues that were bulk-reassigned to the replacement value during deletion
  will **not** automatically revert; you would need to identify and manually
  reassign them back to `Done` if that history matters.

## Final checklist

- [ ] Confirmed no active workflow post-function depends on `Done`.
- [ ] Confirmed no automation rule depends on `Done`.
- [ ] Confirmed no shared filter/dashboard depends on `Done` (or updated
      them).
- [ ] `Done` was not the default resolution, or a new default was set first.
- [ ] Replacement resolution chosen for existing `Done` issues.
- [ ] Deletion confirmed and verified in **Issues → Resolutions**.
- [ ] Spot-checked reassigned issues and re-ran affected filters/dashboards.

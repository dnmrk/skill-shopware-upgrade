# Phase 15 — Lessons Workflow Reference

After every upgrade, close the loop. This phase must always run — even if the upgrade was routine.

## Step 1 — Write lessons to `tasks/lessons.md`

Review the upgrade from Phase 0 to 14 and capture anything that:
- Was surprising, unclear, or not covered by the skill
- Required manual investigation or user clarification
- Caused a phase to be retried or reordered
- Surfaced a new edge case in plugin compatibility, recipe conflicts, or build failures

Append findings to `tasks/lessons.md` in this format:

```
## <target_version> upgrade — <date>
- <specific lesson, Shopware-flavored>
- ...
```

If Phase 15 produces nothing to capture, note that explicitly — a clean upgrade with no surprises is itself worth recording.

## Step 2 — Propose skill improvements

For each lesson, evaluate whether it belongs in the skill's **Common Mistakes** table (`references/common-mistakes.md`) or as a new constraint/invariant in `SKILL.md`. Present the proposed additions to the user:

```
Proposed Common Mistakes additions:
| <Mistake> | <Fix> |
```

**Do not edit the skill file directly.** Surface the proposals and wait for the user to confirm before any skill edits are made.

## Step 3 — Mark upgrade complete

```bash
git log --oneline -5   # confirm upgrade commit is present
```

Summarise:
- Target version confirmed in `composer.lock`
- Smoke test passed (Phase 13 green)
- Deployment checklist written to `tasks/<target_version>/deployment-checklist.md`
- Lessons written to `tasks/lessons.md`

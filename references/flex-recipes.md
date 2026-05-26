# Phase 11 — Flex Recipes Reference

## Find pending updates

```bash
composer recipes | grep "update available"
```

Review each before applying:

```bash
composer recipes <package>   # shows "since" date and affected files
```

## Applying recipes

**`composer recipes:update` is interactive — do NOT use `-T`** with `docker compose exec`. Drop the flag so it gets a proper TTY.

**Apply one recipe at a time and commit between each.** The command requires a clean git index. Running a second update before committing the first will fail silently or produce a confusing error.

```bash
# Correct workflow
composer recipes:update <package>
git add -p   # review and stage changes
git commit -m "apply <package> recipe"
# Then move to the next
```

## When to skip a recipe

If a recipe touches a project-customized config file (e.g. `phpcs.xml.dist`, `config/packages/shopware.yaml`) and the update has been available for years, it was almost certainly left unapplied intentionally — the project has diverged from the upstream default. Skip it and note this in the commit or deployment checklist so the team knows it was a deliberate decision.

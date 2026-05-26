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

**If the project has uncommitted tracked changes** (e.g. a modified `.gitignore`), stash them first:

```bash
git stash push -m "pre-recipe stash" -- path/to/modified/file
composer recipes:update <package>
git add -p
git commit -m "apply <package> recipe"
git stash pop
# Repeat for each remaining recipe
```

```bash
# Normal workflow (clean working tree)
composer recipes:update <package>
git add -p   # review and stage changes
git commit -m "apply <package> recipe"
# Then move to the next
```

## Known recipe side effects

**`shopware/paas-meta`** — if `.platform/applications.yaml` does not exist in the project, the recipe update writes `shopware.paas-meta.updates-for-deleted-files.patch` to the project root. This file is irrelevant to non-PaaS projects — delete it before committing:

```bash
rm shopware.paas-meta.updates-for-deleted-files.patch
git commit -m "apply shopware/paas-meta recipe" symfony.lock
```

**`shopware/core`** — if `.htaccess` / `public/.htaccess.dist` do not exist in the project, the recipe update writes `shopware.core.updates-for-deleted-files.patch` to the project root. Same rule: delete it before committing.

```bash
rm shopware.core.updates-for-deleted-files.patch
```

**`shopware/paas-meta` with customized `.platform/` or `config/services.yaml`** — projects with custom cron jobs, env vars, or service defaults will get merge conflicts. Strategy: `git checkout --ours` for the conflicted files to restore project customizations as base, then manually pick up new additions from the recipe diff (e.g., `MESSENGER_TRANSPORT_DSN` env defaults, updated `NODE_VERSION` / `SHOPWARE_CLI_VERSION`).

## When to skip a recipe

If a recipe touches a project-customized config file (e.g. `phpcs.xml.dist`, `config/packages/shopware.yaml`) and the update has been available for years, it was almost certainly left unapplied intentionally — the project has diverged from the upstream default. Skip it and note this in the commit or deployment checklist so the team knows it was a deliberate decision.

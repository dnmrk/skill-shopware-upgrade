---
name: shopware-upgrade
description: Use when upgrading a Shopware 6 installation to a newer version — patch, minor, or major. Triggers: target version specified, composer.json version constraint needs updating, extension compatibility unknown, or maintenance window required for the upgrade.
---

# Shopware Upgrade

## Overview

Step-by-step Shopware version upgrade. Use TaskCreate for each phase so progress is visible. Detect Docker first — every command runs inside the container if found.

## References

This skill uses reference files for complex phases. Read them when you reach the relevant phase:

| Reference | When to read |
|---|---|
| `references/compatibility-check.md` | Phase 4 — interpreting upgrade-check output, proprietary plugins |
| `references/version-bump.md` | Phase 7 — path-repo plugins, store plugin major bumps |
| `references/build.md` | Phase 9 — deployment-helper logic, API removal fixes |
| `references/flex-recipes.md` | Phase 11 — TTY requirements, commit-between workflow |
| `references/deployment-checklist.md` | Phase 14 — full checklist template and QA notes rules |

## Step 0: Environment Detection (do this before anything else)

```bash
ls docker-compose.yml compose.yml 2>/dev/null
```

If either file exists, determine the PHP service name (`web`, `app`, `php`, `shop`) and prefix **every** subsequent command:

```bash
docker compose exec <service> <command>
```

Confirm `shopware-cli` is available inside the container:

```bash
docker compose exec <service> shopware-cli version   # Docker
shopware-cli version                                  # no Docker
```

## Upgrade Phases

Use **TaskCreate** to create one todo per phase. Mark each complete as you finish it.

### Phase 1 — Backup

```bash
mkdir -p tasks/<target_version>
shopware-cli project dump \
  --output tasks/<target_version>/<current_version>_dump.sql \
  --compression gzip
```

### Phase 2 — Branch

Check existing branch names (`git branch -a`) and match the project's naming convention.  
If the convention is unclear, **ask before creating the branch**.

```bash
git checkout -b <branch-name>
```

### Phase 3 — Vendor Sync

```bash
composer install
```

### Phase 4 — Compatibility Check

```bash
# Requires interactive TTY — do NOT add -T flag
docker compose exec <service> shopware-cli project upgrade-check --no-interaction
```

→ **Read `references/compatibility-check.md`** for how to interpret output, version-mismatch warnings, and proprietary plugin handling before acting on any results.

### Phase 5 — Maintenance On

```bash
bin/console sales-channel:maintenance:enable --all
```

### Phase 6 — Infrastructure Gate (major upgrades only)

If upgrading a major version (e.g. 6.6 → 6.7), verify manually against the Shopware upgrade guide:

- PHP version requirement for the target release
- Redis/Valkey compatibility
- Elasticsearch/OpenSearch compatibility

This is a **manual decision gate** — confirm with the user before proceeding.

### Phase 7 — Version Bump

Edit `composer.json`: update `shopware/core` (and any other pinned `shopware/*` packages) to the target constraint.

→ **Read `references/version-bump.md`** for the full details: which packages to update, path-repo plugin constraint widening for major upgrades, and store plugin major bumps.

### Phase 8 — Composer Update

```bash
composer update --no-scripts
```

### Phase 9 — Build

Check for `shopware/deployment-helper`:

```bash
composer show shopware/deployment-helper 2>/dev/null
```

→ **Read `references/build.md`** for the full build logic: deployment-helper run, failure handling (messenger, OrbStack), skip flags, and the manual chain fallback. Also includes the removed-API scan for major upgrades.

### Phase 10 — Commit

```bash
git add composer.json composer.lock symfony.lock
git add custom/static-plugins/*/composer.json   # if constraints were widened
git add custom/static-plugins/                  # any code fixes for API changes
git commit -m "<branch>: upgrade Shopware to <target_version>"
```

### Phase 11 — Flex Recipes

```bash
composer recipes | grep "update available"
```

→ **Read `references/flex-recipes.md`** for the full workflow: TTY requirement, apply-one-at-a-time rule, and when to skip.

### Phase 12 — Maintenance Off

```bash
bin/console sales-channel:maintenance:disable --all
```

### Phase 13 — Smoke Test

Verify storefront and administration are responding. **Do not generate the deployment checklist until this phase passes.**

**Storefront check** — get all active sales channel domains from `bin/console sales-channel:list`, then:

```bash
curl -o /dev/null -s -w "%{http_code}" <storefront_url>
# Expected: 200. Anything 5xx means the build or migration failed silently.
```

If you cannot determine the storefront URL, ask the user before proceeding.

**Administration check:**

```bash
curl -o /dev/null -s -w "%{http_code}" <admin_url>/admin
# Expected: 200. A 404 means admin assets weren't built or installed correctly.
```

**Log check:**

```bash
tail -n 100 var/log/prod.log | grep -E "ERROR|CRITICAL"
```

If any 5xx responses or CRITICAL log entries appear: **stop, report the errors to the user, do not proceed to Phase 14.**

### Phase 14 — Deployment Checklist

→ **Read `references/deployment-checklist.md`** for the full template and QA Notes rules. Write the output to `tasks/<target_version>/deployment-checklist.md`. Tailor the QA Notes based on upgrade type and everything that changed (plugins bumped, recipes applied, API fixes made).

---

## Model Usage

Use a **low-cost model** (Haiku) for subagents that only run shell commands (backup, composer, cache:clear). Reserve Sonnet/Opus for decision points: compatibility review, infrastructure gate, recipe diffs.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Running commands on host when Docker is present | Always check for `docker-compose.yml` first |
| Running `shopware-cli` on host when Docker is present | `shopware-cli` is also run inside the container — same rule as every other command |
| Updating all `shopware/*` packages blindly | Only update pinned `shopware/*` packages — scan all entries, not just core/storefront/administration |
| Skipping `upgrade-check` before bumping version | Extensions may break silently without it |
| Forgetting `--no-scripts` on `composer update` | Scripts can fail mid-upgrade before build step |
| Not asking about branch naming convention | Creates branch that won't pass CI naming rules |
| Running `composer recipes:update` with `-T` | The command is interactive — requires a TTY; drop `-T` from `docker compose exec` |
| Trusting upgrade-check warnings for a patch upgrade | upgrade-check auto-selects latest major as target; warnings are irrelevant for same-minor patch upgrades |
| Treating "Not compatible" as a hard blocker for private plugins | upgrade-check queries the Store API — private plugins always show "Not compatible". Read the plugin's own `composer.json` |
| Forgetting to widen path-repo plugin constraints for major upgrades | All `custom/static-plugins/*/composer.json` need `shopware/core` widened — the constraint blocks resolution even with path repos |
| Running `composer recipes:update` multiple times without committing between | Each call requires a clean git index — commit after the first recipe before applying the next |
| Generating deployment checklist before smoke tests pass | Phase 14 only runs after Phase 13 confirms HTTP 200 on storefront and admin with no CRITICAL log entries |
| Writing generic QA Notes | QA Notes must name each bumped plugin, each applied recipe, and each API fix — testers need scope, not boilerplate |

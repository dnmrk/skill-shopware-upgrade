---
name: shopware-upgrade
description: Use when upgrading a Shopware 6 installation to a newer version — patch, minor, or major. Triggers, target version specified, composer.json version constraint needs updating, extension or plugin compatibility unknown, maintenance window required, flex recipes out of date, deprecated or removed APIs, version bump blocked by constraint conflicts, upgrade-check warnings need interpretation, or migration required after version change.
allowed-tools:
  - Read
  - Glob
  - Grep
---

# Shopware Upgrade

## Overview

Step-by-step Shopware version upgrade. Use TaskCreate for each phase so progress is visible. Detect Docker first — every command runs inside the container if found.

## Hard Constraints

- **Never skip Phase 4 (compatibility check)** — extensions break silently without it.
- **Every command runs inside the container** — detect Docker at Step 0 before touching anything else.
- **Never proceed to Phase 14 (deployment checklist) until Phase 13 (smoke test) passes** — HTTP 200 alone is not sufficient.
- **Reference files are required context, not optional extras** — read the relevant one before acting on its phase, not after.
- **Never treat upgrade-check output as a final verdict** — always cross-reference private/path-repo plugins against their own `composer.json`.

## Signal → Reference Routing

| Task signal | Always read | Also read if |
|---|---|---|
| compatibility, upgrade-check, plugin warnings, "Not compatible" | `compatibility-check.md` | `version-bump.md` if major upgrade |
| composer.json, version constraint, path-repo, store plugin, constraint conflict | `version-bump.md` | `compatibility-check.md` if warnings are ambiguous |
| build, deployment-helper, OrbStack, messenger, removed API | `build.md` | `version-bump.md` if major upgrade broke APIs |
| symfony recipes, flex, recipe update, recipe out of date | `flex-recipes.md` | — |
| smoke test, deployment checklist, QA notes, maintenance off | `deployment-checklist.md` | — |

## Mode-Specific Behavior

| Mode | Do |
|---|---|
| **Planning / scoping** | Detect version from `composer.json`, classify upgrade type (patch/minor/major), run infrastructure gate if major, surface unknowns before starting phases |
| **Executing upgrade** | Follow phases in order, read the phase reference on entry, mark TaskCreate todos complete as you go, stop at blockers rather than pushing through |
| **Post-upgrade review** | Pin baseline version, walk full Phase 13 smoke test, tailor QA Notes to what actually changed — plugins bumped, recipes applied, API fixes made |

## Cross-Cutting Invariants

- Read the target version from `composer.json` — **never assume it**.
- Major upgrades (e.g. 6.6 → 6.7) require the **infrastructure gate** (Phase 6) before the version bump.
- `upgrade-check` evaluates against the **latest Shopware version**, not your target — patch-upgrade warnings are irrelevant.
- Private/path-repo plugins **always show "Not compatible"** in upgrade-check — inspect the plugin's own `composer.json` instead.
- `composer recipes:update` requires **TTY** and a **clean git index** between each recipe — never batch without committing.
- HTTP 200 on `/admin` is **not a passing smoke test** — confirm translated labels or verify the page title is "Login | Shopware Administration".

## Source Trust Hierarchy

1. `composer.json` / `composer.lock` — authoritative for currently installed versions
2. Shopware changelog and migration guide for the target tag
3. Plugin's own `composer.json` require constraints — overrides upgrade-check for private/path-repo plugins
4. `shopware-cli upgrade-check` output — useful signal, not a hard blocker
5. Shopware docs, blog posts, forum answers — treat as labeled context, verify against code

---

## Step 0: Environment Detection (do this before anything else)

```bash
ls docker-compose.yml docker-compose.yaml compose.yml compose.yaml 2>/dev/null
```

If any file exists, determine the PHP service name (`web`, `app`, `php`, `shop`) and prefix **every** subsequent command:

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

**Preferred** (if `shopware-cli` is available inside the container):

```bash
mkdir -p tasks/<target_version>
docker compose exec <service> shopware-cli project dump \
  --output tasks/<target_version>/<current_version>_dump.sql \
  --compression gzip
```

**Fallback** (if `shopware-cli` is not installed — common in self-hosted projects):

```bash
mkdir -p tasks/<target_version>
docker compose exec <db-service> mysqldump -u <user> -p<pass> <dbname> \
  | gzip > tasks/<target_version>/<current_version>_dump.sql.gz
```

The PROCESS privilege warning about tablespace metadata is harmless — schema and data are captured.

### Phase 2 — Branch

**First, check for an existing upgrade branch** — the work may already exist but not yet be merged:

```bash
git branch -a | grep -i <ticket-or-version>
```

If a matching branch exists: surface it to the user and ask whether to continue from that branch or create a new one. Do **not** silently create a duplicate.

Check existing branch names for the naming convention. If the convention is unclear, **ask before creating the branch**.

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

**Check APP_ENV first** — determines which log file to inspect:

```bash
grep "^APP_ENV" .env .env.local 2>/dev/null | head -1
# dev → var/log/dev.log   |   prod → var/log/prod.log
```

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

HTTP 200 alone is **not sufficient** — the admin can return 200 while showing raw snippet keys if the locales API is broken. Also verify visually (browser) or confirm the page title is "Login | Shopware Administration", not a raw key like `sw-login.general.mainMenuItemIndex`.

**If raw snippet keys appear in the admin** (e.g. `sw-login.index.headlineForm` instead of "Log in to Shopware"), the `/api/_admin/locales` response is likely corrupted by a PHP error injected by the Symfony dev toolbar. Diagnose with:

```bash
docker compose exec <service> curl -s "http://localhost:<port>/api/_admin/locales" | head -c 2000
# If output contains HTML after the JSON object: there is a PHP error being injected.
# The error name in the HTML identifies the class to fix.
```

**Log check** — use the env-appropriate log:

```bash
tail -n 100 var/log/<APP_ENV>.log | grep -E "ERROR|CRITICAL"
```

If any 5xx responses or CRITICAL log entries appear: **stop, report the errors to the user, do not proceed to Phase 14.**

### Phase 14 — Deployment Checklist

→ **Read `references/deployment-checklist.md`** for the full template and QA Notes rules. Write the output to `tasks/<target_version>/deployment-checklist.md`. Tailor the QA Notes based on upgrade type and everything that changed (plugins bumped, recipes applied, API fixes made).

---

## Reference Map

- [`references/compatibility-check.md`](references/compatibility-check.md) — upgrade-check output interpretation, proprietary plugin handling, version-mismatch warnings
- [`references/version-bump.md`](references/version-bump.md) — which packages to update, path-repo constraint widening, store plugin major bumps
- [`references/build.md`](references/build.md) — deployment-helper run, failure handling (messenger, OrbStack), skip flags, removed-API scan
- [`references/flex-recipes.md`](references/flex-recipes.md) — TTY requirement, apply-one-at-a-time rule, commit-between workflow
- [`references/deployment-checklist.md`](references/deployment-checklist.md) — full checklist template, QA Notes rules

## Model Usage

Use a **low-cost model** (Haiku) for subagents that only run shell commands (backup, composer, cache:clear). Reserve Sonnet/Opus for decision points: compatibility review, infrastructure gate, recipe diffs.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Running commands on host when Docker is present | Always check for `docker-compose.yml`, `docker-compose.yaml`, `compose.yml`, or `compose.yaml` first |
| Assuming `shopware-cli` is available | Confirm it inside the container before Phase 1; fall back to `mysqldump` via the DB service if absent |
| Creating a new upgrade branch without checking for an existing one | Run `git branch -a \| grep <ticket>` first — a prior attempt may already have the version bump and recipes |
| Running `composer recipes:update` with uncommitted tracked files | The command needs a clean index — stash the modified tracked files (`git stash push -- path/to/file`), run the recipe, then `git stash pop` |
| Leaving `shopware.paas-meta.updates-for-deleted-files.patch` in the project root | If `.platform/applications.yaml` doesn't exist, paas-meta recipe update writes this stale patch file — delete it before committing |
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
| Treating HTTP 200 on /admin as a passing smoke test | The admin can return 200 while broken — visually confirm translated labels or check the page title is "Login \| Shopware Administration" |
| Tailing prod.log when APP_ENV=dev | Check `.env` for APP_ENV first — use dev.log in dev mode, prod.log in production |
| Type-hinting SalesChannelContextService (concrete) in plugin constructors | In 6.7 the Commercial/B2B packages inject a decorator that implements the interface but does not extend the concrete class — always use `SalesChannelContextServiceInterface` |
| Diagnosing raw snippet keys as a JS bundle problem | Raw snippet keys mean the `/api/_admin/locales` JSON parse failed — curl the endpoint directly and look for HTML injected after the JSON by the Symfony dev toolbar |

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
- **Never skip Phase 15 (lessons)** — every upgrade produces learnings; capture them before closing the branch.
- **Reference files are required context, not optional extras** — read the relevant one before acting on its phase, not after.
- **Never treat upgrade-check output as a final verdict** — always cross-reference private/path-repo plugins against their own `composer.json`.
- **Install `shopware-cli` if absent at Step 0** — Phase 4 requires it; skipping requires explicit user acknowledgement.

## Signal → Reference Routing

| Task signal | Always read | Also read if |
|---|---|---|
| compatibility, upgrade-check, plugin warnings, "Not compatible" | `compatibility-check.md` | `version-bump.md` if major upgrade |
| composer.json, version constraint, path-repo, store plugin, constraint conflict | `version-bump.md` | `compatibility-check.md` if warnings are ambiguous |
| build, deployment-helper, OrbStack, messenger, removed API | `build.md` | `version-bump.md` if major upgrade broke APIs |
| symfony recipes, flex, recipe update, recipe out of date | `flex-recipes.md` | — |
| smoke test, 5xx responses, raw snippet keys, HTTP 200 but broken admin | `smoke-test.md` | `deployment-checklist.md` once passing |
| deployment checklist, QA notes | `deployment-checklist.md` | — |
| shopware-cli not found, install options, Docker detection | `environment-setup.md` | — |

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

Confirm `shopware-cli` is available:

```bash
docker compose exec <service> shopware-cli version   # Docker
shopware-cli version                                  # no Docker
```

→ **If `shopware-cli` is not found, read `references/environment-setup.md`** for install options and the mysqldump fallback decision.

## Upgrade Phases

Use **TaskCreate** to create one todo per phase. Mark each complete as you finish it.

### Phase 1 — Backup

```bash
mkdir -p tasks/<target_version>
docker compose exec <service> shopware-cli project dump \
  --output tasks/<target_version>/<current_version>_dump.sql \
  --compression gzip
```

→ **If `shopware-cli` is unavailable, use the mysqldump fallback in `references/environment-setup.md`.**

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

Verify storefront and administration are responding. **Do not proceed to Phase 14 until this phase passes.**

→ **Read `references/smoke-test.md`** for the full verification workflow: APP_ENV detection, curl checks, raw snippet key diagnosis, log triage, and browser visual confirmation.

### Phase 14 — Deployment Checklist

→ **Read `references/deployment-checklist.md`** for the full template and QA Notes rules. Write the output to `tasks/<target_version>/deployment-checklist.md`. Tailor the QA Notes based on upgrade type and everything that changed (plugins bumped, recipes applied, API fixes made).

### Phase 15 — Compile Lessons and Update Skill

After every upgrade, close the loop. This phase must always run — even if the upgrade was routine.

→ **Read `references/lessons-workflow.md`** for the full workflow: what to capture, the `tasks/lessons.md` format, how to propose skill improvements, and how to mark the upgrade complete.

---

## Reference Map

- [`references/environment-setup.md`](references/environment-setup.md) — shopware-cli install options, Docker detection, mysqldump fallback decision
- [`references/compatibility-check.md`](references/compatibility-check.md) — upgrade-check output interpretation, proprietary plugin handling, version-mismatch warnings
- [`references/version-bump.md`](references/version-bump.md) — which packages to update, path-repo constraint widening, store plugin major bumps
- [`references/build.md`](references/build.md) — deployment-helper run, failure handling (messenger, OrbStack), skip flags, removed-API scan
- [`references/flex-recipes.md`](references/flex-recipes.md) — TTY requirement, apply-one-at-a-time rule, commit-between workflow
- [`references/smoke-test.md`](references/smoke-test.md) — APP_ENV detection, curl checks, raw snippet key diagnosis, log triage, browser visual verification
- [`references/deployment-checklist.md`](references/deployment-checklist.md) — full checklist template, QA Notes rules
- [`references/lessons-workflow.md`](references/lessons-workflow.md) — lessons format, skill improvement proposals, upgrade completion checklist
- [`references/common-mistakes.md`](references/common-mistakes.md) — full 27-row common mistakes reference

## Model Usage

Use a **low-cost model** (Haiku) for subagents that only run shell commands (backup, composer, cache:clear). Reserve Sonnet/Opus for decision points: compatibility review, infrastructure gate, recipe diffs.

## Common Mistakes

→ **See `references/common-mistakes.md` for the full list.** Critical items inline:

| Mistake | Fix |
|---|---|
| Running commands on host when Docker is present | Always check for compose file first — every command runs inside the container |
| Assuming `shopware-cli` is available | Confirm at Step 0; read `references/environment-setup.md` if absent |
| Treating "Not compatible" as a hard blocker for private plugins | upgrade-check queries Store API — private plugins always show "Not compatible"; read the plugin's own `composer.json` |
| Trusting upgrade-check warnings for a patch upgrade | upgrade-check targets latest major — warnings are irrelevant for same-minor patch upgrades |
| Treating HTTP 200 on /admin as a passing smoke test | Admin can return 200 while broken — read `references/smoke-test.md` for full verification |
| Type-hinting SalesChannelContextService (concrete) in plugin constructors | In 6.7+ B2B decorators don't extend the concrete class — always use `SalesChannelContextServiceInterface`; PHPStan misses this, only surfaces at runtime |
| `sw_extends` on old navigation path causes persistent OOM | `storefront/layout/navigation/navigation.html.twig` removed in 6.7 — creates infinite template loop. Grep: `grep -rn "layout/navigation/navigation.html.twig" custom/static-plugins/ --include="*.twig"`. Migrate to `storefront/layout/navbar/navbar.html.twig` — see `references/common-mistakes.md` |

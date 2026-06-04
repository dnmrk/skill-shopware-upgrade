# shopware-upgrade

A Codex and Claude Code skill for upgrading Shopware 6 installations — patch, minor, or major.

## What it does

Guides the agent through a structured 15-phase upgrade workflow: backup, branch, vendor sync, compatibility check, maintenance window, infrastructure gate (major only), version bump, composer update, build, commit, flex recipes, smoke test, deployment checklist, and lessons capture.

Every phase references a dedicated knowledge file so the main skill stays lean while deep domain knowledge remains accessible on demand.

## Install And Use

### Installation

Copy the skill into your Codex skills directory:

```bash
cp -r shopware-upgrade ~/.codex/skills/
```

Or copy it into your Claude Code skills directory:

```bash
cp -r shopware-upgrade ~/.claude/skills/
```

Or clone the repo directly into either skills directory:

```bash
git clone <repo-url> ~/.codex/skills/shopware-upgrade
git clone <repo-url> ~/.claude/skills/shopware-upgrade
```

Codex auto-discovers skills in `~/.codex/skills/`; Claude Code auto-discovers skills in `~/.claude/skills/`.

### Usage

Invoke the skill by mentioning an upgrade intent in your prompt, or trigger it explicitly:

```
upgrade Shopware to 6.7.1
```

```
bump shopware/core to ~6.6.0 — compatibility unknown
```

Codex or Claude Code will invoke the `shopware-upgrade` skill automatically when upgrade signals are detected, or you can reference it directly:

> "Use the shopware-upgrade skill to upgrade this project from 6.5 to 6.6."

In Codex, the bundled UI metadata also supports direct invocation:

> "Use $shopware-upgrade to upgrade this Shopware project to the target version."

### What happens

The agent runs through all 15 phases, tracking each as a visible task so you can see progress at a glance and resume if interrupted.

| Phase | Name | Notes |
|---|---|---|
| 1 | Backup | `shopware-cli project dump` (auto-installs cli if absent); falls back to `mysqldump` |
| 2 | Branch | Checks for an existing upgrade branch before creating a new one |
| 3 | Vendor sync | `composer install` to align `vendor/` with the current lock file before bumping |
| 4 | Compatibility check | `shopware-cli project upgrade-check`; private/path-repo plugins verified against their own `composer.json` |
| 5 | Maintenance on | Enables maintenance mode on all sales channels |
| 6 | Infrastructure gate | **Major upgrades only** — verify PHP, Redis/Valkey, and OpenSearch compatibility before proceeding |
| 7 | Version bump | Edit `composer.json`; widen path-repo plugin constraints for major upgrades |
| 8 | Composer update | `composer update --no-scripts` |
| 9 | Build | `shopware/deployment-helper` if installed; otherwise manual chain (`migrate`, `dal:refresh:index`, `assets:install`, `cache:clear`, `theme:compile`) |
| 10 | Commit | Stage `composer.json`, `composer.lock`, `symfony.lock`, and any plugin constraint changes |
| 11 | Flex recipes | `composer recipes` — apply any pending updates one at a time, commit between each |
| 12 | Maintenance off | Disables maintenance mode on all sales channels |
| 13 | Smoke test | HTTP checks on storefront and admin; visual browser verification; log triage |
| 14 | Deployment checklist | Written to `tasks/<version>/deployment-checklist.md` with tailored QA Notes |
| 15 | Lessons | Captured to `tasks/lessons.md`; skill improvement proposals surfaced to the user |

---

## When to use

Invoke when any of these are true:

- A target Shopware version is specified
- `composer.json` version constraints need updating
- Extension or plugin compatibility is unknown after a bump
- Flex recipes are out of date
- Deprecated or removed APIs need remediating
- `upgrade-check` warnings need interpretation
- A maintenance window is required for the upgrade

## Skill structure

```
shopware-upgrade/
  SKILL.md                          # Orchestration hub — phases, routing, invariants
  agents/
    openai.yaml                     # Codex UI metadata
  references/
    compatibility-check.md          # Phase 4 — upgrade-check output, private plugins
    version-bump.md                 # Phase 7 — package constraints, path-repo widening
    build.md                        # Phase 9 — deployment-helper, OrbStack, API fixes
    flex-recipes.md                 # Phase 11 — TTY, commit-between workflow
    smoke-test.md                   # Phase 13 — storefront/admin verification
    deployment-checklist.md         # Phase 14 — checklist template, QA Notes rules
    lessons-workflow.md             # Phase 15 — lessons capture and skill updates
  evals/
    evals.json                      # Eval scenarios for testing skill compliance
```

## Architecture

The skill uses a two-layer design:

- **`SKILL.md`** is a traffic controller: hard constraints, signal → reference routing table, mode-specific behavior, cross-cutting invariants, and phase orchestration.
- **Reference files** carry the deep domain knowledge for complex phases. They are lazy-loaded — the agent reads a reference only when it reaches the relevant phase.

This keeps the main skill fast to parse while keeping full upgrade knowledge accessible.

## Key invariants

- Every command runs inside the Docker container — detect Docker before anything else.
- Never skip Phase 4 (compatibility check) — extensions break silently without it.
- Never skip Phase 11 (Flex recipes) after the version bump.
- `upgrade-check` output is a signal, not a verdict — always verify private plugins against their own `composer.json`.
- Phase 14 (deployment checklist) only runs after Phase 13 (smoke test) passes.
- Never skip Phase 15 (lessons) before closing the branch.

## Requirements

- `shopware-cli` available inside the container (or `mysqldump` fallback for backup)
- Composer 2.x
- Docker Compose (if containerised — detected automatically)

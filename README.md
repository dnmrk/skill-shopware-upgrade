# shopware-upgrade

A Claude Code skill for upgrading Shopware 6 installations — patch, minor, or major.

## What it does

Guides Claude through a structured 14-phase upgrade workflow: backup, branch, vendor sync, compatibility check, maintenance window, infrastructure gate (major only), version bump, composer update, build, commit, flex recipes, smoke test, and deployment checklist.

Every phase references a dedicated knowledge file so the main skill stays lean while deep domain knowledge remains accessible on demand.

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
  references/
    compatibility-check.md          # Phase 4 — upgrade-check output, private plugins
    version-bump.md                 # Phase 7 — package constraints, path-repo widening
    build.md                        # Phase 9 — deployment-helper, OrbStack, API fixes
    flex-recipes.md                 # Phase 11 — TTY, commit-between workflow
    deployment-checklist.md         # Phase 14 — checklist template, QA Notes rules
  evals/
    evals.json                      # Eval scenarios for testing skill compliance
```

## Architecture

The skill uses a two-layer design:

- **`SKILL.md`** is a traffic controller: hard constraints, signal → reference routing table, mode-specific behavior, cross-cutting invariants, and phase orchestration.
- **Reference files** carry the deep domain knowledge for complex phases. They are lazy-loaded — Claude reads a reference only when it reaches the relevant phase.

This keeps the main skill fast to parse while keeping full upgrade knowledge accessible.

## Key invariants

- Every command runs inside the Docker container — detect Docker before anything else.
- Never skip Phase 4 (compatibility check) — extensions break silently without it.
- `upgrade-check` output is a signal, not a verdict — always verify private plugins against their own `composer.json`.
- Phase 14 (deployment checklist) only runs after Phase 13 (smoke test) passes.

## Requirements

- `shopware-cli` available inside the container (or `mysqldump` fallback for backup)
- Composer 2.x
- Docker Compose (if containerised — detected automatically)

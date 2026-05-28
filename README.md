# shopware-upgrade

A Claude Code skill for upgrading Shopware 6 installations — patch, minor, or major.

## What it does

Guides Claude through a structured 14-phase upgrade workflow: backup, branch, vendor sync, compatibility check, maintenance window, infrastructure gate (major only), version bump, composer update, build, commit, flex recipes, smoke test, and deployment checklist.

Every phase references a dedicated knowledge file so the main skill stays lean while deep domain knowledge remains accessible on demand.

## Install And Use

### Installation

Copy the skill into your Claude Code skills directory:

```bash
cp -r shopware-upgrade ~/.claude/skills/
```

Or clone the repo directly into `~/.claude/skills/`:

```bash
git clone <repo-url> ~/.claude/skills/shopware-upgrade
```

Claude Code auto-discovers skills in `~/.claude/skills/` — no further configuration needed.

### Usage

Invoke the skill by mentioning an upgrade intent in your prompt, or trigger it explicitly:

```
upgrade Shopware to 6.7.1
```

```
bump shopware/core to ~6.6.0 — compatibility unknown
```

Claude will invoke the `shopware-upgrade` skill automatically when upgrade signals are detected, or you can reference it directly:

> "Use the shopware-upgrade skill to upgrade this project from 6.5 to 6.6."

### What happens

Claude runs through up to 14 phases interactively — stopping at decision points (backup confirmation, maintenance window, deployment approval) and asking before irreversible actions. Each phase is tracked as a visible task so you can see progress and resume if interrupted.

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

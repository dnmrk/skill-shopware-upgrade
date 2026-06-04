# Changelog

All notable changes to the `shopware-upgrade` skill.

## [1.3.0] — 2026-06-04

### Added

- README: full 15-phase table with a one-line description per phase and notes on conditions (major-only gate, fallback behaviour, commit-between rule).

### Changed

- Step 0: `shopware-cli` is now **auto-installed inside the container** when absent — no user prompt, no Phase 4 skip. The install script detects architecture (`uname -m`), maps to the correct GitHub release binary (`arm64` / `amd64`), downloads via `curl`, and installs to `/var/www/html/shopware-cli` (writable by `www-data`; `/usr/local/bin` is not). Install is invoked via full path for the remainder of the upgrade.
- `environment-setup.md`: replaced the vague "check the image's package manager" container install note with the exact auto-install script and path rationale.

### Fixed

- Step 0 version check used `shopware-cli version` (invalid command — exits with `FATAL unknown command "version"`). Corrected to `shopware-cli --version` in both `SKILL.md` and `environment-setup.md`.

---

## [1.2.0] — 2026-06-04

### Added

- Codex support through `agents/openai.yaml` with display metadata and a `$shopware-upgrade` default prompt.

### Changed

- Progress tracking instructions now support Codex `update_plan` and Claude Code `TaskCreate`.
- README installation and usage instructions now cover both Codex and Claude Code.
- Replaced model-vendor-specific subagent guidance with portable subagent usage guidance.
- Deployment checklist template now uses an agent-neutral prepared-by label.

## [1.1.0] — 2026-05-26

### Added

- **Hard Constraints block** — five explicit non-negotiables at the top of SKILL.md covering Docker detection, skipping the compat check, premature checklist generation, and treating upgrade-check output as authoritative.
- **Signal → Reference Routing table** — keyword-based routing with an "Also read if" column for cross-cutting cases (e.g. major upgrade triggers both `compatibility-check.md` and `version-bump.md`).
- **Mode-Specific Behavior table** — explicit behavioral contracts for Planning, Executing, and Post-upgrade review modes.
- **Cross-Cutting Invariants section** — six always/never rules promoted out of the Common Mistakes table; too important to be buried.
- **Source Trust Hierarchy** — explicit epistemological ladder: `composer.json` > changelog > plugin composer.json > upgrade-check > docs/blogs.
- **Reference Map** — replaces the plain reference list with scope descriptions per entry.
- `allowed-tools` frontmatter field (`Read`, `Glob`, `Grep`).
- Richer `description` field with additional trigger keywords: flex recipes out of date, deprecated or removed APIs, version bump blocked by constraint conflicts, upgrade-check warnings need interpretation, migration required after version change.

### Changed

- Phase 1 (Backup) now includes a `mysqldump` fallback for projects without `shopware-cli`.
- Phase 2 (Branch) now checks for an existing upgrade branch before creating a new one — avoids silent duplicate branches.
- Step 0 docker-compose detection now checks `compose.yaml` in addition to `docker-compose.yml` / `compose.yml`.

## [1.0.0] — 2026-05-25

### Added

- Initial release: 14-phase upgrade workflow (backup → deployment checklist).
- Reference file architecture: `compatibility-check.md`, `version-bump.md`, `build.md`, `flex-recipes.md`, `deployment-checklist.md`.
- Docker-first environment detection at Step 0.
- Common Mistakes table covering the most frequent upgrade errors.
- Model Usage guidance: Haiku for shell-only subagents, Sonnet/Opus for decision points.
- Eval scenarios in `evals/evals.json`.

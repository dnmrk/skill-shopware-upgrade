# Changelog

All notable changes to the `shopware-upgrade` skill.

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

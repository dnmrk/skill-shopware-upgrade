# Phase 4 — Compatibility Check Reference

## Running the check

```bash
# Requires interactive TTY — do NOT add -T flag
docker compose exec <service> shopware-cli project upgrade-check --no-interaction
```

## Warning: upgrade-check auto-selects the latest Shopware version

`upgrade-check` always evaluates compatibility against the **latest available Shopware version**, not your target. For a patch upgrade (e.g. 6.6.10.x → 6.6.10.y), any "Not compatible" warnings shown are for a newer major and are completely irrelevant. Only treat incompatibility warnings as blockers if you are actually targeting that version.

## Handling proprietary and private-registry plugins

`upgrade-check` determines compatibility by querying the Shopware Store API. Plugins not published to the public store will always show "Not compatible" or "Not available in Store" — this is not a real incompatibility signal.

**Always verify by reading the plugin's own `composer.json` require constraints directly** before treating the flag as a blocker:

```bash
cat vendor/<vendor>/<package>/composer.json | grep -A2 '"require"'
# Or if it's a path-repo plugin:
cat custom/static-plugins/<PluginName>/composer.json | grep -A2 '"require"'
```

A constraint like `^6.6 || ^6.7` or `^6.7` means the plugin is already compatible. A `~6.6.0` or `<6.7` constraint is the real blocker.

## Decision after reviewing output

| Situation | Action |
|---|---|
| Extension incompatible with target version (per store API) AND public plugin | Stop and report to user before continuing |
| Extension flagged "Not compatible" but is a private/path-repo plugin | Read the plugin's own composer.json — do not treat store API result as authoritative |
| Warnings are for a newer major than your target | Ignore — irrelevant for patch/minor upgrades |

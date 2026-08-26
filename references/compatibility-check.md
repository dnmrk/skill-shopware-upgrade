# Phase 4 — Compatibility Check Reference

## Running the check

```bash
# Requires interactive TTY — do NOT add -T flag
docker compose exec <service> shopware-cli project upgrade-check --no-interaction
```

## Warning: upgrade-check auto-selects the latest Shopware version

`upgrade-check` evaluates compatibility against the **latest available Shopware version**, which is not necessarily your target. It prints which one it picked:

```
INFO  Auto selected version 6.7.12.2
```

**Read that line and compare it to your target before discounting anything:**

| Auto-selected vs target | Meaning |
|---|---|
| **Same** (your target *is* the latest release) | Warnings are **directly relevant** — act on them. This is common when upgrading to the newest patch. |
| **Newer major/minor than your target** | Warnings describe a version you are not going to — irrelevant, discount them. |

Do not apply the blanket rule "patch-upgrade warnings are always irrelevant". When you are moving to the current latest patch, the auto-selected version and your target coincide and the output is real signal.

## Handling proprietary and private-registry plugins

`upgrade-check` determines compatibility by querying the Shopware Store API. Plugins not published to the public store will always show "Not compatible" or "Not available in Store" — this is not a real incompatibility signal.

**Always verify by reading the plugin's own `composer.json` require constraints directly** before treating the flag as a blocker:

```bash
cat vendor/<vendor>/<package>/composer.json | grep -A2 '"require"'
# Or if it's a path-repo plugin:
cat custom/static-plugins/<PluginName>/composer.json | grep -A2 '"require"'
```

A constraint like `^6.6 || ^6.7` or `^6.7` means the plugin is already compatible. A `~6.6.0` or `<6.7` constraint is the real blocker.

### "Not available in Store" ≠ "Not compatible"

These are two different verdicts and only one of them can ever block:

| Verdict | Meaning | Blocker? |
|---|---|---|
| **"Not available in Store"** | The Store API lookup found nothing. Normal and expected for *every* path-repo / private plugin. | **Never** |
| **"Not compatible"** | The plugin *was* found and your target exceeds its declared max version. | **Possibly** — verify against the plugin's own `composer.json`, since Store metadata lags vendor releases |

## Decision after reviewing output

| Situation | Action |
|---|---|
| Extension incompatible with target version (per store API) AND public plugin | Stop and report to user before continuing |
| Extension flagged "Not compatible" but is a private/path-repo plugin | Read the plugin's own composer.json — do not treat store API result as authoritative |
| Warnings are for a newer major than your target | Ignore — irrelevant for patch/minor upgrades |

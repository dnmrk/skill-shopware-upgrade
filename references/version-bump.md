# Phase 7 — Version Bump Reference

## Editing composer.json

Update `shopware/core` (or `shopware/platform`) to the target constraint. Also update `shopware/storefront`, `shopware/administration`, and `shopware/elasticsearch` **if and only if** they appear as explicit top-level deps with a pinned constraint (i.e. not `"*"`). Scan **all** `shopware/*` entries — don't assume it's only those three.

```json
"shopware/core": "~6.7.0",
"shopware/storefront": "~6.7.0",
"shopware/administration": "~6.7.0"
```

Do not touch other packages.

## Major upgrades: path-repo plugins

Widen `shopware/core` in every path-repo plugin's own `composer.json`. Even though Composer prefers path repos, the constraint still blocks resolution.

```bash
# Find all path-repo plugins with pinned constraints
grep -rn '"shopware/core"' custom/static-plugins/ --include="composer.json"
```

Change `~6.6.0` → `^6.6 || ^6.7` in each hit. This is a widening — it keeps the plugin usable on both the old and new version.

## Major upgrades: store plugins with a hard version cap

Run `composer show <package> --all` to find the 6.7-compatible release, then bump the constraint. Common packages:

| Package | Old constraint | New constraint |
|---------|---------------|----------------|
| `store.shopware.com/swagcommercial` | `^6.10` | `^7.0` |
| `store.shopware.com/swagplatformsecurity` | `^3.0` | `^4.0` |
| Third-party themes (e.g. ThemeWare) | `^3.7` | `^4.0` |

These are examples — always verify the actual compatible version with `composer show`.

## Theme major bumps are breaking

A theme constraint bump (e.g. ThemeWare 3.x → 4.x) is a breaking change. Flag this to the user: **template overrides in the project need auditing after the upgrade** — block names, template paths, and JS plugin signatures may have changed.

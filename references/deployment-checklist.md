# Phase 14 — Deployment Checklist Reference

## Output file

Write to: `tasks/<target_version>/deployment-checklist.md`

## Checklist template

Fill in all placeholders. The QA Notes section must be tailored — do not leave it generic.

```markdown
# Deployment Checklist — Shopware <target_version>

**Upgrade path:** <current_version> → <target_version>  
**Branch:** <branch-name>  
**Prepared by:** Claude Code  
**Date:** <YYYY-MM-DD>

---

## Pre-Deployment

- [ ] Database backup confirmed at `tasks/<target_version>/<current_version>_dump.sql`
- [ ] Branch `<branch-name>` reviewed and approved
- [ ] All extension compatibility warnings resolved
- [ ] Maintenance window communicated to stakeholders

## Deployment Steps

- [ ] Enable maintenance mode: `bin/console sales-channel:maintenance:enable --all`
- [ ] Pull branch to production
- [ ] Run `composer install --no-dev --optimize-autoloader`
- [ ] Run deployment helper or manual build chain (see build reference)
- [ ] Confirm no CRITICAL errors in `var/log/prod.log`
- [ ] Smoke-test storefront (HTTP 200)
- [ ] Smoke-test administration (HTTP 200)
- [ ] Disable maintenance mode: `bin/console sales-channel:maintenance:disable --all`

## Rollback Procedure

1. Re-enable maintenance mode
2. Restore DB: `shopware-cli project import tasks/<target_version>/<current_version>_dump.sql.gz`
3. Checkout previous commit: `git checkout <previous_commit>`
4. `composer install --no-dev --optimize-autoloader`
5. `bin/console cache:clear`
6. Disable maintenance mode

---

## QA Notes

<!-- Tailored content goes here — see rules below -->
```

## QA Notes content rules

Base coverage by upgrade type:

| Upgrade type | Minimum testing scope |
|---|---|
| Patch (x.y.Z) | Regression test features in the Shopware changelog for that patch; payment flows if `shopware/core` touched checkout |
| Minor (x.Y.z) | All deprecated-and-removed APIs; any admin modules rebuilt; storefront JS plugin changes |
| Major (X.y.z) | Full checkout flow; B2B Components if used; all custom admin modules; custom Twig template blocks; Rule Builder conditions; any plugins that had constraint bumps |

Also add a specific line item for each of:
- **Plugin with bumped constraint** — name the plugin, old/new version, which features it owns in this project
- **Flex recipe applied** — which config file changed and what the functional impact is
- **API fix made** (e.g. `getDefinition()` → `getEntityName()`) — list affected write commands and a test that exercises them
- **Theme major bump** — note that Twig block names and JS plugin signatures may have changed; full storefront visual regression recommended

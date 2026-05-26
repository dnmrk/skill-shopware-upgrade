# Phase 9 — Build Reference

## Check for deployment-helper

```bash
composer show shopware/deployment-helper 2>/dev/null
```

## Major upgrades: scan for removed APIs before building

```bash
# WriteCommand::getDefinition() was removed in Shopware 6.7
grep -rn "->getDefinition()" custom/static-plugins/ --include="*.php"
```

Fix any hits: `$command->getDefinition()::class === SomeDefinition::class` becomes `$command->getEntityName() === SomeDefinition::ENTITY_NAME`.

```bash
# SalesChannelContextService concrete type breaks with swagcommercial 7.x B2B decorators
# PHPStan does NOT catch this — only surfaces at runtime when the DI container wires the decorator
grep -rn "SalesChannelContextService[^I]" custom/static-plugins/ --include="*.php" | grep -v Parameters
```

Fix any hits: change `SalesChannelContextService $x` to `SalesChannelContextServiceInterface $x` and update the `use` import accordingly.

## If deployment-helper is installed

```bash
vendor/bin/shopware-deployment-helper run
```

**If it fails on `messenger:setup-transports`:** There is no `--skip-messenger` flag. Fix the underlying connectivity issue — don't add a flag that doesn't exist.

- In OrbStack setups: the messaging container (LavinMQ/RabbitMQ) may not have its `.orb.local` DNS domain registered yet.
  ```bash
  docker compose up -d --force-recreate <lavinmq|rabbitmq>
  bin/console cache:clear
  # Then retry
  vendor/bin/shopware-deployment-helper run
  ```

Available skip flags (actual flags, not made up):
- `--skip-theme-compile`
- `--skip-asset-install` (also accepted as `--skip-assets-install`)

## If deployment-helper is NOT installed

Run the build chain manually:

```bash
bin/console database:migrate --all
bin/console dal:refresh:index
bin/console assets:install
bin/console cache:clear
bin/console theme:compile
```

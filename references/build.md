# Phase 9 — Build Reference

## Vendor install mode — source checkouts abort the upgrade

Checked at Step 0, but the consequences land in Phase 8/9, so the detail lives here.

```bash
docker compose exec <service> find vendor -maxdepth 3 -name .git
```

Any `vendor/shopware/*` hit means that package was installed from **source** (a git clone) rather than **dist** (a zip). Composer will then refuse to update it if the working tree is dirty:

```
In VcsDownloader.php line 268:
  Source directory /var/www/html/vendor/shopware/storefront has uncommitted changes.
```

The tree is nearly always dirty, for two reasons that have nothing to do with anyone editing `vendor/`:

- `bin/build-administration.sh` and `bin/build-storefront.sh` compile bundle assets **into** `vendor/shopware/*/Resources/public/`, and emit `.js.map` files plus appended `//# sourceMappingURL=` lines.
- macOS/OrbStack bind mounts flip file modes `100644 → 100755`, which git reports as modifications with **zero** content change.

Confirm the changes are disposable before discarding — `git -C vendor/shopware/<pkg> diff --stat` showing only mode changes and `1 +` insertions on minified assets means there is nothing to preserve.

**Why this is a hard blocker rather than a nuisance:** the exception fires during the *install* step, i.e. **after** Composer has already rewritten `composer.lock`. The result is a split-version vendor tree — `shopware/core` on the new version, `storefront` / `administration` / `elasticsearch` stranded on the old one — while the lock reads as fully correct. `bin/console --version` reports core only, so it also looks fine.

**Fix at root cause, not symptom:**

```bash
# NOT just cleaning the dirty files — that leaves the checkout in place to fail next time
docker compose exec <service> rm -rf vendor/shopware/core vendor/shopware/storefront \
  vendor/shopware/administration vendor/shopware/elasticsearch
docker compose exec <service> composer install --prefer-dist --no-scripts
```

Verify afterwards: `find vendor/shopware -maxdepth 2 -name .git` returns nothing, and each package's version matches the lock. Check `composer config preferred-install` too — it is frequently already `dist`, meaning the source checkouts were contradicting the project's own declared intent. Once on dist, asset builds writing into `vendor/` are invisible to Composer and this failure class is gone permanently.

Note that other developers' machines may still carry source checkouts; their next upgrade will abort the same way.

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

```bash
# storefront/layout/navigation/navigation.html.twig was removed in 6.7
# Plugins still extending this path create an infinite sw_extends loop → persistent OOM on every request
grep -rn "layout/navigation/navigation.html.twig" custom/static-plugins/ --include="*.twig"
```

Fix any hits: delete the plugin override of the old path and create a new `storefront/layout/navbar/navbar.html.twig` that extends `@Storefront/storefront/layout/navbar/navbar.html.twig`, overriding `layout_navbar_nav_element` or `layout_navbar_menu_items` with the custom nav logic. Also audit header templates for `sw_include` calls to the old navigation path and remove them — `layout_header_navigation` in ThemeWare 6.7 already renders `navbar.html.twig`.

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

## Verify the build actually completed

`tail` on the deployment-helper output hides the middle of the run. Verify outcomes directly rather than trusting the visible portion:

```bash
bin/console plugin:list                      # expect "0 upgradeable" and all plugins active
bin/console database:migrate --all core      # idempotent; expect "Migrated 0 out of 0"
bin/console debug:messenger                  # transports wired
```

`plugin:list` reporting `0 upgradeable` is the strongest single signal that deployment-helper's plugin-update step ran, not just the theme compile you can see at the tail.

## Verify plugin code still type-checks against the new core

The greps above catch *known* removed-API traps by name. This catches the ones nobody has documented yet — a core bump can change signatures no grep anticipates.

If the project has PHPStan configured, run it after the build and split the errors by location:

```bash
./vendor/bin/phpstan analyze custom/static-plugins --no-progress --error-format=json \
  | php -r '$j=json_decode(stream_get_contents(STDIN),true);
      foreach($j["files"]??[] as $f=>$d)
        printf("%-5s %-3d %s\n", preg_match("#/[Tt]ests?/#",$f)?"TEST":"SRC", count($d["messages"]), $f);'
```

Errors under `tests/` are usually a pre-existing baseline — commonly `getContainer()->get()` returning untyped `object`, which trips level 8 on `argument.type` and `assign.propertyType`. **Any error under production `src/` after a version bump is a genuine core-API break** — investigate before Phase 10.

Record the baseline count and the test-vs-src split in the deployment checklist so the next upgrade can diff against it instead of re-deriving what "normal" looks like.

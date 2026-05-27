# Phase 13 — Smoke Test Reference

Verify storefront and administration are responding. Do not proceed to Phase 14 until all checks pass.

## 1. Determine APP_ENV

```bash
grep "^APP_ENV" .env .env.local 2>/dev/null | head -1
# dev → var/log/dev.log   |   prod → var/log/prod.log
```

## 2. Storefront check

Get all active sales channel domains:

```bash
bin/console sales-channel:list
```

Then curl each storefront URL:

```bash
curl -o /dev/null -s -w "%{http_code}" <storefront_url>
# Expected: 200. Anything 5xx means the build or migration failed silently.
```

If you cannot determine the storefront URL, ask the user before proceeding.

## 3. Administration check

```bash
curl -o /dev/null -s -w "%{http_code}" <admin_url>/admin
# Expected: 200. A 404 means admin assets weren't built or installed correctly.
```

HTTP 200 alone is **not sufficient** — the admin can return 200 while showing raw snippet keys if the locales API is broken. Verify visually or confirm the page title is "Login | Shopware Administration", not a raw key like `sw-login.general.mainMenuItemIndex`.

## 4. Diagnose raw snippet keys in the admin

If snippet keys appear (e.g. `sw-login.index.headlineForm` instead of "Log in to Shopware"), the `/api/_admin/locales` JSON parse failed — likely HTML injected by the Symfony dev toolbar:

```bash
docker compose exec <service> curl -s "http://localhost:<port>/api/_admin/locales" | head -c 2000
# If output contains HTML after the JSON object: a PHP error is being injected.
# The error class name in the HTML identifies what to fix.
```

## 5. Log check

```bash
tail -n 100 var/log/<APP_ENV>.log | grep -E "ERROR|CRITICAL"
```

If any 5xx responses or CRITICAL log entries appear: **stop, report the errors to the user, do not proceed to Phase 14.**

**OOM at `Template.php` — distinguish transient from persistent:**

- **Transient (not a real error):** First request after `cache:clear` exhausts memory while Twig simultaneously compiles all plugin templates in the debug container. Signal: `OutOfMemoryError` is the *only* CRITICAL entry and **subsequent requests return 200**. The smoke test passes — do not investigate further.
- **Persistent (real error — circular template loop):** OOM appears on **every** storefront or ESI request and is accompanied by `Maximum call stack size of ... bytes reached. Infinite recursion?`. A plugin's `sw_extends` chain has no terminating base template. In 6.7 the most common cause is a plugin still extending `storefront/layout/navigation/navigation.html.twig` (removed in 6.7 — moved to `storefront/layout/navbar/navbar.html.twig`). Without a core base to terminate the chain, Shopware's template resolver wraps remaining plugin overrides in an infinite loop. **Grep:** `grep -rn "layout/navigation/navigation.html.twig" custom/static-plugins/ --include="*.twig"`. Confirm the loop by inspecting compiled `.php` files in `var/cache/dev_*/twig/` — check `doGetParent()` return values for A→B→A circular references.

## 6. Visual browser verification

curl confirms HTTP status only — it cannot confirm the page renders correctly. Use `/browser` to visually confirm:

- **Storefront:** homepage loads with correct theme, no layout breakage, no raw Twig/snippet keys visible
- **Admin:** login page shows "Log in to Shopware Administration" (translated label, not `sw-login.index.headlineForm`)

```
/browser
```

Navigate to the storefront URL and admin URL. If the browser skill is not available, ask the user to confirm visually in their browser before marking Phase 13 complete.

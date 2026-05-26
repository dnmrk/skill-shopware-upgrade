# Step 0 — Environment Setup Reference

## Docker detection

```bash
ls docker-compose.yml docker-compose.yaml compose.yml compose.yaml 2>/dev/null
```

If any file exists, determine the PHP service name (`web`, `app`, `php`, `shop`) from the file and prefix **every** subsequent command:

```bash
docker compose exec <service> <command>
```

## Confirming shopware-cli

```bash
docker compose exec <service> shopware-cli version   # Docker
shopware-cli version                                  # no Docker
```

## Installing shopware-cli (if absent)

**macOS / Linux host (Homebrew):**
```bash
brew install shopware-ag/tap/shopware-cli
```

**Linux host (direct binary via Cloudsmith apt repo):**
```bash
curl -1sLf 'https://dl.cloudsmith.io/public/shopware/shopware-cli/setup.deb.sh' | sudo bash
sudo apt install shopware-cli
```

**Inside the container (if the image supports it):**
Check the image's package manager — or mount the host binary as a volume.

## Fallback decision

If `shopware-cli` cannot be installed inside the container, the mysqldump fallback covers Phase 1 (backup) only:

```bash
mkdir -p tasks/<target_version>
docker compose exec <db-service> mysqldump -u <user> -p<pass> <dbname> \
  | gzip > tasks/<target_version>/<current_version>_dump.sql.gz
```

The PROCESS privilege warning about tablespace metadata is harmless — schema and data are captured.

Phase 4 (`upgrade-check`) **still requires `shopware-cli`**. If it cannot be installed at all:
- Note the gap explicitly
- Skip Phase 4 only with explicit user acknowledgement
- Rely on manual plugin `composer.json` inspection for every extension

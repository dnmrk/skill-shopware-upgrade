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
docker compose exec <service> shopware-cli --version   # Docker
shopware-cli --version                                  # no Docker
```

## Installing shopware-cli (if absent)

**If running in Docker, install inside the container automatically** — do not ask the user:

```bash
# 1. Detect architecture and map to GitHub release name
ARCH=$(docker compose exec <service> uname -m)
# aarch64 → arm64 | x86_64 → amd64
case "$ARCH" in aarch64) GOARCH=arm64 ;; *) GOARCH=amd64 ;; esac

# 2. Download latest release into /tmp inside the container
docker compose exec <service> sh -c "
  curl -fsSL https://github.com/shopware/shopware-cli/releases/latest/download/shopware-cli_linux_${GOARCH}.tar.gz \
    -o /tmp/swcli.tar.gz && tar xzf /tmp/swcli.tar.gz -C /tmp
"

# 3. Install to the project root (writable by www-data; /usr/local/bin is not)
docker compose exec <service> sh -c "
  cp /tmp/shopware-cli /var/www/html/shopware-cli && chmod +x /var/www/html/shopware-cli
"

# 4. Verify
docker compose exec <service> /var/www/html/shopware-cli --version
```

After installation, invoke shopware-cli via its **full path** `/var/www/html/shopware-cli` for the rest of the upgrade — the binary is not on `$PATH`.

**Important:** This install is not persistent across container rebuilds. Note this in the deployment checklist and recommend a host-side install (`brew install shopware-ag/tap/shopware-cli`) for teams.

**If not running in Docker (host install):**

```bash
brew install shopware-ag/tap/shopware-cli          # macOS
# or
curl -1sLf 'https://dl.cloudsmith.io/public/shopware/shopware-cli/setup.deb.sh' | sudo bash
sudo apt install shopware-cli                       # Debian/Ubuntu
```

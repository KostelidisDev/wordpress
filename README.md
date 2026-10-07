# WordPress

Docker Compose stack for one WordPress site (Apache + PHP) backed by MariaDB, served through an existing Traefik reverse proxy.

## What's included

- **wordpress**: official `wordpress:*-apache` image. WordPress core lives in a volume and updates itself from wp-admin; the image tag only controls PHP and Apache.
- **db**: MariaDB on an internal network that has no outside access.
- Database passwords are passed as Docker secrets, not as plain environment variables.
- Apache sees the real visitor IP (`mod_remoteip`), the worker count is capped to fit the memory limit, and server tokens are hidden.
- Traefik labels add routing, an IP allowlist and security headers (HSTS, nosniff, referrer policy, frame options).
- Logs rotate, and each container has CPU, memory and PID limits.

## Requirements

- Docker Engine with Compose v2.23 or later (for inline `configs.content` and `secrets.environment`).
- A running Traefik instance with:
  - a `websecure` entrypoint whose default TLS certificate covers `DOMAIN`
  - a `strict-path` middleware defined via the Docker provider (`strict-path@docker`)
  - access to this site's network (see [Traefik networking](#traefik-networking))

## Setup

```sh
cp .env.example .env
chmod 600 .env
```

Edit `.env` and set at least:

| Variable           | Description                                                        |
| ------------------ | ------------------------------------------------------------------ |
| `SITE_NAME`        | Unique name; used for container, network, volume and router names  |
| `DOMAIN`           | Public hostname, e.g. `blog.example.com`                           |
| `DB_PASSWORD`      | WordPress database user password                                   |
| `DB_ROOT_PASSWORD` | MariaDB root password; only used when the database is first created |
| `WORDPRESS_VERSION`| Pin an exact tag from Docker Hub                                   |
| `MARIADB_VERSION`  | Pin a series such as `11.8`, never `lts` (see below)               |
| `WORDPRESS_NETWORK_SUBNET_V4` | IPv4 subnet of the frontend network, e.g. `10.100.0.0/24` |
| `WORDPRESS_NETWORK_SUBNET_V6` | IPv6 subnet of the frontend network, e.g. `fd00:100::/64` |
| `WORDPRESS_DB_NETWORK_SUBNET_V4` | IPv4 subnet of the database network, e.g. `10.101.0.0/24` |
| `WORDPRESS_DB_NETWORK_SUBNET_V6` | IPv6 subnet of the database network, e.g. `fd00:101::/64` |

Then start the stack:

```sh
docker compose up -d
docker compose ps   # both services should become healthy
```

Open `https://$DOMAIN` to finish the WordPress install.

## Configuration

All settings live in `.env`. See `.env.example` for the full list with comments. The main groups are:

- **Access control**: `IPV4_ALLOWLIST` and `IPV6_ALLOWLIST` (comma-separated CIDRs). The defaults allow everyone.
- **PHP / Apache**: `PHP_UPLOAD_MAX_FILESIZE`, `WORDPRESS_PHP_MEMORY_LIMIT`, `WORDPRESS_MAX_WORKERS`.
- **Networking**: `WORDPRESS_NETWORK_SUBNET_V4`/`_V6` (frontend) and `WORDPRESS_DB_NETWORK_SUBNET_V4`/`_V6` (database). Apache trusts forwarded client IPs only from the frontend subnets.
- **Resources**: CPU, memory and PID limits for each service, plus `DB_INNODB_BUFFER_POOL_SIZE`.

### Memory sizing

Peak WordPress memory is roughly `WORDPRESS_MAX_WORKERS × per-request memory`. A request usually takes 40–80 MB, and can go up to `WORDPRESS_PHP_MEMORY_LIMIT`. Keep that product within `WORDPRESS_MEMORY_LIMIT`, or the container will be OOM-killed under load.

### MariaDB upgrades

`MARIADB_AUTO_UPGRADE` is on, so changing `MARIADB_VERSION` upgrades the data files on the next start. That upgrade can't be undone. Pin a series and back up before you change it.

### Hardening defaults

- `DISALLOW_FILE_EDIT` turns off the theme and plugin file editor in wp-admin.
- `WP_HOME` and `WP_SITEURL` are fixed to `https://$DOMAIN`.
- Both containers run with `no-new-privileges`.

## Traefik networking

The stack creates two networks named after `SITE_NAME`:

- `$SITE_NAME`: the frontend network, dual-stack (IPv4 + IPv6) on `WORDPRESS_NETWORK_SUBNET_V4` and `WORDPRESS_NETWORK_SUBNET_V6`. Traefik must be attached to it. Each site needs its own non-overlapping subnets.
- `$SITE_NAME-db`: internal, shared only by WordPress and MariaDB; dual-stack on `WORDPRESS_DB_NETWORK_SUBNET_V4` and `WORDPRESS_DB_NETWORK_SUBNET_V6`.

If Traefik isn't already on the frontend network, connect it once the stack has started:

```sh
docker network connect "$SITE_NAME" traefik
```

Docker can't change the subnets of an existing network. After changing them, recreate the networks with `docker compose down && docker compose up -d` (volumes are kept), then reconnect Traefik.

## Running multiple sites

Copy this folder and give the copy a different `SITE_NAME` and `DOMAIN`. Container, network, volume and router names all derive from `SITE_NAME`, so sites don't collide.

## Data and backups

Two named volumes hold all state:

- `$SITE_NAME-wordpress`: WordPress core, themes, plugins and uploads
- `$SITE_NAME-db`: the MariaDB data directory

Example backup:

```sh
# Database dump
docker compose exec db sh -c \
  'mariadb-dump -u"$MARIADB_USER" -p"$(cat /run/secrets/db-password)" --single-transaction "$MARIADB_DATABASE"' \
  > backup.sql

# Files
docker run --rm -v "${SITE_NAME}-wordpress":/data -v "$PWD":/backup alpine \
  tar czf /backup/wordpress-files.tgz -C /data .
```

## Common commands

```sh
docker compose logs -f wordpress   # follow logs
docker compose pull && docker compose up -d   # apply new image tags
docker compose down                # stop (volumes are kept)
```

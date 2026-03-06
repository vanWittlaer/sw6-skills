---
name: ddev
description: >
  DDEV local development for Shopware 6 projects. Trigger when the user
  mentions DDEV, ddev exec, local development setup, watchers, or workers.
version: "2.0.0"
sw_versions: "6.6.x, 6.7.x"
references:
  - url: https://notebook.vanwittlaer.de/ddev-for-shopware
    topic: Primary DDEV + Shopware reference (install, watchers, workers, performance, RabbitMQ, media proxy)
  - url: https://notebook.vanwittlaer.de/ddev-for-shopware/using-shopware-cli-with-ddev
    topic: shopware-cli installation in DDEV
requires: shopware6/SKILL.md
---

# DDEV — Shopware 6 Local Development Skill

The primary reference for DDEV + Shopware is
https://notebook.vanwittlaer.de/ddev-for-shopware — consult it for detailed
setup guides. This skill covers conventions, decisions, and gotchas.

---

## 1. Core Rule

**All Shopware commands run via `ddev exec`.**

```bash
ddev exec bin/console cache:clear       # correct
ddev exec composer require foo/bar      # correct
php bin/console cache:clear             # WRONG — outside container
```

---

## 2. Project Setup

Reference: https://notebook.vanwittlaer.de/ddev-for-shopware/less-than-5-minutes-install-with-ddev-and-symfony-flex

### Minimal config.yaml

```yaml
name: myshop
type: shopware6
php_version: "8.3"
docroot: shopware/public
composer_root: shopware
working_dir:
  web: /var/www/html/shopware
database:
  type: mysql
  version: "8.4"
nodejs_version: "22"
webimage_extra_packages:
  - php${DDEV_PHP_VERSION}-amqp
web_environment:
  - APP_ENV=dev
```

Key choices:
- `docroot` and `composer_root` point into `shopware/` subdirectory (production template layout)
- Node 22 for SW 6.7; use Node 18 for SW 6.5.x
- Add `php-amqp` if using RabbitMQ

### Post-import-db hook

After importing a production DB dump, fix domains and rebuild:

```yaml
hooks:
  post-import-db:
    - exec: |
        mysql -uroot -proot -e "
          UPDATE sales_channel_domain
          SET url = REPLACE(url, 'https://www.myshop.de', 'https://myshop.ddev.site');
        "
        bin/console database:migrate --all
        bin/console theme:compile
        bin/console cache:clear
```

---

## 3. shopware-cli

Reference: https://notebook.vanwittlaer.de/ddev-for-shopware/using-shopware-cli-with-ddev

Install by creating `.ddev/web-build/Dockerfile.shopware-cli`:
```dockerfile
COPY --from=shopware/shopware-cli:bin /shopware-cli /usr/local/bin/shopware-cli
```

Then `ddev restart`.

Key commands:
```bash
ddev exec shopware-cli project storefront-build
ddev exec shopware-cli project admin-build
ddev exec shopware-cli extension zip MyPlugin    # package for store upload
```

**Never run `shopware-cli project ci` locally** — it removes source files as
part of its optimization step. It is meant for Docker builds and CI pipelines
only, where the build context is disposable.

---

## 4. Watchers (Hot-Reload)

Reference: https://notebook.vanwittlaer.de/ddev-for-shopware/storefront-and-admin-watchers-with-ddev

Configuration differs by Shopware version. Create `.ddev/config.watcher.yaml`:

### SW 6.7.4.2+

```yaml
web_environment:
  - HOST=0.0.0.0
  - PROXY_URL=${DDEV_PRIMARY_URL}:9998
  - STOREFRONT_SKIP_SSL_CERT=true
web_extra_exposed_ports:
  - name: vite-admin
    container_port: 5173
    http_port: 5172
    https_port: 5173
  - name: storefront-proxy
    container_port: 9998
    http_port: 8888
    https_port: 9998
  - name: storefront-assets
    container_port: 9999
    http_port: 8889
    https_port: 9999
```

Admin watcher (uses shopware-cli):
```bash
ddev exec shopware-cli extension admin-watch custom/static-plugins/MyPlugin/ \
    https://myshop.ddev.site \
    --listen :5173 \
    --external-url https://myshop.ddev.site:5173
```

Storefront watcher:
```bash
ddev exec shopware-cli project storefront-watch
```

### SW 6.5 / 6.6

```yaml
web_environment:
  - HOST=0.0.0.0
  - PORT=9997
  - DISABLE_ADMIN_COMPILATION_TYPECHECK=1
  - PROXY_URL=${DDEV_PRIMARY_URL}:9998
  - STOREFRONT_SKIP_SSL_CERT=true
web_extra_exposed_ports:
  - name: admin-proxy
    container_port: 9997
    http_port: 8887
    https_port: 9997
  - name: storefront-proxy
    container_port: 9998
    http_port: 8888
    https_port: 9998
  - name: storefront-assets
    container_port: 9999
    http_port: 8889
    https_port: 9999
```

```bash
ddev exec bin/watch-administration.sh    # or: composer run watch:admin
ddev exec bin/watch-storefront.sh        # or: composer run watch:storefront
```

Always `ddev restart` after changing watcher config.

---

## 5. Message Queue Workers

Reference: https://notebook.vanwittlaer.de/ddev-for-shopware/message-queue-setup-with-ddev

For local dev that mirrors production queue behavior:

1. Create `shopware/bin/run-worker.sh`:
```bash
#!/usr/bin/env bash
while :; do
  /usr/bin/php /var/www/html/shopware/bin/console $1 -n
done
```

2. Create `.ddev/web-build/worker.conf` (supervisor config) for `scheduled-task:run`,
   `messenger:consume async`, and `messenger:consume failed`.

3. Create `.ddev/web-build/Dockerfile.worker`:
```dockerfile
ADD worker.conf /etc/supervisor/conf.d
```

4. Disable the admin worker:
```yaml
# shopware/config/packages/z-shopware.yaml
shopware:
  admin_worker:
    enable_admin_worker: false
```

See the notebook reference for the full supervisor config.

---

## 6. Media Proxy (Fetch from Production)

Reference: https://notebook.vanwittlaer.de/ddev-for-shopware/fetch-media-from-production-or-staging-server

Avoids downloading gigabytes of media locally. Create `.ddev/nginx/media-redirect.conf`:

```nginx
   set $media_proxy_url "https://staging.example.com";

   location @mediaserver {
        resolver 1.1.1.1;
        proxy_pass $media_proxy_url$request_uri;
        proxy_set_header Authorization "Basic your-basic-auth-secret";
    }

    location ^~ /media/ {
        access_log off;
        expires max;
        try_files $uri $uri/ @mediaserver;
        break;
    }

    location ^~ /thumbnail/ {
        access_log off;
        expires max;
        try_files $uri $uri/ @mediaserver;
        break;
    }
```

Serves local files when available, transparently proxies to production for
the rest. Commit this to the repo — once set up, it just works.

---

## 7. RabbitMQ

Reference: https://notebook.vanwittlaer.de/ddev-for-shopware/rabbitmq-with-ddev

```bash
ddev get b13/ddev-rabbitmq && ddev restart
ddev exec composer require symfony/amqp-messenger
```

Environment variables (`.env.dev`):
```dotenv
MESSENGER_TRANSPORT_DSN=amqp://rabbitmq:rabbitmq@rabbitmq:5672/%2f/async
MESSENGER_TRANSPORT_LOW_PRIORITY_DSN=amqp://rabbitmq:rabbitmq@rabbitmq:5672/%2f/low_priority
MESSENGER_TRANSPORT_FAILURE_DSN=amqp://rabbitmq:rabbitmq@rabbitmq:5672/%2f/failed
```

Management UI: `https://myshop.ddev.site:15673` (rabbitmq/rabbitmq)

Requires `php-amqp` extension — see §2 `webimage_extra_packages`.

---

## 8. Performance Tweaks

Reference: https://notebook.vanwittlaer.de/ddev-for-shopware/performance-tweaks

### PHP config (`.ddev/php/shopware.ini`)

```ini
assert.active = 0
opcache.interned_strings_buffer = 20
opcache.validate_timestamps = 1         # keep 1 in dev to detect changes
zend.detect_unicode = 0
realpath_cache_ttl = 3600
```

### MySQL config (`.ddev/mysql/my.cnf`)

Remove `ONLY_FULL_GROUP_BY` from SQL mode, increase `group_concat_max_len`.

### Environment shortcuts (`.env.local`)

For faster local testing with production-like performance:
```dotenv
SQL_SET_DEFAULT_SESSION_VARIABLES=0
APP_URL_CHECK_DISABLED=1
SHOPWARE_CACHE_ID=myshop-local
```

### General tips
- **macOS**: Use OrbStack instead of Docker Desktop for faster file sync
- **Xdebug**: Keep off by default — `ddev xdebug on` only when needed
- **shopware-cli**: Faster builds than `bin/build-*.sh` scripts

---

## 9. What to Commit

| File | Commit? |
|------|---------|
| `.ddev/config.yaml` | Yes |
| `.ddev/config.watcher.yaml` | Yes |
| `.ddev/docker-compose.*.yaml` | Yes |
| `.ddev/web-build/*` | Yes |
| `.ddev/nginx/media-redirect.conf` | Yes |
| `.ddev/php/shopware.ini` | Yes |
| `.ddev/mysql/my.cnf` | Yes |
| `.ddev/config.local.yaml` | **No** — personal overrides |

---

## 10. Quick Reference

```bash
ddev start / stop / restart / poweroff
ddev describe                              # show URLs and ports
ddev ssh                                   # shell into web container
ddev mysql                                 # interactive MySQL
ddev import-db --file=dump.sql.gz          # import DB (triggers hooks)
ddev export-db --gzip --file=dump.sql.gz
ddev xdebug on / off / status
ddev exec bin/console <command>
ddev exec composer <command>
ddev logs -f                               # follow web logs
ddev logs -s db                            # database logs
```

---
name: shopware6
description: >
  Shopware 6 development conventions and patterns for client projects.
  Trigger whenever the user works on Shopware 6 code, plugins, or configuration.
version: "3.0.0"
sw_versions: "6.6.x, 6.7.x"
references:
  - url: https://developer.shopware.com/docs/guides/plugins/plugins/
    topic: Official plugin development docs
  - url: https://symfony.com/doc/7.4/service_container.html
    topic: Symfony 7.4 service container, autowiring, attributes
related_skills:
  - ddev/SKILL.md
---

# Shopware 6 — Development Skill

This skill provides conventions, architectural decisions, and gotchas for
Shopware 6 client projects. For standard "how to" questions, refer to the
official docs above — they are comprehensive. This file focuses on what the
docs don't tell you.

---

## 1. Code Conventions

Apply these to all plugin PHP code:

- **No `else` / `elseif`** — guard clauses and early returns only
- **`instanceof`** checks over loose truthy checks
- **Typed class constants**: `public const string NAME = 'value';` (PHP 8.3+)
- **Constructor promotion**: `private readonly FooService $foo`
- **No inline `style=`** in Twig — use Bootstrap utility classes (`d-none`, `d-flex`, etc.) in storefront, Shopware grid system in admin
- **No `!important`** in CSS/SCSS — use more specific selectors instead

---

## 2. Plugin Structure

### Location and installation

All plugins live in `custom/static-plugins/` and **must be Composer-installed**.

```json
"repositories": [
    { "type": "path", "url": "custom/static-plugins/*", "options": { "symlink": true } }
]
```

Then: `composer require myvendor/my-plugin:*`

**Do not use `custom/plugins/`.** Plugins there require a database to resolve,
which breaks `shopware-cli project ci` and any build pipeline without DB access.

### Composer version constraint

Target the minor versions you support:
```json
"require": {
    "shopware/core": "~6.6.0 || ~6.7.0"
}
```

### Theme plugins

Implement `ThemeInterface` on the base class. Configuration goes in
`Resources/theme.json`, not PHP. Refer to:
https://developer.shopware.com/docs/guides/plugins/themes/

---

## 3. Dependency Injection

Reference: https://symfony.com/doc/7.4/service_container/autowiring.html

**Use autowiring for everything.** Symfony 7.4 eliminates the need for explicit
service XML in virtually all cases.

### Standard services.xml (the only one you need)

```xml
<?xml version="1.0" ?>
<container xmlns="http://symfony.com/schema/dic/services"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://symfony.com/schema/dic/services
           http://symfony.com/schema/dic/services/services-1.0.xsd">
  <services>
    <defaults autowire="true" autoconfigure="true" public="false"/>
    <prototype namespace="MyVendor\MyPlugin\" resource="../../*">
      <exclude>../../Resources</exclude>
      <exclude>../../MyPlugin.php</exclude>
    </prototype>
  </services>
</container>
```

### Entity repositories autowire by name

Shopware autowires repositories by parameter naming convention —
`$<camelCaseEntityName>Repository`:

```php
public function __construct(
    private readonly EntityRepository $productRepository,        // product.repository
    private readonly EntityRepository $orderLineItemRepository,  // order_line_item.repository
    private readonly EntityRepository $myCustomEntityRepository, // my_custom_entity.repository
) {}
```

This works for all entity repositories including custom ones. No `#[Autowire]`
attribute or XML needed.

### Decorating services

Use PHP attributes, not XML:

```php
use Symfony\Component\DependencyInjection\Attribute\AsDecorator;
use Symfony\Component\DependencyInjection\Attribute\AutowireDecorated;

#[AsDecorator(decorates: ProductDetailRoute::class)]
class MyProductDetailRoute extends AbstractProductDetailRoute
{
    public function __construct(
        #[AutowireDecorated]
        private readonly AbstractProductDetailRoute $inner,
    ) {}
}
```

### Event listeners via attribute

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener(event: ProductPageLoadedEvent::class)]
class MyListener
{
    public function __invoke(ProductPageLoadedEvent $event): void { /* ... */ }
}
```

Or implement `EventSubscriberInterface` — `autoconfigure` handles the tag.

---

## 4. Key Architectural Decisions

### Entity extension vs custom fields

Reference: https://developer.shopware.com/docs/guides/plugins/plugins/framework/data-handling/add-complex-data-to-existing-entities.html

**Prefer entity extensions.** Custom fields are translatable by default, which
adds overhead (extra joins on `_translation` tables) and causes unwanted effects
for attributes that don't need translation (flags, external IDs, numeric values).

Use custom fields only when:
- The data must be merchant-editable via admin UI (custom field sets)
- The data genuinely needs per-language values

Use entity extensions for everything else:
- Relational data (associations to other entities)
- Data that needs Criteria filtering or sorting
- Non-translatable attributes (flags, external IDs, technical config)
- Performance-sensitive fields queried frequently

### Store API route vs storefront controller

| Situation | Use |
|-----------|-----|
| Headless / PWA consumers | Store API route (`_routeScope: store-api`) |
| Server-rendered Twig pages | Storefront controller (`_routeScope: storefront`) |
| Admin panel integrations | Admin API route (`_routeScope: api`) |
| AJAX from storefront JS | Store API route (preferred) or storefront with `XmlHttpRequest: true` |

### Message queue vs synchronous

Dispatch to the message queue for anything that:
- Takes > 1 second (API calls, bulk writes, email sending)
- Can be retried independently
- Doesn't need to block the user response

---

## 5. Twig Template Overrides

### Block tracking convention

When overriding a Shopware Twig block, add a comment with the original block's
content hash and the Shopware version. This makes it easy to detect when an
upstream change breaks your override:

```twig
{# shopware-block: a1b2c3d4e5f6@v6.7.5 #}
{% sw_extends '@Storefront/storefront/page/product-detail/index.html.twig' %}

{% block page_product_detail_buy %}
    {{ parent() }}
    <div class="my-addition">...</div>
{% endblock %}
```

Generate the hash: `sha256sum` of the original block content, first 12 chars.

### CSRF tokens

Shopware 6.7+ removed CSRF tokens entirely. Do not add them to forms.

### Product data in templates

When working with product data via Store API includes or in Twig, always include
both `name` and `translated` fields. The `translated` object contains the
resolved translation; the root `name` may be the parent product's value
(variant inheritance).

---

## 6. Migrations

Reference: https://developer.shopware.com/docs/guides/plugins/plugins/plugin-fundamentals/database-migrations.html

Key conventions beyond the docs:

- IDs: `BINARY(16)` — Shopware UUIDs
- Timestamps: `DATETIME(3)` — millisecond precision
- Always include `created_at DATETIME(3) NOT NULL` and `updated_at DATETIME(3) NULL`
- Use `CREATE TABLE IF NOT EXISTS` / `ADD COLUMN IF NOT EXISTS` for idempotent migrations
- Foreign key naming: `fk.<table>.<column>`
- Never put business logic in migrations — only schema changes and essential data seeding

---

## 7. Search & Indexing Gotchas

- Default tokeniser splits on **whitespace only** — hyphens and special chars are NOT split. Partial product number matching (e.g. `AB-123` → `AB`) requires decorating `ProductSearchBuilderInterface`
- `product_search_keyword` is NOT updated on incremental writes — a full reindex is needed
- Never run `es:index` during business hours on catalogs > 50k products
- `throw_exception: false` in OpenSearch config is recommended for production — it falls back to DAL on ES failure
- Always set `Criteria::setLimit()` — unbounded queries on large catalogs will time out

---

## 8. Configuration Patterns

### Plugin settings via config.xml

Reference: https://developer.shopware.com/docs/guides/plugins/plugins/plugin-fundamentals/add-plugin-configuration.html

Read in PHP:
```php
// Key format: PluginTechnicalName.config.fieldName
$value = $this->systemConfigService->get('MyPlugin.config.apiKey');
$value = $this->systemConfigService->get('MyPlugin.config.apiKey', $salesChannelId);
```

### Production shopware.yaml essentials

```yaml
shopware:
  deployment:
    runtime_extension_management: false  # prevent plugin installs from admin
```

### Redis cache + sessions (framework.yaml)

```yaml
framework:
  cache:
    app: cache.adapter.redis
    default_redis_provider: '%env(REDIS_URL)%'
  session:
    handler_id: '%env(REDIS_URL)%'
```

### S3 filesystem

For S3/object storage configuration, refer to:
https://developer.shopware.com/docs/guides/hosting/infrastructure/filesystem.html

---

## 9. Common Gotchas

| Trap | What happens | Fix |
|------|-------------|-----|
| `Context::createDefaultContext()` in storefront | Bypasses sales channel rules, prices, permissions | Use injected `SalesChannelContext` |
| `BLUE_GREEN_DEPLOYMENT=1` without two DB users | Silent failures on deploy | Set to `0` or configure two DB users |
| Editing compiled CSS directly | Overwritten on next `theme:compile` | Edit SCSS source files |
| No `--time-limit` on messenger workers | Memory leaks, eventual OOM kill | Always set `--time-limit=300` + supervisor restart |
| Running `es:index` during peak traffic | Locks tables, degrades performance | Schedule during off-hours |
| `customFields` for relational data | No joins, no Criteria filtering, poor performance | Use entity extension |
| Assuming `product.name` is translated | Returns parent value on variants | Use `product.translated.name` |

---

## 10. CLI Quick Reference

```bash
# Build (CI/CD only — never run locally, it removes source files)
# shopware-cli project ci                        # use in Dockerfile / pipeline only

# Plugin lifecycle
bin/console plugin:refresh
bin/console plugin:install --activate MyPlugin

# Cache & assets
bin/console cache:clear
bin/console theme:compile
bin/console assets:install

# Database
bin/console database:migrate --all
bin/console database:migrate --all MyPlugin

# Queue
bin/console messenger:consume async low_priority --time-limit=300 --memory-limit=512M
bin/console scheduled-task:run

# Search
bin/console es:index
bin/console dal:refresh:index

# Debug
bin/console debug:container | grep product
bin/console debug:event-dispatcher
bin/console debug:router | grep my-plugin
```

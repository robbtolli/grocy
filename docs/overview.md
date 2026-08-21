# Grocy Research Report

## 1. What is Grocy?

**Grocy** is a web-based, self-hosted groceries and household management system — marketed as **"ERP beyond your fridge."** It is an open-source hobby project by [Bernd Bestel](https://berrnd.de), first created in 2017, licensed under **MIT**.

- **Website:** https://grocy.info
- **Source:** https://github.com/grocy/grocy
- **Current version (this repo):** **4.6.0** (released 2026-03-06)
- **Demos:**
  - Stable: https://demo.grocy.info
  - Pre-release: https://demo-prerelease.grocy.info

**Purpose:** Track pantry/stock, reduce food waste, automate shopping lists, plan meals/recipes, and manage chores, tasks, equipment, and batteries — all under your own control (no SaaS dependency).

**Stack (high level):** PHP 8.5 + SQLite 3.40+, Slim 4 web framework, Blade templates, Bootstrap/jQuery frontend. Single-app SQLite design keeps deployment simple.

---

## 2. Core Features

| Area | Capabilities |
|------|----------------|
| **Stock / inventory** | Purchase, consume, transfer, inventory counts; locations; min stock levels; expiration/due tracking; price tracking (optional) |
| **Shopping lists** | Auto-suggest from low stock; group by product groups/assortments for store layout |
| **Recipes** | Ingredient availability vs stock; auto-add missing items to shopping list; "Due Score" to prefer recipes using soon-to-expire items |
| **Meal plan** | Plan meals from recipes; one-click shopping list from plan |
| **Chores** | Recurring household chores with history/assignment |
| **Tasks** | Simple to-do list |
| **Batteries** | Track charge/replacement cycles |
| **Equipment** | Device inventory + manuals/notes |
| **Userfields / user entities** | Custom fields on entities; fully custom objects/lists |
| **Barcode** | USB scanners, camera scanning (ZXing, HTTPS required), Open Food Facts lookup plugin (extensible) |
| **API** | Full REST API; Swagger UI at `/api`; UI uses the same API |
| **PWA** | Installable web app (no offline mode) |
| **i18n** | Many locales via Transifex; EN + DE maintained; RTL not supported |
| **UX** | Date input shorthands, keyboard button shortcuts, night mode, feature flags to hide modules |
| **Auth options** | Built-in users, reverse-proxy auth, LDAP |
| **Printing** | Label printer webhooks; thermal/receipt support in stack |

**Ecosystem (community):** Grocy Desktop (Windows), LinuxServer Docker image, Home Assistant add-on, Grocy Android, Grocy Mobile (iOS), Barcode Buddy, and various API/MCP integrations.

**Default login:** `admin` / `admin` — change immediately.

---

## 3. Self-Hosting with Docker

Official recommendation points at the **LinuxServer.io** image:
https://hub.docker.com/r/linuxserver/grocy · docs: https://docs.linuxserver.io/images/docker-grocy/

### Recommended `docker-compose.yml`

```yaml
services:
  grocy:
    image: lscr.io/linuxserver/grocy:latest
    container_name: grocy
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /path/to/grocy/config:/config
    ports:
      - 9283:80
    restart: unless-stopped
```

### Equivalent `docker run`

```bash
docker run -d \
  --name=grocy \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Etc/UTC \
  -p 9283:80 \
  -v /path/to/grocy/config:/config \
  --restart unless-stopped \
  lscr.io/linuxserver/grocy:latest
```

### Setup notes

1. Set `PUID`/`PGID` to your host user (`id your_user`) so `/config` permissions work.
2. Open `http://host:9283` → login `admin`/`admin` → change password.
3. Persist **`/config`** (SQLite DB, config, uploads).
4. Architectures: **amd64** and **arm64**.
5. After image upgrades, visit **`/`** (logo) so DB migrations run.
6. Production: put behind a reverse proxy (Caddy/Traefik/nginx/SWAG) for HTTPS; don't expose bare HTTP publicly.
7. Optional env overrides use `GROCY_*` (see `config-dist.php`).
8. Backup the config volume regularly (SQLite + user data).

### Non-Docker install (from README)

- Unpack release → copy `config-dist.php` → `data/config.php`
- Web root = `public/`
- PHP 8.5 + extensions: `fileinfo`, `pdo_sqlite`, `gd`, `ctype`, `intl`, `zlib`, `mbstring`
- Or clone `release` branch and install Composer + Yarn deps

---

## 4. This Repository — How the Code Is Organized

This checkout is the **upstream Grocy application source** (not a Docker-only wrapper). Version file: `version.json` → 4.6.0.

### Top-level layout

| Path | Role |
|------|------|
| `app.php` | App bootstrap: Composer autoload, config, Slim app, DI container |
| `routes.php` | All HTTP + API route definitions |
| `public/` | Web root (`index.php`, static CSS/JS/img, `viewjs/`) |
| `controllers/` | Request handlers (UI + `Api/`) |
| `services/` | Business logic / domain services |
| `middleware/` | Auth, CORS, JSON, locale |
| `helpers/` | Config validation, Blade view, barcodes, webhooks, etc. |
| `views/` | Blade templates (`layout/`, `components/`, `errors/`) |
| `migrations/` | Numbered SQL/PHP schema migrations (~150+) |
| `localization/` | Per-locale translation catalogs |
| `plugins/` | Built-in plugins (e.g. Open Food Facts barcode lookup) |
| `data/` | Runtime data dir (config, DB, plugins, backups) — must be writable |
| `config-dist.php` | Default settings template |
| `grocy.openapi.json` | OpenAPI spec for the REST API |
| `composer.json` / `package.json` | PHP and frontend dependencies |
| `update.sh` | Linux update helper (backup + overwrite, keep `data/`) |
| `changelog/` | Release notes |
| `docs/` | Extra docs (e.g. grocycode, label printing) |
| `.devtools/` | Dev/data-generation tooling |

### Request flow

```
Browser/API → public/index.php → app.php (Slim 4 + DI)
  → middleware (auth, locale, JSON/CORS)
  → routes.php
  → controllers/*  →  services/*  →  SQLite (via DatabaseService / lessql)
  → views/* (Blade) or JSON API response
```

### Controllers (feature map)

UI controllers mirror product areas:

- `StockController`, `StockReportsController` — inventory
- `RecipesController` — recipes & meal plan
- `ChoresController`, `TasksController`, `BatteriesController`, `EquipmentController`, `CalendarController`
- `UsersController`, `LoginController`, `SystemController`
- `GenericEntityController` — userfields/userentities
- `controllers/Api/*` — REST counterparts (`StockApiController`, `RecipesApiController`, …, `OpenApiController`)

### Services (domain layer)

Examples: `StockService`, `RecipesService`, `ChoresService`, `TasksService`, `BatteriesService`, `UsersService`, `UserfieldsService`, `DatabaseService`, `DatabaseMigrationService`, `SessionService`, `ApiKeyService`, `LocalizationService`, `FilesService`, `PrintService`, `DemoDataGeneratorService`.

### PHP stack (`composer.json`)

- **Slim 4** + PSR-7/HTTP
- **php-di** dependency injection
- **webman/blade** templates
- **lessql** (fork) for SQLite ORM-ish access
- gettext, Guzzle, barcode/image/print libs (escpos, iCal, HTMLPurifier, etc.)
- Autoload PSR-4: `Grocy\Services|Controllers|Middleware|Helpers`
- Vendor dir name: **`packages/`** (not `vendor/`)

### Frontend (`package.json`)

Bootstrap 4, jQuery, DataTables, FullCalendar, Chart.js, ZXing (camera barcodes), Swagger UI, Summernote, Font Awesome, etc. Page logic lives largely under `public/viewjs/`.

### Config model (`config-dist.php`)

Priority order:

1. `data/settingoverrides/<SETTING>.txt`
2. Environment `GROCY_<SETTING>`
3. Values in `data/config.php` / defaults in `config-dist.php`

Notable settings: `MODE`, `DEFAULT_LOCALE`, `CURRENCY`, `BASE_URL`/`BASE_PATH`, feature flags, auth class (default / reverse proxy / LDAP), barcode plugin, label printer webhook, entry page.

### Data & migrations

- SQLite DB and uploads live under **`data/`** (or container `/config`).
- Migrations in `migrations/` run when hitting **`/`** after version change.
- Migrations are release-to-release safe; running bleeding-edge `master` may need manual care.

### Auth middleware

Under `middleware/Auth/`: default local users, reverse-proxy header/env, LDAP.

---

## 5. Operational Takeaways

| Topic | Guidance |
|-------|----------|
| **Simplest deploy** | LinuxServer Docker image + persistent `/config` volume |
| **Security** | Change default admin password; HTTPS via reverse proxy; optional LDAP/SSO |
| **Updates (Docker)** | `docker compose pull && up -d`, then open `/` for migrations |
| **Updates (bare metal)** | Overwrite app files, keep `data/`; or use `update.sh` |
| **Customization** | Feature flags; `data/custom_css.html` / `custom_js.html`; plugins in `data/plugins` |
| **Integrations** | REST API + API keys; HA, mobile apps, Barcode Buddy, MCP servers, etc. |
| **Support** | Reddit r/grocy; GitHub issues (no private support from author) |

---

## 6. Summary

Grocy is a mature, single-binary-style **home ERP** focused on stock, waste reduction, shopping, recipes/meal plans, and light household ops. This repo is the full PHP application: Slim controllers + services, Blade UI, SQLite migrations, and a complete OpenAPI surface. For self-hosting, the practical path is **`lscr.io/linuxserver/grocy`** on port **9283** with a mounted config volume, reverse proxy TLS, and routine backups of that volume.

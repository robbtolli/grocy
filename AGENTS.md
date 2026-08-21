# AGENTS.md

Upstream Grocy (PHP household ERP). No automated test/lint suite. Upstream does **not** accept code PRs (see `.github/CONTRIBUTING.md`).

## Stack & layout

- **PHP 8.5** + SQLite 3.40+; Slim 4; Blade views; Bootstrap 4 / jQuery frontend
- Web root: `public/` → `public/index.php` → `app.php` → `routes.php`
- Domain logic: `services/`; HTTP: `controllers/` (UI) and `controllers/Api/`; templates: `views/*.blade.php`; page JS: `public/viewjs/<same-name>.js`
- Namespaces (PSR-4): `Grocy\{Services,Controllers,Middleware,Helpers}`
- OpenAPI source of truth: `grocy.openapi.json` (Swagger UI at `/api`)
- Version: `version.json`

## Dependencies (easy to get wrong)

```bash
composer install   # PHP deps → packages/  (NOT vendor/)
yarn install       # frontend → public/packages/  (see .yarnrc)
```

Both `packages/` and `public/packages/` are gitignored. Dev helper scripts live under `.devtools/` (Windows `.bat`).

Required PHP extensions: `fileinfo`, `pdo_sqlite`, `gd`, `ctype`, `intl`, `zlib`, `mbstring` (+ core: `filter`, `iconv`, `tokenizer`, `json`).

## Config & data

- Runtime data dir: `data/` (or `GROCY_DATAPATH` env/server var). Must be writable.
- First run: copy `config-dist.php` → `data/config.php` (do not delete `config-dist.php`).
- DB file: `data/grocy.db`; uploads: `data/storage/`; Blade/route cache: `data/viewcache/`.
- Settings become constants `GROCY_<NAME>`. Resolution order in `Setting()`:
  1. `data/settingoverrides/<NAME>.txt`
  2. env `GROCY_<NAME>`
  3. value from `data/config.php` / default in `config-dist.php`
- Feature toggles: `FEATURE_FLAG_*` in `config-dist.php`.
- Optional hooks: `data/custom_css.html`, `data/custom_js.html`.
- Barcode plugins: built-in `plugins/`; user plugins `data/plugins/`. Class name = filename without `.php`; select via `STOCK_BARCODE_LOOKUP_PLUGIN`.
- Auth classes: `middleware/Auth/` — Default, Session, ApiKey, ReverseProxy, LDAP (`AUTH_CLASS`).
- API auth header/query: `GROCY-API-KEY`.

## Migrations

- Files in `migrations/` (`NNNN.sql` or `NNNN.php`), applied by `DatabaseMigrationService`.
- Triggered on visit to `/` after version/`BASE_URL`/`BASE_PATH` change (see `app.php` viewcache hash).
- **Release-to-release only** — not safe between arbitrary `master` commits.
- `8888.php` always runs; `9999` is local emergency (never shipped).

## Conventions when editing

- Pair UI changes: `routes.php` + controller + `views/foo.blade.php` + `public/viewjs/foo.js` as needed.
- Prefer putting business rules in `services/*Service.php`, not controllers.
- Keep API and UI behavior aligned — frontend uses the same REST API.
- Do not commit secrets, `data/config.php`, or DB files.
- No project formatter/linter/test commands to run; verify by reading related routes/services and (if running) hitting the UI/API.

## Local reference

- Product/architecture notes: `docs/overview.md`
- Docker deploy is external: `lscr.io/linuxserver/grocy` (not defined in this repo)

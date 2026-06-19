# AGENTS.md

## What this is

Laravel 12 proxy management app (V2Board fork). PHP 8.2+, Swoole Octane, Redis, MySQL. Manages VPN/proxy subscriptions, users, and server nodes.

## Quick commands

```bash
composer install && php artisan xboard:install   # first-time setup
php artisan xboard:update                         # after pulling changes

php artisan octane:start --host=0.0.0.0 --port=7001 --workers=4 --task-workers=1
php artisan horizon                               # queue workers
php artisan ws-server start --host=0.0.0.0 --port=8076  # node WebSocket

# verification
./vendor/bin/phpstan analyse                      # static analysis (level 5, paths: app/)
./vendor/bin/phpunit tests/Unit/Services/Auth/LoginServiceTest.php  # single test (see caveat)
```

- `vendor/` is not committed — nothing runs (including `artisan`) until `composer install`.
- Only CI workflow is `.github/workflows/docker-publish.yml`: builds a multi-arch image to ghcr.io on push to `master` / `new-dev`. No test/lint CI, pre-commit hooks, or linter/formatter configs exist. Verification is manual.

## Testing — broken out of the box

> [!WARNING]
> Both `phpunit.xml` and `tests/TestCase.php` are missing in this fork. Tests import `Tests\TestCase`, so PHPUnit fatal-errors until you create both (TestCase = Laravel base + `RefreshDatabase`; SQLite in-memory is the intended setup).

Existing tests: `tests/Feature/Server/ServerHandshakeTest.php` and `tests/Unit/Services/Auth/{LoginServiceTest,RegisterServiceTest}.php`.

```bash
./vendor/bin/phpunit                              # after creating phpunit.xml + tests/TestCase.php
./vendor/bin/phpunit tests/Unit/Services/Auth/   # directory
./vendor/bin/phpunit --filter test_method_name   # by method
```

## Architecture

### Entry points

| Channel | Address | Notes |
|---|---|---|
| HTTP | Octane/Swoole :7001 (or :7002 behind Caddy) | Primary API |
| Queue | Horizon | `config/horizon.php` |
| WebSocket | Workerman :8076 | `app/Console/Commands/NodeWebSocketServer.php` |
| Scheduler | Laravel Kernel | `app/Console/Kernel.php` |

### API routes

Routes are NOT in `routes/*.php` — they are classes auto-loaded via glob from `app/Http/Routes/V1/*.php` (prefix `/api/v1`, legacy) and `app/Http/Routes/V2/*.php` (prefix `/api/v2`, current) in `app/Providers/RouteServiceProvider.php`. Dropping a new route class file in those dirs is enough; V2 has `AdminRoute`, `ClientRoute`, `PassportRoute`, `ServerRoute`, `UserRoute`.

### Layout notes (non-obvious)

- All Eloquent models use the `v2_` table prefix (`app/Models/`).
- Global helpers (`admin_setting()`, `admin_settings_batch()`, `subscribe_template()`) live in `app/Helpers/Functions.php`, autoloaded via composer `files`.
- Subscription format renderers are classes in `app/Protocols/` (Clash, SingBox, Surge, Shadowrocket, ...), dispatched by `app/Support/ProtocolManager.php`. Clients can override templates stored in `v2_subscribe_templates`; retrieve via `subscribe_template(string $name)`.
- Node WebSocket handling: `app/WebSocket/NodeWorker.php` + `NodeEventHandlers.php`.
- `public/assets/admin` is a git submodule (xboard-admin-dist) — empty until `git submodule update --init`.
- `plugins-core/` holds bundled payment/notification plugins; `plugins/` is user-installed and gitignored; `theme/` holds theme source templates.

### Middleware

Aliases registered in `app/Http/Kernel.php`: `user`, `admin`, `staff`, `client`, `log`, `server` (V1 node auth), `server.v2` (V2 node auth). Global: `InitializePlugins` (plugin boot on every request), plus `ApplyRuntimeSettings`, `ForceJson`, `Language` on the api/web groups.

### Queue jobs (`app/Jobs/`)

Jobs run on named Horizon queues: `traffic_fetch`, `stat`, `user_alive_sync`, `default`, `order_handle`, `send_email`, `send_telegram`, `send_email_mass`, `node_sync` (see `config/horizon.php`).

### Scheduled tasks (`app/Console/Kernel.php`)

| Command | Frequency |
|---|---|
| `xboard:statistics` | Daily at 00:10 |
| `check:order`, `check:commission`, `check:ticket` | Every minute |
| `check:traffic-exceeded` | Every minute (background) |
| `reset:traffic` | Every minute |
| `reset:log` | Daily |
| `send:remindMail --force` | Daily at 11:30 |
| `horizon:snapshot`, `cleanup:online-status` | Every 5 minutes |
| Plugin schedules | Via `PluginManager::registerPluginSchedules()` |

Artisan commands are auto-loaded from `app/Console/Commands/` (`php artisan list` shows them). Plugin boot also happens on Artisan boot, so plugin hooks fire in CLI too.

### Settings system

- Stored in `v2_settings` table, Redis-cached under key `admin_settings`; keys are lowercased.
- Read via `admin_setting('key', 'default')`: DB/Redis cache first, then falls back to `config('v2board.key')`, then to `$default` (config fallback wins over the passed default).
- Batch read: `admin_settings_batch(['key1', 'key2'])`. Write: `admin_setting(['key' => 'value'])`.
- Implemented in `app/Support/Setting.php` (singleton).

### Cache keys (`app/Utils/CacheKey.php`)

All Redis access goes through `CacheKey::get(string $key, mixed $uniqueValue = null)`. Core keys: `EMAIL_VERIFY_CODE`, `TEMP_TOKEN`, `SCHEDULE_LAST_CHECK_AT`, `REGISTER_IP_RATE_LIMIT`, `PASSWORD_ERROR_LIMIT`, `USER_SESSIONS`, etc. Dynamic patterns like `SERVER_*_ONLINE_USER`, `USER_ONLINE_CONN_*_*`. Unknown keys don't throw — they only log a warning in local/dev.

### Plugin system

- Base class: `app/Services/Plugin/AbstractPlugin.php`; managers under `app/Services/Plugin/`.
- Loaded by `PluginManager` on every request (via `InitializePlugins` middleware) and on Artisan boot.
- Plugins register hooks via `HookManager`, inject cron schedules, and can expose payment/notification drivers.
- Config persisted via `PluginConfigService`; records in `v2_plugins` table.

## Environment variables (`.env`)

Key variables beyond standard Laravel defaults:

| Variable | Default | Notes |
|---|---|---|
| `INSTALLED` | `false` | Set to `true` after `xboard:install` |
| `ENABLE_AUTO_BACKUP_AND_UPDATE` | `false` | Enables scheduled GCS backup |
| `GOOGLE_CLOUD_KEY_FILE` | `config/googleCloudStorageKey.json` | GCS credentials |
| `GOOGLE_CLOUD_STORAGE_BUCKET` | _(empty)_ | GCS bucket name |
| `ENABLE_CADDY` | _(not set)_ | Docker only: Caddy owns :7001, Octane moves to :7002 localhost |
| `RESOURCE_PROFILE` | `auto` | Docker worker tuning: `minimal`, `balanced`, `performance`, `auto` |

## Docker

Supervisor-managed single container. Optional Caddy reverse proxy (`ENABLE_CADDY=true`). Port mapping when Caddy enabled: public :7001 → Caddy → Octane :7002 (localhost-only inside container).

```bash
cp compose.sample.yaml compose.yaml               # bridge network
cp compose.host.sample.yaml compose.yaml          # host network (aaPanel)
docker compose run -it --rm xboard php artisan xboard:install
docker compose up -d
```

Compose variants: `compose.sample.yaml` (bridge), `compose.host.sample.yaml` (aaPanel), `compose.1panel.sample.yaml` (1Panel), `compose.split.sample.yaml` (K8s/split).

`RESOURCE_PROFILE` auto-tunes worker counts from cgroup CPU/memory limits (see `.docker/entrypoint.sh`).

## Known dead references

- `library/` is listed in `composer.json` PSR-4 autoload (`Library\\`) but the directory does not exist. No runtime error, but PHPStan warns if anything references it.

## Fork maintenance

Fork of [cedar2025/Xboard](https://github.com/cedar2025/Xboard) (`origin`), synced from `upstream`. Always rebase, never merge. Fork changes stay as exactly 3 categorized commits on top of `upstream/master`:

| Position | Category | Prefix | Example paths |
|---|---|---|---|
| `HEAD` | Code | `fix:` / `feat:` | `app/**`, `routes/**`, `database/**` |
| `HEAD~1` | Build | `build:` | `Dockerfile`, `compose.*.yaml`, `.github/**`, `update.sh`, `init.sh` |
| `HEAD~2` | Docs | `docs:` | `README.md`, `AGENTS.md`, `docs/**`, `*.md`, `.gitignore` |

Fold new fork changes into the matching category commit with `--fixup`, then sync with `--autosquash`:

```bash
git add <files>
git commit --fixup=HEAD      # code change
git commit --fixup=HEAD~1    # build change
git commit --fixup=HEAD~2    # docs change

git fetch upstream
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash upstream/master
git push --force-with-lease origin master
```

## Style

- 4-space indent, UTF-8, LF (`.editorconfig`)
- No inline comments unless explicitly requested
- YAML: 2-space indent

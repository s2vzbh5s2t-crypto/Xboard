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
./vendor/bin/phpunit                              # tests (SQLite in-memory, RefreshDatabase)
./vendor/bin/phpunit tests/Unit/Services/Auth/LoginServiceTest.php  # single test
```

No CI workflows, pre-commit hooks, or linter/formatter configs exist. Verification is manual.

## Architecture

### Entry points

| Channel | Address | Notes |
|---|---|---|
| HTTP | Octane/Swoole :7001 (or :7002 behind Caddy) | Primary API |
| Queue | Horizon | `config/horizon.php` |
| WebSocket | Workerman :8076 | `app/Console/Commands/NodeWebSocketServer.php` |
| Scheduler | Laravel Kernel | `app/Console/Kernel.php` |

### API routes

Routes are loaded via glob in `app/Providers/RouteServiceProvider.php`.

| Prefix | Route files | Status |
|---|---|---|
| `/api/v1/*` | `app/Http/Routes/V1/*.php` | Legacy |
| `/api/v2/*` | `app/Http/Routes/V2/*.php` | Current |

**V1 route files:** `ClientRoute`, `GuestRoute`, `PassportRoute`, `ServerRoute`, `UserRoute`

**V2 route files:** `AdminRoute`, `ClientRoute`, `PassportRoute`, `ServerRoute`, `UserRoute`

### Directory map

```
app/
├── Console/
│   ├── Kernel.php                  # cron schedule definitions
│   └── Commands/                   # artisan commands (see below)
├── Contracts/
│   └── PaymentInterface.php        # payment gateway contract
├── Helpers/
│   └── Functions.php               # global helpers: admin_setting(), admin_settings_batch(), subscribe_template()
├── Http/
│   ├── Kernel.php                  # middleware stack
│   ├── Controllers/
│   │   ├── V1/                     # legacy controllers (Client, Guest, Passport, Server, User)
│   │   └── V2/
│   │       ├── Admin/              # admin panel controllers
│   │       │   ├── Server/         # GroupController, MachineController, ManageController, RouteController
│   │       │   ├── ConfigController, CouponController, GiftCardController, KnowledgeController
│   │       │   ├── MailTemplateController, NoticeController, OrderController, PaymentController
│   │       │   ├── PlanController, PluginController, StatController, SystemController
│   │       │   ├── ThemeController, TicketController, TrafficResetController, UpdateController
│   │       │   └── UserController
│   │       ├── Client/
│   │       │   └── AppController   # subscription delivery
│   │       └── Server/
│   │           ├── MachineController
│   │           └── ServerController
│   ├── Middleware/                  # see Middleware section below
│   ├── Requests/
│   └── Routes/
├── Jobs/                           # queue jobs (see Queue Jobs section)
├── Models/                         # Eloquent models (v2_ table prefix)
├── Observers/                      # PlanObserver, ServerObserver, ServerRouteObserver, UserObserver
├── Protocols/                      # subscription format renderers (see Protocols section)
├── Providers/                      # service providers
├── Scope/                          # Eloquent query scopes
├── Services/                       # business logic (see Services section)
├── Support/
│   ├── AbstractProtocol.php        # base protocol class
│   ├── ProtocolManager.php         # protocol dispatch
│   └── Setting.php                 # settings wrapper
├── Traits/
├── Utils/
│   ├── CacheKey.php                # type-safe cache key builder
│   ├── Dict.php
│   └── Helper.php
└── WebSocket/
    ├── NodeEventHandlers.php       # handles node push events
    └── NodeWorker.php              # Workerman worker for node connections
```

```
plugins-core/                       # bundled payment / notification plugins
├── AlipayF2f/
├── Btcpay/
├── CoinPayments/
├── Coinbase/
├── Epay/
├── Mgate/
└── Telegram/

plugins/                            # user-installed plugins (gitignored)
theme/                              # theme source templates
public/
├── assets/admin/                   # git submodule: xboard-admin-dist
└── theme/                          # published theme assets
```

### Services (`app/Services/`)

| Service | Responsibility |
|---|---|
| `Auth/LoginService` | Login logic, token handling |
| `Auth/MailLinkService` | Magic-link email login |
| `Auth/RegisterService` | Registration, invite codes |
| `AuthService` | Shared auth utilities |
| `CaptchaService` | reCAPTCHA / turnstile verification |
| `CouponService` | Coupon validation and application |
| `DeviceStateService` | Per-user device-limit enforcement |
| `GiftCardService` | Gift card generation, redemption |
| `MailService` | Email dispatch (SMTP / Mailgun) |
| `NodeRegistry` | Node capability registry |
| `NodeSyncService` | Pushes config to nodes |
| `OrderService` | Order lifecycle, payment hooks |
| `PaymentService` | Payment gateway dispatch |
| `PlanService` | Subscription plan management |
| `Plugin/PluginManager` | Plugin loading, hook registration, schedule injection |
| `Plugin/HookManager` | Hook event fire/listen |
| `Plugin/AbstractPlugin` | Base class for plugins |
| `Plugin/PluginConfigService` | Plugin configuration persistence |
| `ServerService` | Node CRUD and config generation |
| `SettingService` | Thin wrapper around `admin_setting()` |
| `StatisticalService` | Traffic and revenue stats aggregation |
| `TelegramService` | Telegram bot notification dispatch |
| `ThemeService` | Theme install/publish |
| `TicketService` | Support ticket management |
| `TrafficResetService` | Periodic traffic counter resets |
| `UpdateService` | Application self-update via GitHub |
| `UserService` | User CRUD, subscription renewal, commission |

### Models (`app/Models/` — all use `v2_` table prefix)

`AdminAuditLog`, `CommissionLog`, `Coupon`, `GiftCardCode`, `GiftCardTemplate`, `GiftCardUsage`, `InviteCode`, `Knowledge`, `MailLog`, `MailTemplate`, `Notice`, `Order`, `Payment`, `Plan`, `Plugin`, `Server`, `ServerGroup`, `ServerLog`, `ServerMachine`, `ServerMachineLoadHistory`, `ServerRoute`, `ServerStat`, `Setting`, `Stat`, `StatServer`, `StatUser`, `SubscribeTemplate`, `Ticket`, `TicketMessage`, `TrafficResetLog`, `User`

### Middleware

| Alias | Class | Used on |
|---|---|---|
| `user` | `User.php` | User API routes |
| `admin` | `Admin.php` | Admin routes |
| `staff` | `Staff.php` | Staff-only routes |
| `client` | `Client.php` | Client/subscription routes |
| `server` | `Server.php` | V1 node auth |
| `server.v2` | `ServerV2.php` | V2 node auth |
| `log` | `RequestLog.php` | Request audit logging |
| _(global)_ | `InitializePlugins.php` | Plugin boot on every request |
| _(global)_ | `ApplyRuntimeSettings.php` | Apply DB-backed runtime settings |
| _(global)_ | `ForceJson.php` | Force `Accept: application/json` on API group |
| _(global)_ | `Language.php` | Locale detection |

### Protocols (`app/Protocols/`)

Each class renders a subscription response for a specific client format:

`Clash`, `ClashMeta`, `General`, `Loon`, `QuantumultX`, `Shadowrocket`, `Shadowsocks`, `SingBox`, `Stash`, `Surfboard`, `Surge`

Dispatch is handled by `app/Support/ProtocolManager.php`. Override templates are stored in `v2_subscribe_templates` (model: `SubscribeTemplate`); the global helper `subscribe_template(string $name)` retrieves them.

### Queue jobs (`app/Jobs/`)

| Job | Default queue |
|---|---|
| `NodeUserSyncJob` | `user_alive_sync` |
| `OrderHandleJob` | `order_handle` |
| `SendEmailJob` | `send_email` |
| `SendTelegramJob` | `send_telegram` |
| `StatServerJob` | `stat` |
| `StatUserJob` | `stat` |
| `TrafficFetchJob` | `traffic_fetch` |

Horizon queue configuration: `config/horizon.php`. Named queues: `traffic_fetch`, `stat`, `user_alive_sync`, `default`, `order_handle`, `send_email`, `send_telegram`, `send_email_mass`, `node_sync`.

### Scheduled tasks (`app/Console/Kernel.php`)

| Command | Frequency |
|---|---|
| `xboard:statistics` | Daily at 00:10 |
| `check:order` | Every minute |
| `check:commission` | Every minute |
| `check:ticket` | Every minute |
| `check:traffic-exceeded` | Every minute (background) |
| `reset:traffic` | Every minute |
| `reset:log` | Daily |
| `send:remindMail --force` | Daily at 11:30 |
| `horizon:snapshot` | Every 5 minutes |
| `cleanup:online-status` | Every 5 minutes |
| Plugin-registered schedules | Via `PluginManager::registerPluginSchedules()` |

### Artisan commands (`app/Console/Commands/`)

`BackupDatabase`, `CheckCommission`, `CheckOrder`, `CheckServer`, `CheckTicket`, `CheckTrafficExceeded`, `CleanupOnlineStatus`, `ClearUser`, `HookList`, `MigrateFromV2b`, `NodeWebSocketServer`, `ResetLog`, `ResetPassword`, `ResetTraffic`, `ResetUser`, `SendRemindMail`, `Test`, `XboardInstall`, `XboardRollback`, `XboardStatistics`, `XboardUpdate`

### Settings system

- Stored in `v2_settings` table.
- Redis-cached under the `admin_settings` key.
- Read via `admin_setting('key', 'default')` — checks Redis cache first, falls back to `config('v2board.key')`, then to `$default`.
- Batch read: `admin_settings_batch(['key1', 'key2'])`.
- Write: `admin_setting(['key' => 'value'])`.
- Implemented in `app/Support/Setting.php`; the `Setting` instance is a singleton.

### Cache keys (`app/Utils/CacheKey.php`)

All Redis access goes through `CacheKey::get(string $key, mixed $uniqueValue = null)`. Core named keys include `EMAIL_VERIFY_CODE`, `TEMP_TOKEN`, `SCHEDULE_LAST_CHECK_AT`, `REGISTER_IP_RATE_LIMIT`, `PASSWORD_ERROR_LIMIT`, `USER_SESSIONS`, etc. Dynamic patterns follow the form `SERVER_*_ONLINE_USER`, `USER_ONLINE_CONN_*_*`, etc.

### Plugin system

- Plugins live in `plugins/` (user-installed, gitignored) and `plugins-core/` (bundled).
- Base class: `app/Services/Plugin/AbstractPlugin.php`.
- Loaded and initialized by `PluginManager` on every request (via `InitializePlugins` middleware) and on Artisan boot.
- Plugins can register hooks via `HookManager`, inject cron schedules, and expose their own payment or notification drivers.
- Plugin configuration persisted via `PluginConfigService`; plugin records stored in `v2_plugins` table (`Plugin` model).

## Key dependencies

| Package | Purpose |
|---|---|
| `laravel/octane` 2.11.* | Swoole-backed HTTP server |
| `laravel/horizon` ^5.30 | Redis queue dashboard and workers |
| `workerman/workerman` ^5.1 | WebSocket server for node comms |
| `workerman/redis` ^2.0 | Async Redis in Workerman context |
| `zoujingli/ip2region` ^2.0 | Offline IP geolocation |
| `spatie/db-dumper` ^3.4 | Database backup |
| `stripe/stripe-php` ^7.36 | Stripe payment |
| `symfony/yaml` | YAML serialisation for proxy configs |
| `bacon/bacon-qr-code` ^2.0 | QR code generation |
| `google/cloud-storage` ^1.35 | GCS backup target |
| `larastan/larastan` ^3.0 (dev) | PHPStan Laravel extensions |

## Environment variables (`.env`)

Key variables beyond standard Laravel defaults:

| Variable | Default | Notes |
|---|---|---|
| `INSTALLED` | `false` | Set to `true` after `xboard:install` |
| `ENABLE_AUTO_BACKUP_AND_UPDATE` | `false` | Enables scheduled GCS backup |
| `GOOGLE_CLOUD_KEY_FILE` | `config/googleCloudStorageKey.json` | GCS credentials |
| `GOOGLE_CLOUD_STORAGE_BUCKET` | _(empty)_ | GCS bucket name |
| `ENABLE_CADDY` | _(not set)_ | Set `true` in Docker to enable Caddy reverse proxy |
| `RESOURCE_PROFILE` | `auto` | Docker worker tuning: `minimal`, `balanced`, `performance`, `auto` |

## Testing

Tests use `Tests\TestCase` (Laravel base), `RefreshDatabase` trait, SQLite in-memory. Test bootstrap sets `APP_KEY` and seeds `admin_settings` cache directly.

> [!WARNING]
> `tests/TestCase.php` does not exist in this fork. Tests importing `Tests\TestCase` may need it created.

```bash
./vendor/bin/phpunit                              # all
./vendor/bin/phpunit tests/Unit/Services/Auth/   # directory
./vendor/bin/phpunit --filter test_method_name   # by method
```

Test layout:

```
tests/
├── Feature/
│   └── Server/
└── Unit/
    └── Services/
```

## Docker

Supervisor-managed single container. Optional Caddy reverse proxy (`ENABLE_CADDY=true`).

```bash
cp compose.sample.yaml compose.yaml               # bridge network
cp compose.host.sample.yaml compose.yaml          # host network (aaPanel)
docker compose run -it --rm xboard php artisan xboard:install
docker compose up -d
```

Compose variants:

| File | Use case |
|---|---|
| `compose.sample.yaml` | Standard bridge network |
| `compose.host.sample.yaml` | aaPanel (host network) |
| `compose.1panel.sample.yaml` | 1Panel |
| `compose.split.sample.yaml` | K8s / split-service |

`RESOURCE_PROFILE` env controls worker counts auto-tuned from cgroup CPU/memory limits (see `.docker/entrypoint.sh`).

## Known dead references

- `library/` is listed in `composer.json` PSR-4 autoload (`Library\\`) but the directory does not exist. This causes no runtime error but will produce PHPStan warnings if anything references it.

## Fork maintenance

Fork of [cedar2025/Xboard](https://github.com/cedar2025/Xboard). Always rebase, never merge. Fork changes stay as exactly 3 categorized commits on top of `upstream/master`:

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

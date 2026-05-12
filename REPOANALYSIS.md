# rails-client-logger — RepoDocs
_Generated on 2026-05-11_

## Summary

### Overview
`rails-client-logger` is a Ruby on Rails engine (packaged as a gem) that exposes a server endpoint and a small CoffeeScript/jQuery client so browser-side JavaScript can push log messages (debug/info/warn/error/fatal) into the standard Rails server log. The repository is an internal PowerReviews fork of the upstream OSS gem (`girishso/rails-client-logger`), used as a shared utility library by Rails applications in the org that want to capture client-side JS errors server-side; if `exception_notification` is loaded in the host app, `error`/`fatal` messages are also routed through `ExceptionNotifier`.

### Tech Stack
| Category | Technology | Version |
|----------|-----------|---------|
| Language | Ruby | _Not specified in repo (no `.ruby-version` / no `required_ruby_version`)_ |
| Language | CoffeeScript (client asset) | n/a |
| Framework | Rails (engine host) | `>= 4.0` (gemspec); Gemfile.lock pins 4.0.0 |
| Framework | jQuery (via `jquery-rails`) | 2.2.0 (dev dummy app) |
| Database | SQLite3 (dummy test app only) | 1.3.7 |
| Build Tool | Bundler / RubyGems | Bundled With 1.12.5 |
| Build Tool | Rake | 10.1.0 |
| CI/CD | GitHub Actions (secrets scan only) | n/a |
| Cloud/Infra | None (library gem — no deployment artifacts) | n/a |

### Consumers
| Consumer | Type | How They Use It |
|----------|------|----------------|
| Host Rails applications in the org | Internal (gem consumer) | `gem 'rails-client-logger'` + `mount RailsClientLogger::Engine, :at => "logger"`; the engine's `RailsClientLoggersController#log` action accepts client JS log posts |
| Browser-side JavaScript (in host apps) | Client (asset) | Loads `rails_client_logger.js.coffee` via Sprockets, then calls `jsLogger.debug/info/warn/error/fatal(message)` which `POST`s to the mounted engine URL |
| GitHub Actions (trufflehog secrets scan) | CI/CD | Cron-scheduled secrets scan workflow under `.github/workflows/secrets-scan.yml` |
| Slack (`github-token-scan` channel) | External notification | Posted via `rtCamp/action-slack-notify` if secrets scan fails |

Note: no other repo in this org has been positively identified as a consumer from code in this repo — it is a generic Rails engine and consumers cannot be enumerated from inside it.

### Dependencies on Org Repos
_None._ The gem depends only on Rails (`>= 4.0`) and optionally cooperates with `exception_notification` (external OSS gem). No PowerReviews-internal repos are referenced.

### External Integrations
| Service | Purpose | Integration Type |
|---------|---------|-----------------|
| ExceptionNotifier (`exception_notification` gem) | Forward `error` / `fatal` client-side log entries as exception notifications, if the gem is loaded in the host app | SDK (optional, soft `defined?` check) |
| Slack (via `rtCamp/action-slack-notify`) | Notify `github-token-scan` channel on secrets-scan failure | Webhook (outbound) |
| TruffleHog (`edplato/trufflehog-actions-scan`) | Repository secrets scanning | SDK (GitHub Action) |

### Async & Scheduled Work
| Channel / Job | Type | Direction | Purpose |
|--------------|------|-----------|---------|
| `secret-scan` GitHub Actions job (`cron: 0 14 * * 1-5`) | Scheduled (cron) | N/A (for jobs) | Weekday secrets scan of the repo via TruffleHog, with Slack notification on failure |

### Upgrade Alerts
| Dependency | Current Version | Issue | Severity |
|-----------|----------------|-------|----------|
| Rails | 4.0.0 (Gemfile.lock) / `>= 4.0` (gemspec) | Rails 4.x is end-of-life (security maintenance ended 2017); multiple known CVEs in 4.0.0 | Critical |
| jquery-rails | 2.2.0 | Ships jQuery 1.9-era client; long EOL; XSS CVEs in this jQuery line | Critical |
| sqlite3 (gem) | 1.3.7 | EOL major; incompatible with modern Ruby/SQLite; only used by the test dummy app | Severe |
| protected_attributes | 1.0.3 | EOL Rails 4-era gem (replaced by Strong Parameters) | Severe |
| `actions/checkout@master` | floating `master` ref | Uses unpinned `master` ref of a third-party Action (supply-chain risk) | Severe |
| `edplato/trufflehog-actions-scan@master` | floating `master` ref | Same — unpinned third-party Action | Severe |
| Ruby runtime | Unpinned (no `.ruby-version` / no `required_ruby_version`) | No declared Ruby; gem effectively constrained by Rails 4 to EOL Ruby 2.x | Severe |

## API Reference

### HTTP Endpoint (mounted as a Rails Engine)

Mount in host app:
```ruby
mount RailsClientLogger::Engine, :at => "logger"
```

Default route (see `config/routes.rb:1`):

| Method | Path | Controller#Action |
|--------|------|-------------------|
| `POST` | `/logger/rails_client_logger/log` | `RailsClientLogger::RailsClientLoggersController#log` |

**`POST /rails_client_logger/log`** (`app/controllers/rails_client_logger/rails_client_loggers_controller.rb:3`)

Request params:
| Param | Type | Required | Allowed values |
|-------|------|----------|----------------|
| `level` | string | yes | `debug`, `info`, `warn`, `error`, `fatal` |
| `message` | string | yes | free-form log message |

Responses:
- `200 OK` (`head :ok`) — on accepted level; writes `Rails.logger.<level>(message)`.
- `400 Bad Request` (`head :bad_request`) — when `level` is not in the allowed set.

Side effect: if `ExceptionNotifier` is defined in the host app and `level` is `:error` or `:fatal`, calls `ExceptionNotifier.notify_exception(Exception.new(params[:message]), env: request.env)`.

Strong-parameters helper (`log_params`): permits `:level, :message`. Note: defined but not invoked inside `log` (the action reads `params[:level]` / `params[:message]` directly).

### Ruby modules / classes

- `RailsClientLogger` (`lib/rails-client-logger.rb`) — top-level namespace.
- `RailsClientLogger::VERSION = "1.1.1"` (`lib/rails-client-logger/version.rb:2`).
- `RailsClientLogger::Engine < ::Rails::Engine` (`lib/rails-client-logger/engine.rb`) — isolates namespace.
- `RailsClientLogger::RailsClientLoggersController < ApplicationController` — `log`, `log_params`.
- `JsLogger::ApplicationController < ActionController::Base` (`app/controllers/js_logger/application_controller.rb`) — legacy/empty base class (likely vestigial; not referenced by the engine controller).
- `RailsClientLoggerGenerator < Rails::Generators::Base` (`lib/generators/rails_client_logger/rails_client_logger_generator.rb`) — `generate` task: inserts `mount RailsClientLogger::Engine, :at => "logger"` into host `routes.rb`, and adds `require rails_client_logger` to the host's Sprockets manifest (`application.js`, `application.js.coffee`, or `application.coffee`) immediately after the `jquery` require.

### Client-side JS API (`vendor/assets/javascripts/rails_client_logger.js.coffee`)

Global `window.jsLogger` object:

| Method | Signature | Behavior |
|--------|-----------|----------|
| `jsLogger.invoke(level, message)` | `(level: string, message: string)` | jQuery `POST` to `window.jsLoggerBasePath + window.jsLoggerUrl` with `{level, message}`; sets `X-CSRF-Token` from `meta[name="csrf-token"]` |
| `jsLogger.debug(message)` | `(message)` | `invoke('debug', message)` |
| `jsLogger.info(message)` | `(message)` | `invoke('info', message)` |
| `jsLogger.warn(message)` | `(message)` | `invoke('warn', message)` |
| `jsLogger.error(message)` | `(message)` | `invoke('error', message)` |
| `jsLogger.fatal(message)` | `(message)` | `invoke('fatal', message)` |

Configurable globals (`||=` defaults):
- `window.jsLoggerBasePath` — default `''`
- `window.jsLoggerUrl` — default `"/logger/rails_client_logger/log"`

## Architecture

### System context diagram

```
                       ┌──────────────────────────────────────────┐
                       │            Host Rails Application        │
                       │  (consumes this gem via `gem 'rails-     │
                       │   client-logger'` + Engine mount)        │
                       │                                          │
   ┌──────────┐  POST  │  ┌────────────────────────────────────┐  │
   │  Browser ├───────►│  │ /logger/rails_client_logger/log    │  │
   │ JS code  │  (CSRF)│  │  RailsClientLoggersController#log  │  │
   │ jsLogger │        │  └─────────────────┬──────────────────┘  │
   │ .info /  │        │                    │                     │
   │ .error / │        │                    ▼                     │
   │ .fatal …│         │            Rails.logger.<level>          │
   └──────────┘        │                    │                     │
        ▲              │                    ▼                     │
        │ loads via    │       (development|production|…).log     │
        │ Sprockets    │                                          │
        │              │   if defined?(ExceptionNotifier) and     │
   rails_client_       │   level in {:error,:fatal} → notify      │
   logger.js.coffee    │                    │                     │
   (this gem)          │                    ▼                     │
                       │           ExceptionNotifier (opt)        │
                       └──────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────────┐
   │ GitHub Actions (cron, this repo only): TruffleHog secrets    │
   │ scan → Slack on failure                                      │
   └──────────────────────────────────────────────────────────────┘
```

### Key components

- **`RailsClientLogger::Engine`** (`lib/rails-client-logger/engine.rb`) — isolated Rails engine. Mounted by host apps; routes drawn in `config/routes.rb` expose a single `POST :log` member on the `rails_client_logger` singular resource.
- **`RailsClientLoggersController`** (`app/controllers/rails_client_logger/rails_client_loggers_controller.rb`) — receives client log posts, whitelists levels, dispatches to `Rails.logger`, optionally fans out to `ExceptionNotifier`.
- **CoffeeScript client** (`vendor/assets/javascripts/rails_client_logger.js.coffee`) — installs `window.jsLogger`, posts via jQuery with the CSRF header.
- **Generator** (`lib/generators/rails_client_logger/rails_client_logger_generator.rb`) — `rails g rails_client_logger` patches the host's `routes.rb` and Sprockets manifest.

### Data flow

1. Browser JS calls `jsLogger.<level>(message)`.
2. `jsLogger.invoke` sends `POST {level, message}` with `X-CSRF-Token` to `window.jsLoggerBasePath + window.jsLoggerUrl` (defaults to `/logger/rails_client_logger/log`).
3. Engine route dispatches to `RailsClientLoggersController#log`.
4. Controller validates `params[:level]` ∈ {debug, info, warn, error, fatal}, calls `Rails.logger.<level>(params[:message])`.
5. If `ExceptionNotifier` is loaded and level is `error` or `fatal`, an exception notification is emitted with `Exception.new(params[:message])` and `request.env`.
6. Response: `200 OK` (or `400 Bad Request` for invalid level).

### CI/CD tooling

GitHub Actions is the only CI tool present (`.github/workflows/secrets-scan.yml`). There is **no build, test, or release workflow** in this repo. The single workflow:
- Trigger: cron `0 14 * * 1-5` (weekdays, 14:00 UTC).
- Steps: `actions/checkout@master` → `edplato/trufflehog-actions-scan@master` (regex-based scan, depth 1) → on failure, `rtCamp/action-slack-notify@v2.0.2` posts to Slack channel `github-token-scan`.

### Test architecture

A dummy Rails app lives under `test/dummy/` (Rails 4 conventions), and there are scaffolded test files:
- `test/js-logger_test.rb`
- `test/integration/navigation_test.rb`
- `test/functional/js_logger/js_loggers_controller_test.rb`
- `test/unit/helpers/js_logger/js_loggers_helper_test.rb`
- `test/test_helper.rb`

These are the default Rails engine generator templates; no CI runs them.

### Data model / database schema

Not relevant — this gem persists nothing. The only DB is the dummy app's SQLite (development scaffolding); no migrations or models ship with the gem.

### Auth & trust boundaries

- **Inbound auth**: none enforced by the gem itself — the engine controller has no `before_action :authenticate*` and no token check. CSRF is mitigated via Rails' default CSRF check (the client explicitly sends `X-CSRF-Token` from `meta[name="csrf-token"]`).
- **Authorization model**: none in the gem. The README documents a host-side recipe to opt in (CanCan: subclass `RailsClientLoggersController` and `skip_authorization_check`); enforcement is delegated entirely to the host app.
- **Routes reachable without authentication**: `POST /logger/rails_client_logger/log` is the only route and is open by default — any session with a valid CSRF token can write arbitrary strings to the host's Rails log and (if `ExceptionNotifier` is wired up) trigger `error`/`fatal` exception notifications. This is a potential log-injection / notification-spam surface that the host app is expected to gate.
- **Outbound auth**: none. `Rails.logger` and `ExceptionNotifier` are in-process calls.

### Data ownership

_Not applicable — this gem owns no datastore._ It writes to `Rails.logger` (a file handle owned by the host app). The SQLite database under `test/dummy/` is dummy-app scaffolding only.

### Deployment topology

_Deployment topology not in this repo._ This is a library/gem; it has no Dockerfile, no k8s/Terraform, no deployment scripts. Distribution is via `gem build` / `gem push` (Rakefile is the stock `bundler/gem_tasks` default).

## Repo Activity

Derived from git history; current HEAD is `28f2147`.

- **Created**: 2013-01-30 (`7b77e47` initial commit by upstream OSS authors).
- **Last meaningful change**: `d038710` (2019-02-01) — "rename folder to be consistent with rails autoloading" (renamed `js_logger` controller folder → `rails_client_logger`). The most recent commits after that are CI-only (`a93afd0`/`28f2147`, 2020-07-07/08 — add/update the secrets-scan workflow).
- **Activity level**: 0 commits in the last 90 days (HEAD is from 2020-07-08; no commits since). The repo is effectively dormant.
- **Hot spots** (top files by churn over the last 6 months): _No commits in the last 6 months — no hot spots to report._ Lifetime, the most-touched files are `README.md` (~10+ touches), `lib/rails-client-logger/version.rb` (version bumps), and `app/controllers/rails_client_logger/rails_client_loggers_controller.rb` (logic + autoload-rename).
- **Recent major changes**: _No major changes in the last 6 months._ The most recent structural change in the repo's lifetime was the 2019-02-01 autoload-driven controller folder rename; the most recent feature work was the 2014-09-05 `ExceptionNotifier` integration and namespaced-URL support.

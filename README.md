# VSCode Debug Configurations

This directory has two files that work together:

- **`environment.example`** — a shell script of `export`s for every host/port/secret the debug configs need. Fill in the blanks (devbackend host, DB/admin secrets, SMTP creds, etc.) and `source` it **before** launching VSCode, so VSCode's own environment (and therefore `${env:VAR}` substitution in `launch.json`) picks the values up:
  ```bash
  source .vscode/environment.example && code .
  ```
  If you'd rather not edit this file directly, copy it first (`cp .vscode/environment.example .vscode/environment.local.sh`) and source the copy instead. If you change a value while VSCode is already open, re-source and restart VSCode (or at least the terminal/debug session) for the change to take effect. See **Loading the env vars automatically with direnv** below for a way to skip this manual step entirely.
- **`launch.json`** — the Run & Debug configs, one per backend service plus the API gateway, Redis, and the mobile app. Each reads its config from `${env:SWAYRIDER_...}` vars rather than hardcoding values, so the same `launch.json` works whether you're pointed at your own dev-mini or a teammate's.

## Loading the env vars automatically with direnv

Manually `source`-ing the file before every `code .` gets old fast, and if you forget, `${env:VAR}` substitution in `launch.json` silently resolves to empty strings instead of erroring. [direnv](https://direnv.net/) fixes this by auto-loading/unloading a directory's env vars as you `cd` in and out; its VSCode extension makes VSCode's own environment pick them up too, so you can just open the workspace and go — no manual sourcing, no relaunching VSCode from a pre-sourced shell.

1. Install direnv and hook it into your shell (zsh):
   ```bash
   brew install direnv
   echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc && source ~/.zshrc
   ```
2. Install the [direnv VSCode extension](https://marketplace.visualstudio.com/items?itemName=mkhl.direnv) (`mkhl.direnv`). Without it, VSCode's own process (and therefore `${env:VAR}` substitution in `launch.json`) won't see vars direnv loads into your shell — the extension is what syncs them into the editor.
3. Copy `environment.example` to `.envrc` at the **workspace root** (not `.vscode/`) — direnv only looks for `.envrc` in the directory you `cd` into, or a parent of it:
   ```bash
   cp .vscode/environment.example .envrc
   direnv allow
   ```
4. Fill in `.envrc` with your real values, the same way you would `environment.example`. direnv re-checks the file on every change; it'll ask you to `direnv allow` again each time you edit it (a safety check against unreviewed changes to files that get sourced automatically).

With this set up, `cd`-ing into the repo (or just having VSCode open on it) keeps both your shell and VSCode's environment in sync with `.envrc` automatically — the manual `source ... && code .` step above becomes unnecessary.

## The three variable families

| Prefix | Points at | Used by |
|---|---|---|
| `SWAYRIDER_<SERVICE>_HOST` / `_HTTP_PORT` / `_GRPC_PORT` | The deployed dev backend, always | A service's **own** config, for the downstream services it depends on (e.g. Mail Service's `AUTHSERVICE_HOST`/`PORT`) |
| `SWAYRIDER_LOCAL_<SERVICE>_*` | `127.0.0.1`, loopback ports | A service's **own** debug config, for the ports *it itself* binds to when you run it locally |
| `SWAYRIDER_DEBUG_<SERVICE>_*` | Either the dev backend or `LOCAL_*`, whichever pair is uncommented | Only `API - Debug` — lets you decide, per service, whether the gateway should reach it on the dev backend or on your machine |

The `DEBUG_*` vars are the mechanism for mixing local and remote services (see below). Each service has two exports in `environment.example`: one pointing at the dev backend (active by default) and one commented-out pointing at `LOCAL_*`. Flip which is commented to change where `API - Debug` looks for that service, then re-source.

## Running a single service locally

Just launch that service's config, e.g. **Mail Service**. It binds to its own `LOCAL_*` ports and reaches its dependencies (authservice) on the dev backend — no other setup needed. Same pattern for Region/Router/Tiles Service and AuthService.

## Running the gateway fully against the dev backend

1. Launch **Redis - Local** (the gateway needs Redis for rate limiting/queueing, even when every microservice it proxies to is remote).
2. Launch **API - DevMini** — every downstream service is the dev backend.

## Mixed mode — gateway + some services local, the rest on the dev backend

This is the useful case when you're actively working on one or two services and want to exercise them through the real gateway. Worked example: running **AuthService** and **Mail Service** locally, everything else on the dev backend.

1. In `environment.example`, under "Dynamic configurations", find the `Auth Service` and `Mail Service` blocks. Comment out the devbackend-pointing exports and uncomment the `_LOCAL_` alternatives:
   ```bash
   # Auth Service
   #export SWAYRIDER_DEBUG_AUTHSERVICE_HOST="${SWAYRIDER_AUTHSERVICE_HOST}"
   #export SWAYRIDER_DEBUG_AUTHSERVICE_HTTP_PORT="${SWAYRIDER_AUTHSERVICE_HTTP_PORT}"
   #export SWAYRIDER_DEBUG_AUTHSERVICE_GRPC_PORT="${SWAYRIDER_AUTHSERVICE_GRPC_PORT}"
   #export SWAYRIDER_DEBUG_AUTHSERVICE_WEB_PORT="${SWAYRIDER_AUTHSERVICE_WEB_PORT}"
   export SWAYRIDER_DEBUG_AUTHSERVICE_HOST="${SWAYRIDER_LOCAL_AUTHSERVICE_HOST}"
   export SWAYRIDER_DEBUG_AUTHSERVICE_HTTP_PORT="${SWAYRIDER_LOCAL_AUTHSERVICE_HTTP_PORT}"
   export SWAYRIDER_DEBUG_AUTHSERVICE_GRPC_PORT="${SWAYRIDER_LOCAL_AUTHSERVICE_GRPC_PORT}"
   export SWAYRIDER_DEBUG_AUTHSERVICE_WEB_PORT="${SWAYRIDER_LOCAL_AUTHSERVICE_WEB_PORT}"
   ```
   Do the same for the `Mail Service` block. Leave every other service's `DEBUG_*` block untouched (still pointing at the dev backend).
2. Re-source the file: `source .vscode/environment.example` (or your local copy).
3. Launch **Redis - Local**, then **AuthService** and **Mail Service** — each starts listening on its `LOCAL_*` port.
4. Launch **API - Debug** — it reads the `DEBUG_*` vars, so it reaches AuthService and Mail Service on `127.0.0.1` at their local ports, and every other service (Region/Router/Search/Tiles) on the dev backend.
5. VSCode runs multiple debug sessions at once from a single window — start each config in turn from the Run & Debug panel; you don't need multiple VSCode windows.
6. Exercise the mixed setup through the gateway with either **Mobile - SwayriderApp (Local API)** (below) or the Bruno "public" collection against `127.0.0.1:8888` — see `testing/README.md`.

## Running everything locally

Same as mixed mode, but flip every service's `DEBUG_*` block to its `LOCAL_*` alternative, then launch every service, Redis, and **API - Debug**.

## Mobile app

Two configs:

- **Mobile - SwayriderApp** — no setup required. Talks to the hosted dev backend (`https://api.swayrider-dev.hevanto-it.com`) using the app's compiled-in defaults.
- **Mobile - SwayriderApp (Local API)** — talks to a **locally-launched** gateway at `SWAYRIDER_LOCAL_API_HOST:SWAYRIDER_LOCAL_API_HTTP_PORT` (`127.0.0.1:8888` by default). Requires **API - DevMini** or **API - Debug** to already be running.

Under the hood, `swayriderapp` has no `.env`/dotenv setup — all backend connection settings are compile-time `--dart-define` values consumed in `swayriderapp/lib/config/app_config.dart`, one `SCHEME`/`HOST`/`PORT`/`PATH_PREFIX` group each for `AUTH_API_*`, `TILES_API_*`, `SEARCH_API_*`. The "Local API" config only overrides `SCHEME`/`HOST`/`PORT` — the path-prefix defaults already match what `swayrider-api` serves them on. See `swayriderapp/DEVELOPMENT.md` ("Backend / Configuration") for the full variable list and manual `flutter run --dart-define=...` usage.

**Simulator/emulator gotcha**: `127.0.0.1` only reaches your host machine from a run target that shares its network namespace — iOS Simulator, macOS, and web all work out of the box with the config as-is. An **Android emulator** needs `10.0.2.2` instead; a **physical device** needs your machine's real LAN IP (and the phone must be able to reach it — same network, firewall permitting). Override by running manually rather than via the canned config, e.g.:
```bash
cd swayriderapp
flutter run \
  --dart-define=AUTH_API_SCHEME=http --dart-define=AUTH_API_HOST=10.0.2.2 --dart-define=AUTH_API_PORT=8888 \
  --dart-define=TILES_API_SCHEME=http --dart-define=TILES_API_HOST=10.0.2.2 --dart-define=TILES_API_PORT=8888 \
  --dart-define=SEARCH_API_SCHEME=http --dart-define=SEARCH_API_HOST=10.0.2.2 --dart-define=SEARCH_API_PORT=8888
```

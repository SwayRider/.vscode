# VSCode Debug Configurations

This directory has two files that work together:

- **`environment.example`** — a shell script of `export`s for every host/port/secret the debug configs need. Fill in the blanks (devbackend host, DB/admin secrets, SMTP creds, etc.) and `source` it **before** launching VSCode, so VSCode's own environment (and therefore `${env:VAR}` substitution in `launch.json`) picks the values up:
  ```bash
  source .vscode/environment.example && code .
  ```
  If you'd rather not edit this file directly, copy it first (`cp .vscode/environment.example .vscode/environment.local.sh`) and source the copy instead. If you change a value while VSCode is already open, re-source and restart VSCode (or at least the terminal/debug session) for the change to take effect. See **Loading the env vars automatically with direnv** below for a way to skip this manual step entirely. It also sets `SWAYRIDER_ROOT` (the workspace root) and puts `tools/` on `PATH`, so scripts like `gitstatus.py` run from anywhere once it's sourced.
- **`launch.json`** — the Run & Debug configs, one per backend service plus the API gateway, Redis, Postgres, and the mobile app. Each reads its config from `${env:SWAYRIDER_...}` vars rather than hardcoding values, so the same `launch.json` works whether you're pointed at your own dev-mini or a teammate's.

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
| `SWAYRIDER_DEBUG_<SERVICE>_*` | Either the dev backend or `LOCAL_*`, whichever pair is uncommented | `API - Debug` (lets you decide, per service, whether the gateway should reach it on the dev backend or on your machine) **and** each standalone service config's own AuthService/MailService dependency — e.g. `Mail Service`'s `AUTHSERVICE_HOST`/`PORT` and `AuthService`'s `MAILSERVICE_HOST`/`PORT` read `SWAYRIDER_DEBUG_AUTHSERVICE_*`/`SWAYRIDER_DEBUG_MAILSERVICE_*` too, so the same toggle governs both the gateway path and direct standalone launches |
| `SWAYRIDER_DEBUG_DB_*` | Either the local test Postgres or the dev backend's, whichever pair is uncommented | The `AuthService` config — its Postgres connection (host/port/credentials/db name) |
| `SWAYRIDER_DEBUG_AUTHSERVICE_WEB_HOST` | `127.0.0.1`, or an emulator host alias, whichever line is uncommented | The `AuthService` config's `REGISTRATION_URL`/`VERIFICATION_URL`/`RESET_PASSWORD_URL` — which host the invite-registration link (and unused-by-mobile fallback links) should use, so it's reachable from whichever mobile run target opens it. See "Registration/verification/reset-password links" under "Mobile app" below |

The `DEBUG_*` vars are the mechanism for mixing local and remote services (see below). Each service has two exports in `environment.example`: one pointing at the dev backend (active by default) and one commented-out pointing at `LOCAL_*`. Flip which is commented to change where `API - Debug` looks for that service, then re-source. The `SWAYRIDER_DEBUG_DB_*` block works the same way for AuthService's database — except its default is the **local** Postgres (see the AuthService note below for why). `SWAYRIDER_DEBUG_AUTHSERVICE_WEB_HOST` follows the same comment/uncomment pattern, but toggles between three *local* reachability options (loopback vs. the Android/Genymotion emulator aliases) rather than local-vs-dev-backend.

## Running a single service locally

Just launch that service's config, e.g. **Mail Service**, **Search Service**, **Region Service**, **Router Service**, or **Tiles Service**. It binds to its own `LOCAL_*` ports and reaches AuthService via `SWAYRIDER_DEBUG_AUTHSERVICE_HOST`/`_GRPC_PORT` — the dev backend by default, no other setup needed. Flip that block in `environment.example` to its `LOCAL_*` alternative and re-source if you want it to reach a locally-debugged **AuthService** instead (same toggle `API - Debug` uses). **AuthService** works the same way in the other direction: its `MAILSERVICE_HOST`/`PORT` reads `SWAYRIDER_DEBUG_MAILSERVICE_*`, so flipping *that* block lets AuthService send mail through a locally-debugged **Mail Service** instead of the dev backend's.

**AuthService** is the exception: it connects to Postgres via the `SWAYRIDER_DEBUG_DB_*` vars, which default to the **local** test Postgres (`testing/infra/postgres`) rather than the dev backend, because AuthService runs DB migrations at startup — pointing a locally-debugged instance at the shared dev database would apply migrations to it. Launch **Postgres - Local** first, then **AuthService**. To debug against the deployed dev database instead, flip the `SWAYRIDER_DEBUG_DB_*` block in `environment.example` to its commented-out devbackend alternatives and re-source — only do this if applying migrations to that database is OK. Prefer a database that starts empty every time (e.g. clearing out test users between runs) instead of reusing state across sessions? Launch **Postgres - Local (testing/infra/postgres, Transient)** instead — see `testing/README.md` ("Local Postgres") for the transient-vs-persistent tradeoff and how to reset the persistent volume.

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
3. Launch **Postgres - Local** (AuthService's database), **Redis - Local**, then **AuthService** and **Mail Service** — each starts listening on its `LOCAL_*` port.
4. Launch **API - Debug** — it reads the `DEBUG_*` vars, so it reaches AuthService and Mail Service on `127.0.0.1` at their local ports, and every other service (Region/Router/Search/Tiles) on the dev backend.
5. VSCode runs multiple debug sessions at once from a single window — start each config in turn from the Run & Debug panel; you don't need multiple VSCode windows.
6. Exercise the mixed setup through the gateway with either **Mobile - SwayriderApp (Local API)** (below) or the Bruno "public" collection against `127.0.0.1:8888` — see `testing/README.md`.

## Debugging AuthService and Mail Service directly, without the gateway

Useful when you're working on Mail Service (or AuthService) itself and don't need to exercise it through `swayrider-api` — e.g. debugging template rendering, SMTP delivery, or a specific AuthService endpoint.

1. Flip the `Auth Service` and/or `Mail Service` blocks in `environment.example` as in step 1 of Mixed mode above (whichever side(s) you want local — you don't need both; e.g. leave `Mail Service` on the dev backend and flip only `Auth Service` to debug AuthService against a local Postgres while Mail Service stays on dev-mini).
2. Re-source the file.
3. Launch **Postgres - Local** if AuthService is one of the local sides, then launch **AuthService** and/or **Mail Service** directly — each reads the *other's* `DEBUG_*` block for its dependency (`Mail Service`'s `AUTHSERVICE_HOST`/`PORT` and `AuthService`'s `MAILSERVICE_HOST`/`PORT`), so no gateway or `API - Debug` session is needed at all.

This reuses the exact same `DEBUG_AUTHSERVICE_*`/`DEBUG_MAILSERVICE_*` blocks as Mixed mode — flipping them once affects both the gateway path and these direct standalone launches together.

## Running everything locally

Same as mixed mode, but flip every service's `DEBUG_*` block to its `LOCAL_*` alternative, then launch **Postgres - Local** and **Redis - Local** (infrastructure), every service, and **API - Debug**.

## Registering the API gateway's service client (automatic)

The gateway needs a service client on authservice (`region:query routing:execute search:execute tiles:serve` scopes). In the docker stack the `swayrider-api-register` container handles this; for the debugger it's a task:

- **`Register API Service Client`** (`Terminal → Run Task…`) runs `swctl auth ensure-service-client` against the authservice that **API - Debug** reaches (`SWAYRIDER_DEBUG_AUTHSERVICE_HOST`/`_GRPC_PORT`, defaulting to the dev backend) using the admin account (`SWAYRIDER_ADMIN_EMAIL`/`_PASSWORD`), and writes `SWAYRIDER_API_CLIENT_ID`/`SWAYRIDER_API_CLIENT_SECRET` to `.local/swayrider-api.env` at the workspace root.
- **API - Debug** has it as `preLaunchTask` and loads the file via `envFile`, so launching that config registers automatically. The command is idempotent: if the credentials file already exists **and** the target authservice still has a matching service client, it skips — it doesn't just trust the local file, it verifies against the authservice it's about to register against, retrying while authservice is still booting (e.g. right after launching AuthService). If the file exists but the target authservice has no matching client (a reset/fresh database, or a different authservice than last time), it treats the file as stale and re-registers automatically — no manual cleanup needed, including after resetting the local test Postgres (`testing/infra/postgres/reset.sh` or its transient mode).
- The task can also be run manually before launching anything.

Caveats:

- If the target authservice **already has a `swayrider-api` client** — always the case for the deployed dev backend — and no local credentials file exists, the task fails with an "already exists" error and aborts the launch. That's expected: for the dev backend with *every* downstream service remote, use **API - DevMini** and set `SWAYRIDER_API_CLIENT_ID`/`SWAYRIDER_API_CLIENT_SECRET` yourself (e.g. from the dev stack's credentials volume — see the Secrets section of `environment.example`). If you want AuthService to stay on the dev backend while still mixing *other* services locally (the case Mixed mode's worked example doesn't cover, since it flips AuthService itself to local), use **API - Debug (Personal Client)** instead — see below.
- The admin account must **not have MFA enabled** — `swctl` can't complete the second factor, so registration would fail.
- `.local/` at the workspace root is outside every git repo here, so the credentials file is never committed.

### Registering a personal client (AuthService stays on the dev backend)

`API - DevMini`'s downstream hosts/ports are hardcoded to the always-remote `SWAYRIDER_*_HOST` vars, not `DEBUG_*`, so it can't route to a locally-run service the way `API - Debug` can. If you want that per-service mixing *and* AuthService has to stay on the dev backend (its shared `swayrider-api` client's secret is unrecoverable, so the plain `Register API Service Client` task will always fail there — see the caveat above), use **API - Debug (Personal Client)** instead of **API - Debug**:

1. Set `SWAYRIDER_DEV_CLIENT_NAME` in `environment.example`/`.envrc` to something unique to you, e.g. `swayrider-api-<yourname>-dev`.
2. Launch **API - Debug (Personal Client)**. Its prelaunch task, **Register API Service Client (Personal)**, registers a client under that name instead of the shared `swayrider-api` one, and writes credentials to `.local/swayrider-api-personal.env` (kept separate from `API - Debug`'s file so the two never collide). It's idempotent the same way the shared-client task is.
3. Everything else — the `DEBUG_*` per-service local/remote toggles — works exactly like **API - Debug**.

A personal client, once created, is yours: nobody else should be registering under the same name, so there's no shared-secret problem to run into again.

## Mobile app

Five configs:

- **Mobile - SwayriderApp** — no setup required. Talks to the hosted dev backend (`https://api.swayrider-dev.hevanto-it.com`) using the app's compiled-in defaults.
- **Mobile - SwayriderApp (Local API)** — talks to a **locally-launched** gateway at `SWAYRIDER_LOCAL_API_HOST:SWAYRIDER_LOCAL_API_HTTP_PORT` (`127.0.0.1:8888` by default). Requires **API - DevMini** or **API - Debug** to already be running.
- **Mobile - SwayriderApp (Local Genymotion)** / **Mobile - SwayriderApp (Local Android Emulator)** — same as "Local API", but using the emulator host alias instead of `127.0.0.1` (see the gotcha below).
- **Mobile - SwayriderApp (Android Emulator, dev-mini)** — same target as plain **Mobile - SwayriderApp** (the hosted dev backend), but works around a split-horizon DNS gotcha specific to Android emulators — see below.

(`swayriderapp/.vscode/` also has launch configs of its own — those are deprecated leftovers; use the configs here instead.)

Under the hood, `swayriderapp` has no `.env`/dotenv setup — all backend connection settings are compile-time `--dart-define` values consumed in `swayriderapp/lib/config/app_config.dart`, one `SCHEME`/`HOST`/`PORT`/`PATH_PREFIX` group each for `AUTH_API_*`, `TILES_API_*`, `SEARCH_API_*`. The "Local API" config only overrides `SCHEME`/`HOST`/`PORT` — the path-prefix defaults already match what `swayrider-api` serves them on. See `swayriderapp/DEVELOPMENT.md` ("Backend / Configuration") for the full variable list and manual `flutter run --dart-define=...` usage.

**Simulator/emulator gotcha**: `127.0.0.1` only reaches your host machine from a run target that shares its network namespace — iOS Simulator, macOS, and web all work out of the box with the config as-is. A stock **Android emulator (AVD)** needs `10.0.2.2` instead; a **Genymotion** virtual device needs `10.0.3.2` (its VirtualBox-based networking uses a different host alias than the standard AVD); a **physical device** needs your machine's real LAN IP (and the phone must be able to reach it — same network, firewall permitting). Set `SWAYRIDER_LOCAL_API_HOST` accordingly in your `.envrc`/`environment.local.sh`, or override by running manually rather than via the canned config, e.g.:
```bash
cd swayriderapp
flutter run \
  --dart-define=AUTH_API_SCHEME=http --dart-define=AUTH_API_HOST=10.0.2.2 --dart-define=AUTH_API_PORT=8888 \
  --dart-define=TILES_API_SCHEME=http --dart-define=TILES_API_HOST=10.0.2.2 --dart-define=TILES_API_PORT=8888 \
  --dart-define=SEARCH_API_SCHEME=http --dart-define=SEARCH_API_HOST=10.0.2.2 --dart-define=SEARCH_API_PORT=8888
```

### Debugging against a hosted dev backend from an Android emulator

A different gotcha from the one above, specific to hosted dev-mini domains (e.g. `api.swayrider-dev.hevanto-it.com`) rather than a locally-launched gateway: these use **split-horizon DNS** — from your LAN they resolve to a private IP that's directly reachable, but from anywhere else (a public resolver, an Android emulator's own virtual network) they may resolve to a public IP that doesn't accept connections at all. A desktop tool run on your Mac (Bruno, `dig`) gets the private answer and works; a plain **Mobile - SwayriderApp** run on an Android emulator can silently fail to reach the backend for this reason alone (the app used to misreport this as "email not verified" instead of a connection problem — now it shows a proper connection-issue screen, see `swayriderapp/lib/data/services/api/connection_exception.dart`).

**iOS Simulator needs nothing special** — it shares the Mac's real network stack and resolver directly, so it already gets the same LAN answer as the Mac. Only the Android emulator, which runs its own virtual network with its own DNS, needs a workaround.

Use **Mobile - SwayriderApp (Android Emulator, dev-mini)** instead of plain **Mobile - SwayriderApp** when running on an Android emulator against a hosted dev backend. Its `preLaunchTask`, **Boot Android Emulator with LAN DNS (dev-mini)**, boots the `SWAYRIDER_ANDROID_AVD_NAME` AVD directly with `-dns-server SWAYRIDER_LAN_DNS_SERVER` (your LAN router — same DNS server your Mac uses) so the emulator resolves the hostname to the same private IP, polls `adb ... getprop sys.boot_completed` until it's ready, then attaches. Set both vars in `environment.example`/`.envrc` for your setup; if you ever run more than one Android emulator instance concurrently, also update the `emulator-5554` device id in both `launch.json` and `tasks.json` (ids increment for later instances).

The task needs `ANDROID_SDK_ROOT` to find the AVD's system image — it falls back to `~/Library/Android/sdk` if unset, but export it yourself if your SDK lives elsewhere. If the task fails with `PANIC: Cannot find AVD system path`, that's the fix.

Alternatively, for a one-off run without touching DNS at all, bypass the hostname entirely with `--dart-define=AUTH_API_HOST=<LAN IP>` (or the WireGuard address from the `testing/bruno` dev-mini environments), or fix the dev-mini deployment's public-facing path (HAProxy/port-forward) so the public DNS answer is actually reachable.

### Registration/verification/reset-password links

These links are served by AuthService's own web pages (`/web/register`, `/web/verify-user`, `/web/reset-password`) but, matching production, are only reachable through the **gateway**'s `/web` proxy (`swayrider-api/internal/handlers/web.go`) — never AuthService's own port directly (its ports aren't meant to be reached by anything other than the gateway, per `CLAUDE.md`). So both **AuthService** and a gateway (**API - Debug**) need to be running locally for any of these links to resolve — the gateway alone isn't enough either, since it just proxies through to AuthService's `WEB_PORT`.

Signup and forgot-password in the app always send AuthService a `verificationUrl`/`resetUrl` of their own (`swayriderapp/lib/config/app_config.dart`'s `VERIFICATION_REDIRECT_URL`/`RESET_PASSWORD_REDIRECT_URL`, defaulting to the production dev backend) — independent of `AUTH_API_HOST`. So to have those emails link back to a **locally-debugged AuthService** (reached through the gateway, as above), the three "Local" Mobile configs also override these two dart-defines to `http://<host alias>:${SWAYRIDER_LOCAL_API_HTTP_PORT}/web/...`, using the same host alias and port as their `AUTH_API_HOST`/`_PORT`.

This is separate from AuthService's own `REGISTRATION_URL`/`VERIFICATION_URL`/`RESET_PASSWORD_URL` env vars (see the "Auth Service web host" block in `environment.example`, also gateway-routed the same way) — those only matter for the **invite-registration** email (there's no per-request override for that one), not for verification/reset-password, which the mobile app always overrides itself as described above.

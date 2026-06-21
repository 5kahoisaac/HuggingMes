# Changelog

## 0.3.0 - 2026-06-21

### Features

- **`ENABLE_ENV_BUILDER` flag** — ENV Builder is now opt-in (`false` by default). Set `ENABLE_ENV_BUILDER=true` to show the `/env-builder` link on the dashboard and enable the route. Users who manage secrets directly through HF Space settings have one less exposed surface.
- **`DEV_MODE`-gated terminal button** — "💻 Open Terminal →" on the dashboard is hidden when `DEV_MODE=false`, consistent with the existing JupyterLab skip logic already in `start.sh`.

### Fixes

- **Hermes v0.17 startup crash — Python permission denied** — Hermes v0.17 ships its `.py` files read-only but patches them during `workspace/startup.sh`. Added `find /opt/hermes -name "*.py" -exec chmod u+w {} +` in both the Dockerfile (build time) and `start.sh` (runtime, after HF Dataset restore) so self-patching succeeds.
- **Restore Errno 17 (File exists)** — `shutil.rmtree(ignore_errors=True)` silently failed when a restore target was a symlink-to-directory; the subsequent `copytree` raised "File exists". Fixed by checking `is_symlink()` before rmtree and using `dirs_exist_ok=True` on `copytree`.
- **Restore Errno 13 (Permission denied)** — Hermes ships some skill directories read-only, causing rmtree to fail and restore to crash. Added `_make_writable()` to recursively `chmod u+w` the target tree before removal.
- **Events feed disconnected / tool calls not appearing in chat** — Hermes v0.17 serves `/api/events` and other WebSocket endpoints from the dashboard process (port 9119), not the gateway (port 8642). The WebSocket upgrade handler now routes `/api/*`, `/assets/*`, `/dashboard-plugins/*`, and `/ds-assets/*` to `DASHBOARD_PORT`. `Origin` headers are rewritten to the internal target address so hermes's `_ws_host_origin_is_allowed()` check passes.
- **Gateway showing Offline/Unreachable while hermes is running** — `statusPayload()` ran three sequential `canConnect()` calls (600 ms timeout each); worst case 1800 ms, long enough for the gateway to time out under load and flip to Offline. Replaced with `Promise.all()` (max 600 ms total) and raised the gateway-specific timeout to 2000 ms.
- **SSE events not flushing to browser** — Upstream hop-by-hop headers (`transfer-encoding`, `connection`, `keep-alive`) forwarded from the hermes backend caused double-chunking that blocked SSE flush. These headers are now stripped; `socket.setNoDelay(true)` added for SSE streams.

### Changes

- Gateway supervision migrated to `hermes gateway run` / `hermes gateway stop` CLI commands for proper s6-overlay lifecycle management, with a graceful shutdown timeout to prevent hangs on Space restart.
- `hermes-sync.py`: renamed loop variable `stat` → `file_stat` in `metadata_marker` to stop it shadowing the `stat` module import used in `_make_writable`.
- `health-server.js`: replaced nested ternaries in `renderDashboard` (`syncTone`, `telegramTone`, `keepAliveTone`, `keepAliveDetail`) with `if/else` chains.
- `env-builder.js`: simplified `...[...new Set(...)]` to `...new Set(...)`.

## 0.2.1 - 2026-05-20

### Fixes

- **Build fails after update** — `libasound2` renamed to `libasound2t64` in Debian bookworm. Dockerfile now tries both names, falling back gracefully so builds succeed on all base image variants.
- **Unpinned jupyterlab breaks venv** — `uv pip install jupyterlab` without a version constraint could pull a release incompatible with existing Hermes venv packages. Pinned to `>=4.0,<5` range to bound resolution.
- **`uv` not in PATH during Docker build** — switched from bare `uv` to explicit `/opt/hermes/.venv/bin/uv` so the install works regardless of base image PATH configuration.
- **`visudo` not in PATH during Docker build** — switched to explicit `/usr/sbin/visudo` path.
- **Kanban patch exits with code 1** — entire kanban migration patch now wrapped in `try/except`; any unexpected error (file encoding, permission, changed upstream structure) skips silently instead of failing the Docker build.

## 0.2.0 - 2026-05-19

### Features

- **ENV Builder** — interactive UI at `/env-builder` for configuring all Space secrets. Grouped sections: Core, Backup, Telegram, Terminal, Providers, Cloudflare, Advanced. Model picker with provider/model-name presets. Import/export as `HUGGINGMES_ENV_BUNDLE` or plain `.env`.
- **JupyterLab terminal** — full shell access at `/terminal/`. On by default (`DEV_MODE=true`). Uses `GATEWAY_TOKEN` as terminal password — no separate `JUPYTER_TOKEN` needed. Dashboard button added.
- **Chromium browser tools** — installs Chromium and display/font libs so Hermes browser-use tools work out of the box.
- **Plugin persistence** — Hermes plugin directory symlinked into the persistent volume; plugins survive container restarts.
- **Secret redaction** — enabled by default in Hermes config (`security.redact_secrets: true`).
- **Cloudflare Keepalive** — Cloudflare Worker setup for automatic space keep-awake.

### Fixes

- **Space stuck at RUNNING_APP_STARTING** — root cause: `start_jupyter()` called `python3 -c "import jupyterlab"` using system Python; JupyterLab is installed in the Hermes venv. Import failed → `return 1` → `set -euo pipefail` killed `start.sh` → container crashed every boot. Fixed to use `/opt/hermes/.venv/bin/python`.
- **Terminal double password prompt** — proxy now injects `Authorization: token <JUPYTER_TOKEN>` header when forwarding requests to JupyterLab, bypassing its own login screen. One login instead of two.
- **Gemini 404 errors** — strip `google/` or `gemini/` prefix when setting Hermes model name; Hermes gemini provider expects bare model name (e.g. `gemini-2.5-flash`, not `google/gemini-2.5-flash`).
- **Config persistence** — use `setdefault` for user-configurable fields; always overwrite `model.default` and `model.provider` from env so deploy-time settings win without clobbering dashboard changes.
- **Keys disappearing after restart** — sync state to HF Dataset on natural gateway exit (in addition to periodic sync and SIGTERM path).
- **Sync timeouts** — set `HF_HUB_DOWNLOAD_TIMEOUT=300` and enable `HF_XET_HIGH_PERFORMANCE` for faster dataset transfers.
- **hermes not found in terminal** — symlink `hermes` CLI into `$HERMES_HOME/.local/bin`; add `/etc/profile.d/hermes-venv.sh` so PATH includes venv bin in all shell types.
- **Kanban migration crash** — wrap `ALTER TABLE ADD COLUMN` in try/except; idempotent on existing databases.
- **Health endpoint returning 503** — `/health` always returns HTTP 200 (gateway status in JSON body). Returning 503 when gateway was starting caused Docker HEALTHCHECK to fail indefinitely.

### Changes

- Space emoji updated to 🪽 (Hermes winged sandals) across README and dashboard.
- Login page redesigned to match HuggingClaw dark-card aesthetic.
- ENV Builder, Terminal, and Control UI all require session auth (single `GATEWAY_TOKEN` login).
- `HF_HUB_ENABLE_HF_TRANSFER` (deprecated) replaced with `HF_XET_HIGH_PERFORMANCE=1`.
- HEALTHCHECK `start-period` tuned to 60s; health endpoint always returns 200.

## 0.1.0 - 2026-05-03

- Initial HuggingMes Docker Space wrapper for Nous Research Hermes Agent.
- Added HF Space dashboard, `/health`, `/status`, `/v1/*` proxy, and Telegram webhook proxy.
- Added Cloudflare Worker setup for Telegram Bot API base URL proxying.
- Added private HF Dataset backup and restore for Hermes state.
- Added Cloudflare Keepalive Worker setup for automatic space keep-awake.

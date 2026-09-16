# Plan: Google Health (Fitbit) MCP server in Python

## Context

Develop a maintainable MCP server that lets AI read our Google Health api data that is provided by a wearable. 

The important finding from research: the **legacy Fitbit Web API is being decommissioned this month (September 2026)** and has been replaced by the **Google Health API** (`https://health.googleapis.com/v4`, launched March 2026, Google OAuth 2.0). So this server targets the Google Health API directly and never touches `api.fitbit.com`.

Decisions already made with you:
- **Runtime:** Python, managed by `uv` (it will download Python 3.12; older system Pythons may be too old for the `mcp` v2 SDK).
- **Data scope (read-only):** Activity & fitness, Health metrics, Sleep. No nutrition, profile, or write scopes.



## Prerequisites you must do yourself (I cannot enter credentials or create accounts)

1. Your Fitbit account must already be migrated to a Google Account (Google's deadline was 19 May 2026). Legacy Fitbit-only accounts cannot use the new API.
2. Install `uv`: `winget install astral-sh.uv` (or the PowerShell installer from astral.sh).
3. In Google Cloud Console: create/select a project, enable **Google Health API** (`health.googleapis.com`), configure the OAuth consent screen in **Testing** status and add your own Gmail as a test user (this avoids Google's restricted-scope verification for personal use; testing mode allows up to 100 users). Add the three read-only scopes below on the Data Access page.
4. Create an OAuth client of type **Desktop app**, download the JSON, save it outside the repo (e.g. `%APPDATA%\google-health-mcp\client_secret.json`). Fallback if Desktop type is rejected for this API: a Web client with `http://localhost:8765/` as the redirect URI (the code will support a fixed port).

Scopes requested (all `https://www.googleapis.com/auth/googlehealth.` prefixed):
`activity_and_fitness.readonly`, `health_metrics_and_measurements.readonly`, `sleep.readonly`.

## Project layout

```
pyproject.toml            # uv project; deps: mcp>=2,<3, google-auth, google-auth-oauthlib, requests
.python-version           # 3.12
.gitignore                # token.json, client_secret*.json, .venv
README.md                 # setup steps above + client config snippets
src/google_health_mcp/
  __init__.py
  __main__.py             # CLI: `auth` (run OAuth flow) | `serve` (default, stdio MCP)
  config.py               # env vars, file paths (%APPDATA%\google-health-mcp\), SCOPES
  auth.py                 # load/refresh credentials, InstalledAppFlow.run_local_server
  catalog.py              # registry of supported data types (see below)
  client.py               # HealthClient: thin REST wrapper with retry/backoff
  shaping.py              # compact the verbose API JSON for LLM consumption
  server.py               # MCPServer instance + tool definitions
tests/
  test_catalog.py, test_filters.py, test_shaping.py, test_client.py (mocked HTTP)
```

## Key modules

### `config.py`
- `GOOGLE_HEALTH_CLIENT_SECRETS` env var (path to OAuth client JSON); default `%APPDATA%\google-health-mcp\client_secret.json`.
- `GOOGLE_HEALTH_TOKEN_FILE` env var; default `%APPDATA%\google-health-mcp\token.json`.
- `BASE_URL = "https://health.googleapis.com/v4"`, `SCOPES` list.

### `auth.py`
- `load_credentials()` -> `google.oauth2.credentials.Credentials | None` from token file; refresh if expired (google-auth handles it); persist refreshed token.
- `run_auth_flow()` -> `InstalledAppFlow.from_client_secrets_file(...).run_local_server(port=0)` (fixed port if `GOOGLE_HEALTH_OAUTH_PORT` set), save token. Invoked only by the `auth` CLI command, **never** from inside the MCP server (a stdio server must not open browsers or print to stdout).
- Server tools raise a clear error telling the user to run `uv run google-health-mcp auth` when no valid token exists.

### `catalog.py`
Static table of the data types the server supports, each with: API name, category, human description, which time-filter field pattern `dataPoints.list` accepts (`interval.start_time`, `sample_time.physical_time`, `date`, or session-style), whether `dailyRollUp` / `rollUp` support it, and the max range (14 days for `heart-rate`, `total-calories`, `active-minutes`, `calories-in-heart-rate-zone`; 90 days otherwise).

Included types:
- Activity: `steps`, `distance`, `floors`, `total-calories`, `active-energy-burned`, `active-minutes`, `active-zone-minutes`, `time-in-heart-rate-zone`, `sedentary-period`, `exercise`, `daily-vo2-max`, `vo2-max`.
- Health metrics: `heart-rate`, `daily-resting-heart-rate`, `daily-heart-rate-variability`, `heart-rate-variability`, `daily-oxygen-saturation`, `oxygen-saturation`, `daily-respiratory-rate`, `daily-sleep-temperature-derivations`, `core-body-temperature`, `weight`, `body-fat`, `height`.
- Sleep: `sleep`.

### `client.py`
`HealthClient(credentials)` using `google.auth.transport.requests.AuthorizedSession` (auto token refresh). Methods:
- `list_data_points(data_type, filter, page_size, page_token)` -> `GET /users/me/dataTypes/{t}/dataPoints`
- `daily_roll_up(data_type, start_date, end_date, window_days=1)` -> `POST .../dataPoints:dailyRollUp`
- `roll_up(data_type, start_time, end_time, window_size)` -> `POST .../dataPoints:rollUp`
- Retry on 429/5xx with exponential backoff (3 attempts). Map 401 -> "re-run auth", 403 -> name the missing scope. Per-user limit is 300 req/min so no client-side throttle beyond backoff.

### `shaping.py`
The API returns verbose protobuf-JSON (int64s as strings, nested `interval` with UTC offsets). Compact it: ints as ints, `startTime/endTime` as ISO strings, drop `dataSource` unless requested, and for `sleep` and `exercise` return summaries by default with stages/splits only when `include_details=True`. Cap any list at the page size and always pass `next_page_token` through.

### `server.py` — tools

All handlers are sync (v2 runs them on a worker thread) and return dicts so the SDK emits `structured_content`.

| Tool | Purpose | API call |
|---|---|---|
| `auth_status` | Is a token present/valid, which scopes, where files live, how to fix | local only |
| `list_data_types` | The catalog above, so the model knows valid names | local only |
| `get_daily_summary(start_date, end_date, metrics=[...])` | One row per day with steps, distance, floors, calories, active zone minutes, resting HR, HRV, SpO2, respiratory rate | fan-out `dailyRollUp` per metric, merged by date |
| `get_daily_rollup(data_type, start_date, end_date, window_days=1)` | Generic per-day (or N-day) aggregate for any rollup-capable type | `dailyRollUp` |
| `get_intraday(data_type, start_time, end_time, window="5m")` | Fine-grained series (e.g. heart rate every 5 min, steps per hour) | `rollUp` with `windowSize` |
| `get_data_points(data_type, start, end, page_size, page_token, include_source=False)` | Raw records for any type; builds the AIP-160 filter from the catalog's time-field pattern | `dataPoints.list` |
| `get_sleep(start_date, end_date, include_stages=False)` | Sleep sessions with minutes asleep/awake, stage totals, efficiency | `dataPoints.list` on `sleep` (page size capped at 25 by API) |
| `get_exercises(start_date, end_date, include_details=False)` | Workouts with type, duration, distance, calories, avg HR | `dataPoints.list` on `exercise` (cap 25) |

Input validation: dates as `YYYY-MM-DD`, datetimes as RFC-3339; reject ranges over the per-type maximum with a message that states the limit. Convert validation problems to `ToolError`-style results (not `MCPError`, which in v2 becomes a JSON-RPC error and is worse for the model).

### `__main__.py`
`google-health-mcp auth` runs the flow; `google-health-mcp` (no args) starts `MCPServer("google-health").run()` over stdio. Expose as a `[project.scripts]` entry so `uv run google-health-mcp` works.

## Client configuration (goes in README)

Claude Desktop `claude_desktop_config.json`:
```json
{ "mcpServers": { "google-health": {
    "command": "uv",
    "args": ["--directory", "C:\\path\\to\\google-health-mcp", "run", "google-health-mcp"] } } }
```
Claude Code:
```
claude mcp add google-health -- uv --directory C:\path\to\google-health-mcp run google-health-mcp
```

## Verification

1. `uv sync` then `uv run pytest` — unit tests for filter construction, date validation, shaping, and the client's retry/error mapping using mocked HTTP.
2. You run `uv run google-health-mcp auth`, sign in with your Google account in the browser, confirm `token.json` is written.
3. `uv run mcp dev src/google_health_mcp/server.py` opens the MCP Inspector; call `auth_status`, then `get_daily_summary` for the last 7 days and `get_sleep` for last night, and check values against the Fitbit app.
4. Register in Claude Code with the command above and ask "how did I sleep last night?" end to end.
5. Confirm `git status` never shows `token.json` or the client secret.

## Assumptions / open risks

- Exact `dataPoints.list` filter field names differ per data type (interval vs sample vs date vs session). The catalog encodes what the docs state; step 3 above will surface any that are wrong and they are one-line fixes.
- Google may reject a Desktop-type OAuth client for this API (its setup guide only shows a Web client). The fixed-port fallback covers that.
- The `mcp` v2 Python SDK pins `httpx2`; using `requests` for the Google calls avoids any dependency conflict.

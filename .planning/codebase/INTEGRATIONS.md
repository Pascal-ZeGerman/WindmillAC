# External Integrations

**Analysis Date:** 2026-04-22

## APIs & External Services

**Windmill AC Cloud (Blynk platform):**
- Service: Windmill Air dashboard — hosted Blynk IoT backend
  - Base URL: `https://dashboard.windmillair.com` (defined as `BASE_URL` in `custom_components/windmillac/const.py`)
  - SDK/Client: Raw HTTP via `requests` library (no Blynk SDK client object used at runtime)
  - Auth: Blynk device token passed as query parameter `?token=<TOKEN>` on every request
  - Endpoints used:
    - `GET /external/api/get?token=<TOKEN>&<PIN>` — read a virtual pin value
    - `GET /external/api/update?token=<TOKEN>&<PIN>=<VALUE>` — write a virtual pin value
  - Virtual pins mapped in `custom_components/windmillac/blynk_service.py`:
    - `V0` — power (0/1)
    - `V1` — current temperature (read/write)
    - `V2` — target temperature (read/write)
    - `V3` — HVAC mode (cool/fan/eco/off)
    - `V4` — fan speed (low/medium/high/auto)
  - All HTTP calls are blocking; wrapped via `hass.async_add_executor_job` to keep HA event loop non-blocking.
  - Poll interval: 60 seconds (`UPDATE_INTERVAL` in `custom_components/windmillac/const.py`), managed by `WindmillDataUpdateCoordinator` in `custom_components/windmillac/coordinator.py`.

## Data Storage

**Databases:**
- None — no database or ORM used. State is held in-memory within the `DataUpdateCoordinator` and persisted only through Home Assistant's own config entry storage.

**File Storage:**
- None — no local file reads/writes beyond HA's standard config entry mechanism.

**Caching:**
- Home Assistant `DataUpdateCoordinator` acts as an in-memory cache; data is refreshed on the 60-second poll interval or on explicit `async_request_refresh()` calls triggered after write operations.

## Authentication & Identity

**Auth Provider:**
- Blynk device token (opaque string)
  - Implementation: Token is entered by the user via the HA config flow UI (`custom_components/windmillac/config_flow.py`), stored in `ConfigEntry.data["token"]`, and appended as a query parameter to every API request.
  - No OAuth, no session cookies, no key rotation mechanism present.

## Monitoring & Observability

**Error Tracking:**
- None — no external error tracking service integrated.

**Logs:**
- Python `logging` module used throughout; logger names are module `__name__`. Log level set to `DEBUG` explicitly in `blynk_service.py`, `climate.py`, and `entity.py`. Errors are raised as `UpdateFailed` exceptions within the coordinator, which Home Assistant surfaces in its own log and UI.

## CI/CD & Deployment

**Hosting:**
- Deployed directly inside a Home Assistant instance as a custom component; no cloud hosting for the integration itself.

**CI Pipeline:**
- GitHub Actions (`.github/workflows/`):
  - `main.yml` — HACS Action: validates integration against HACS requirements on push, PR, and daily cron.
  - `hassfest.yml` — home-assistant/actions/hassfest: validates manifest and integration structure on push, PR, and daily cron.
  - `makeHACSRelease.yml` — on GitHub Release creation: zips repo and uploads `hacs_windmill.zip` as a release asset.

## Environment Configuration

**Required env vars:**
- None at the OS/environment level. All configuration is provided through the Home Assistant UI config flow.
- Required user config: `token` — Blynk device token for the Windmill AC device at `https://dashboard.windmillair.com`.

**Secrets location:**
- Stored in Home Assistant's encrypted config entry storage (`.storage/` directory in the HA config directory). Not stored in any file within this repo.

## Webhooks & Callbacks

**Incoming:**
- None — the integration does not expose any webhook endpoints. All communication is outbound polling.

**Outgoing:**
- All outgoing calls go to `https://dashboard.windmillair.com/external/api/get` and `https://dashboard.windmillair.com/external/api/update` as described above.

---

*Integration audit: 2026-04-22*

# Technology Stack

**Analysis Date:** 2026-04-22

## Languages

**Primary:**
- Python 3.x - All integration logic (`custom_components/windmillac/`)

## Runtime

**Environment:**
- Home Assistant Core (minimum version 2024.4.0, per `hacs.json`)

**Package Manager:**
- No standalone Python package manager config present; dependencies declared in `custom_components/windmillac/manifest.json` under `"requirements"` and installed by Home Assistant at integration load time.
- Lockfile: Not applicable (HA manages install)

## Frameworks

**Core:**
- Home Assistant Custom Component framework - Provides `ConfigEntry`, `HomeAssistant`, `CoordinatorEntity`, `ClimateEntity`, `DataUpdateCoordinator`, and `ConfigFlow` base classes used throughout the integration.

**Testing:**
- Not detected - no test files, no pytest config, no test runner configuration present.

**Build/Dev:**
- GitHub Actions - CI/CD via `.github/workflows/`

## Key Dependencies

**Critical:**
- `requests` (version unpinned) - Synchronous HTTP client used in `custom_components/windmillac/blynk_service.py` to call the Blynk/Windmill cloud REST API. Wrapped in `hass.async_add_executor_job` to avoid blocking the async event loop.
- `blynklib` (version unpinned) - Listed in `manifest.json` requirements; imported in `blynk_service.py` via `requests` HTTP calls rather than the library's own client. Included for compatibility declaration; actual HTTP calls bypass the library directly.

**Infrastructure:**
- `voluptuous` - Schema validation library used in `custom_components/windmillac/config_flow.py` for config/options form schemas. Provided by Home Assistant core, not a separate install.

## Configuration

**Environment:**
- No `.env` files used. Configuration is provided at runtime through the Home Assistant UI config flow.
- Required user-provided config key: `token` (Blynk device token for `https://dashboard.windmillair.com`)
- Stored in HA `ConfigEntry.data` under key `CONF_TOKEN = "token"` (defined in `custom_components/windmillac/const.py`)

**Build:**
- `hacs.json` - HACS (Home Assistant Community Store) metadata: minimum HA version 2024.4.0, renders README.
- `custom_components/windmillac/manifest.json` - HA integration manifest: domain, name, version 1.0.6, requirements, integration type, IoT class.

## Platform Requirements

**Development:**
- Home Assistant development environment with Python 3.x
- No virtual environment or requirements file in repo; HA installs `requests` and `blynklib` automatically.

**Production:**
- Deployed as a custom component under `<config>/custom_components/windmillac/` in a running Home Assistant instance (2024.4.0+).
- IoT class: `cloud_polling` — polls `https://dashboard.windmillair.com` every 60 seconds (defined in `const.py` as `UPDATE_INTERVAL = 60`).

---

*Stack analysis: 2026-04-22*

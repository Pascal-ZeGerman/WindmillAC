# Codebase Structure

**Analysis Date:** 2026-04-22

## Directory Layout

```
WindmillAC/                        # Repository root
├── custom_components/             # HA custom component container (required by HA convention)
│   └── windmillac/                # Integration source — domain name must match manifest.json "domain"
│       ├── __init__.py            # Integration entry point (setup/unload)
│       ├── manifest.json          # HA integration metadata (domain, version, requirements)
│       ├── const.py               # Shared constants (DOMAIN, VERSION, URLs, config keys)
│       ├── blynk_service.py       # API service layer — all HTTP I/O with Blynk cloud
│       ├── coordinator.py         # DataUpdateCoordinator — polling and state cache
│       ├── entity.py              # WindmillClimate entity — exposes state and commands to HA
│       ├── climate.py             # Climate platform setup — registers entities
│       └── config_flow.py        # UI config flow — token collection and options
├── .github/
│   └── workflows/
│       ├── main.yml               # HACS validation CI
│       ├── hassfest.yml           # hassfest integration validator CI
│       └── makeHACSRelease.yml    # HACS release automation
├── hacs.json                      # HACS store metadata (min HA version, render_readme)
├── README.md                      # User-facing documentation
└── .planning/
    └── codebase/                  # GSD codebase analysis documents
```

## Directory Purposes

**`custom_components/windmillac/`:**
- Purpose: The entire integration lives here; HA discovers it by finding a `manifest.json` inside `custom_components/<domain>/`
- Contains: All Python source files — no sub-packages, no tests directory
- Key files: `__init__.py` (entry point), `manifest.json` (HA metadata), `entity.py` (main logic)

**`.github/workflows/`:**
- Purpose: CI pipeline definitions for HACS and hassfest compatibility checks
- Contains: Three workflow YAML files (HACS validation, hassfest, release)
- Generated: No — hand-maintained

**`.planning/codebase/`:**
- Purpose: GSD architecture and stack analysis documents for AI-assisted development
- Generated: Yes — written by `/gsd-map-codebase`
- Committed: Yes

## Key File Locations

**Entry Points:**
- `custom_components/windmillac/__init__.py`: `async_setup_entry()` and `async_unload_entry()` — HA loads this first
- `custom_components/windmillac/climate.py`: `async_setup_entry()` — platform registration, entity instantiation
- `custom_components/windmillac/config_flow.py`: `WindmillConfigFlow.async_step_user()` — UI-driven setup

**Configuration:**
- `custom_components/windmillac/manifest.json`: Domain name, version, requirements, HA integration metadata
- `custom_components/windmillac/const.py`: All shared constants — `DOMAIN`, `VERSION`, `BASE_URL`, `CONF_TOKEN`, `UPDATE_INTERVAL`, `PLATFORMS`
- `hacs.json`: HACS store metadata (minimum HA version)

**Core Logic:**
- `custom_components/windmillac/blynk_service.py`: HTTP calls, value mapping dicts, typed getters/setters for each AC attribute
- `custom_components/windmillac/coordinator.py`: `WindmillDataUpdateCoordinator` — 60-second poll cycle, `_async_update_data()`
- `custom_components/windmillac/entity.py`: `WindmillClimate` — all HA climate properties and `async_set_*` command methods

**Testing:**
- No test files exist in the repository.

## Naming Conventions

**Files:**
- `snake_case.py` for all Python modules — matches HA convention
- Platform modules named after the HA platform they implement: `climate.py`
- Service/helper modules named after their function: `blynk_service.py`, `coordinator.py`

**Python Classes:**
- `PascalCase` — `WindmillClimate`, `WindmillDataUpdateCoordinator`, `WindmillConfigFlow`, `BlynkService`
- Entity classes include the domain name prefix: `Windmill*`
- Flow handler named `WindmillConfigFlow` / `WindmillOptionsFlowHandler`

**Python Functions:**
- HA lifecycle functions use HA's required `async_` prefix: `async_setup_entry`, `async_unload_entry`
- Entity command methods follow HA convention: `async_set_temperature`, `async_set_hvac_mode`, `async_set_fan_mode`, `async_turn_on`, `async_turn_off`
- Service layer methods use `async_get_*` / `async_set_*` prefix

**Constants:**
- `SCREAMING_SNAKE_CASE` in `const.py` — `DOMAIN`, `CONF_TOKEN`, `BASE_URL`, `UPDATE_INTERVAL`

**HA Data Keys:**
- Config entry data keys use `CONF_*` prefix (e.g., `CONF_TOKEN = "token"`)

## Where to Add New Code

**New AC attribute (e.g., humidity, filter status):**
1. Add Blynk pin constant and value mapping dict in `custom_components/windmillac/blynk_service.py`
2. Add `async_get_*` and `async_set_*` typed methods in `BlynkService`
3. Extend coordinator's fetch result dict in `custom_components/windmillac/coordinator.py` — `_async_update_data()`
4. Add corresponding `@property` and `async_set_*` method in `custom_components/windmillac/entity.py`

**New HA platform (e.g., sensor, switch):**
1. Create `custom_components/windmillac/<platform>.py` with `async_setup_entry()` and entity class
2. Add platform name string to `PLATFORMS` list in `custom_components/windmillac/const.py`
3. HA will automatically call `async_forward_entry_setups` for the new platform via `__init__.py`

**New config option:**
1. Add constant to `custom_components/windmillac/const.py`
2. Add schema field in `custom_components/windmillac/config_flow.py` — both `WindmillConfigFlow` and `WindmillOptionsFlowHandler`

**Shared utilities or helpers:**
- Place in a new `custom_components/windmillac/helpers.py` (does not exist yet; no shared utility pattern is established)

**Tests:**
- No test infrastructure exists. New tests would go in a `tests/` directory at the repository root, mirroring `custom_components/windmillac/` structure (e.g., `tests/test_blynk_service.py`).

## Special Directories

**`custom_components/`:**
- Purpose: Required top-level container name for HA to discover custom integrations
- Generated: No
- Committed: Yes

**`.planning/codebase/`:**
- Purpose: AI-assisted development reference documents (STACK.md, ARCHITECTURE.md, etc.)
- Generated: Yes (by GSD tooling)
- Committed: Yes

---

*Structure analysis: 2026-04-22*

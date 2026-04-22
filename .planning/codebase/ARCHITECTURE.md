# Architecture

**Analysis Date:** 2026-04-22

## Pattern Overview

**Overall:** Home Assistant Custom Integration using DataUpdateCoordinator + CoordinatorEntity pattern

**Key Characteristics:**
- `cloud_polling` IoT class — state is fetched from the Windmill Blynk cloud API on a fixed interval (60 seconds)
- All external I/O is wrapped in `async_add_executor_job` because the underlying `requests` library is synchronous
- A single `DataUpdateCoordinator` instance is shared across all platform entities; entities read from `coordinator.data` rather than making their own API calls
- Config entries are used for multi-instance support; integration state is stored in `hass.data[DOMAIN][entry.entry_id]`

## Layers

**API Service Layer:**
- Purpose: Wraps all HTTP communication with the Windmill Blynk cloud API
- Location: `custom_components/windmillac/blynk_service.py`
- Contains: `BlynkService` class — pin read/write primitives, value mappings (fan speed, mode, power), higher-level typed getters/setters
- Depends on: `requests`, `homeassistant.components.climate.const` (for HVACMode constants)
- Used by: `WindmillDataUpdateCoordinator`

**Coordinator Layer:**
- Purpose: Polls the API periodically and caches state for entities
- Location: `custom_components/windmillac/coordinator.py`
- Contains: `WindmillDataUpdateCoordinator` subclassing `DataUpdateCoordinator`
- Depends on: `BlynkService`, HA `helpers.update_coordinator`
- Used by: `WindmillClimate` entity, `climate.py` platform setup

**Entity Layer:**
- Purpose: Exposes device state and controls to Home Assistant as a Climate entity
- Location: `custom_components/windmillac/entity.py`
- Contains: `WindmillClimate` subclassing `CoordinatorEntity` + `ClimateEntity`
- Depends on: `WindmillDataUpdateCoordinator`, `BlynkService` (via coordinator reference), HA climate constants
- Used by: `climate.py` platform setup

**Platform Setup:**
- Purpose: Instantiates entities when the integration is loaded for the climate platform
- Location: `custom_components/windmillac/climate.py`
- Contains: `async_setup_entry`, `ENTITY_DESCRIPTIONS` list
- Depends on: `WindmillClimate`, `WindmillDataUpdateCoordinator`, `DOMAIN` constant

**Integration Setup:**
- Purpose: Bootstraps `BlynkService` and `WindmillDataUpdateCoordinator`, registers platform, handles unload
- Location: `custom_components/windmillac/__init__.py`
- Depends on: `BlynkService`, `WindmillDataUpdateCoordinator`, `const.py`

**Configuration Flow:**
- Purpose: Collects the Blynk device token from the user via the HA UI
- Location: `custom_components/windmillac/config_flow.py`
- Contains: `WindmillConfigFlow` (initial setup), `WindmillOptionsFlowHandler` (re-configuration)

## Data Flow

**Polling Cycle (every 60 seconds):**

1. `WindmillDataUpdateCoordinator._async_update_data()` is called by the HA scheduler
2. It calls `BlynkService.async_get_pin_value()` / `async_get_mode()` / `async_get_fan()` / `async_get_power()` for pins V0–V4
3. `BlynkService` wraps synchronous `requests.get()` calls in `hass.async_add_executor_job()` to avoid blocking the event loop
4. Responses are parsed (int, string, or JSON array) and translated via value mappings in `BlynkService`
5. Coordinator stores the result dict: `{current_temp, target_temp, mode, fan, power}`
6. `WindmillClimate` properties read from `coordinator.data` — no direct API calls

**User Command Flow:**

1. HA calls `WindmillClimate.async_set_*()` method (temperature, hvac_mode, fan_mode, turn_on/off)
2. Entity method calls the appropriate `BlynkService.async_set_*()` typed setter
3. Typed setter translates value via mapping dict and calls `BlynkService.async_set_pin_value()` with the correct virtual pin (V0–V4)
4. After write, entity calls `coordinator.async_request_refresh()` to immediately re-poll state

**State Management:**
- All authoritative state lives in `coordinator.data` dict, populated on each poll
- Entities expose state purely via properties reading from `coordinator.data`
- Local instance variables in `WindmillClimate` (`_hvac_mode`, `_target_temperature`, etc.) are partially redundant with coordinator data — `async_update()` also sets `_attr_*` attributes

## Key Abstractions

**BlynkService:**
- Purpose: Translates HA climate semantics (HVACMode, fan speed strings) into Blynk virtual pin protocol (V0–V4 integer/string values) and vice versa
- Location: `custom_components/windmillac/blynk_service.py`
- Pattern: Service object injected into coordinator; contains value mapping dicts as instance attributes

**WindmillDataUpdateCoordinator:**
- Purpose: HA-standard polling coordinator; decouples entity lifecycle from fetch cadence
- Location: `custom_components/windmillac/coordinator.py`
- Pattern: Subclasses `DataUpdateCoordinator`, overrides `_async_update_data()`

**WindmillClimate:**
- Purpose: HA climate entity; all state read from coordinator, all commands delegated to blynk_service via coordinator reference
- Location: `custom_components/windmillac/entity.py`
- Pattern: Subclasses both `CoordinatorEntity` and `ClimateEntity`

## Entry Points

**Integration Load:**
- Location: `custom_components/windmillac/__init__.py` — `async_setup_entry()`
- Triggers: HA loading a config entry for domain `windmillac`
- Responsibilities: Creates `BlynkService` and `WindmillDataUpdateCoordinator`, stores them in `hass.data`, triggers initial refresh, forwards platform setup

**Integration Unload:**
- Location: `custom_components/windmillac/__init__.py` — `async_unload_entry()`
- Triggers: User removes integration or HA restarts
- Responsibilities: Unloads climate platform, removes entry from `hass.data`

**Platform Setup:**
- Location: `custom_components/windmillac/climate.py` — `async_setup_entry()`
- Triggers: Integration setup forwarding to the `climate` platform
- Responsibilities: Instantiates `WindmillClimate` entities from `ENTITY_DESCRIPTIONS`

**Config Flow:**
- Location: `custom_components/windmillac/config_flow.py` — `WindmillConfigFlow.async_step_user()`
- Triggers: User adds integration via HA UI
- Responsibilities: Collects and stores `token` in config entry data

## Error Handling

**Strategy:** Exceptions from HTTP calls surface as `UpdateFailed` in the coordinator, which causes HA to mark the integration as unavailable until the next successful poll.

**Patterns:**
- `BlynkService.async_get_pin_value()` raises a plain `Exception` on non-200 status or parse failure
- `WindmillDataUpdateCoordinator._async_update_data()` catches all exceptions and re-raises as `UpdateFailed`
- Command methods (`async_set_*`) do not catch exceptions — failures propagate to HA's service call error handling
- No retry logic; failures resolve on the next scheduled poll interval

## Cross-Cutting Concerns

**Logging:** `logging.getLogger(__name__)` in each module; debug-level logging is set explicitly in `blynk_service.py` and `entity.py` via `_LOGGER.setLevel(logging.DEBUG)`
**Validation:** User input validated only by voluptuous schema in `config_flow.py` (token is required string); no runtime validation of API response values beyond type checks
**Authentication:** Single Blynk device token passed as query parameter on every API request; stored in config entry data

---

*Architecture analysis: 2026-04-22*

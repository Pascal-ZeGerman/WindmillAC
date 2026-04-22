# Codebase Concerns

**Analysis Date:** 2026-04-22

---

## Tech Debt

**Unreachable duplicate `return` statement in `async_unload_entry`:**
- Issue: Two consecutive `return unload_ok` statements at the end of `async_unload_entry`. The second is dead code and will never execute.
- Files: `custom_components/windmillac/__init__.py` (lines 39–40)
- Impact: Cosmetic/dead code today; confusing when reading the unload path.
- Fix approach: Delete the duplicate `return unload_ok` on line 40.

**`blynklib` declared as a dependency but never imported or used:**
- Issue: `manifest.json` lists `"requirements": ["requests", "blynklib"]`, but `blynk_service.py` never imports `blynklib`. All HTTP calls are made directly via `requests`.
- Files: `custom_components/windmillac/manifest.json` (line 10), `custom_components/windmillac/blynk_service.py`
- Impact: HA installs `blynklib` on every setup unnecessarily, adding install time and a transitive dependency footprint for no benefit.
- Fix approach: Remove `"blynklib"` from `requirements` in `manifest.json`.

**Hardcoded logger `setLevel(logging.DEBUG)` left in production code:**
- Issue: `_LOGGER.setLevel(logging.DEBUG)` is set explicitly in `blynk_service.py` (line 7), `climate.py` (line 13), and `entity.py` (line 11). This forces debug output regardless of the HA-configured log level, flooding logs in production.
- Files: `custom_components/windmillac/blynk_service.py:7`, `custom_components/windmillac/climate.py:13`, `custom_components/windmillac/entity.py:11`
- Impact: Users' HA logs are polluted with DEBUG-level messages even at the default WARNING/INFO log level. Can hide real errors in log volume.
- Fix approach: Remove all three `_LOGGER.setLevel(logging.DEBUG)` lines. HA's log level configuration will govern debug output correctly.

**Debug log message contains a stray "2" prefix:**
- Issue: `_LOGGER.debug("2Starting data from Windmill AC")` in the coordinator constructor is clearly a typo from development.
- Files: `custom_components/windmillac/coordinator.py` (line 14)
- Impact: Cosmetic; produces a confusing log entry.
- Fix approach: Fix string to `"Starting data from Windmill AC"` or a more descriptive message.

**Redundant state management: `_attr_*` attributes, bare `_` attributes, and coordinator data are all used simultaneously:**
- Issue: `WindmillClimate.__init__` sets `_hvac_mode`, `_target_temperature`, `_fan_mode`, `_is_on` as bare instance variables (lines 37–40 in `entity.py`). Properties read from `coordinator.data` directly. `async_update()` also writes `_attr_*` prefixed attributes (lines 120–124). Three parallel state locations for the same data.
- Files: `custom_components/windmillac/entity.py` (lines 17–43, 116–129)
- Impact: Bare `_hvac_mode` etc. variables set in `__init__` are never read back (dead state). Confusing when adding new attributes. Risk of stale data if one path is updated and the others are not.
- Fix approach: Remove bare `_hvac_mode`, `_target_temperature`, `_fan_mode`, `_is_on` from `__init__`. Remove `async_update()` override entirely — `CoordinatorEntity` handles state refresh via `_handle_coordinator_update()`. Properties reading from `coordinator.data` are sufficient.

**`async_write_ha_state()` called redundantly in `async_turn_on` / `async_turn_off`:**
- Issue: Both `async_turn_on()` and `async_turn_off()` call `coordinator.async_request_refresh()` AND `self.async_write_ha_state()` (entity.py lines 107–108, 113–114). `async_request_refresh()` triggers a coordinator update which already causes `CoordinatorEntity` to schedule a state write automatically. The explicit `async_write_ha_state()` call is redundant and may write stale state before the refresh completes.
- Files: `custom_components/windmillac/entity.py` (lines 104–114)
- Impact: Minor — may briefly show old state until the refresh completes. Clutters the code with non-standard HA patterns.
- Fix approach: Remove the `self.async_write_ha_state()` calls from `async_turn_on` and `async_turn_off`.

---

## Known Bugs

**Mode mapping asymmetry: write uses HVACMode constants, read uses raw strings:**
- Symptoms: Setting mode writes using `HVACMode` enum values as keys (e.g., `HVACMode.COOL = "cool"`), but `async_set_mode` lowercases the input and looks up in `mode_mapping`. The pin API apparently returns string labels like `"cool"`, `"fan"`, `"eco"` rather than integers. `async_get_mode` matches on the raw lowercased string via a `match` block. However, `async_set_mode` passes those same Blynk strings back through `mode_mapping` which maps `HVACMode.COOL -> "1"`, `HVACMode.AUTO -> "2"`, `HVACMode.FAN_ONLY -> "0"` (integers). If the API actually accepts integer pin values for writes but returns string labels for reads, this asymmetry works; if the API changed its write contract, silent wrong-mode commands would be sent.
- Files: `custom_components/windmillac/blynk_service.py` (lines 21–25, 92–96, 117–127)
- Trigger: Any HVAC mode change.
- Workaround: Integration appears to function, suggesting the API accepts integer writes and returns string reads. No documentation or test coverage confirms this.

**`async_get_pin_value` response parsing is fragile:**
- Symptoms: The response parser in `async_get_pin_value` branches on `response.text.isdigit()` then `response.text.isalpha()` then falls back to `response.json()[0]`. This means: floating-point temperature strings (e.g. `"72.5"`) are neither digit-only nor alpha, so they fall to `response.json()[0]` which would fail to parse a plain float string (not valid JSON). Temperature responses like `"72"` would parse as `int`, but `"72.5"` would raise a `ValueError` caught and re-raised as a generic `Exception`.
- Files: `custom_components/windmillac/blynk_service.py` (lines 49–56)
- Trigger: Any AC unit reporting a decimal temperature reading from the Blynk API.
- Workaround: The integration may work if the Blynk API always returns integer-only temperature strings, but this is not guaranteed.

**Config flow performs no token validation:**
- Symptoms: Any string (including empty spaces, malformed values) is accepted as a token. The entry is created immediately without testing the API connection.
- Files: `custom_components/windmillac/config_flow.py` (lines 19–20, 39–40)
- Trigger: User enters an invalid token during setup.
- Workaround: The coordinator's first refresh (`async_config_entry_first_refresh()` in `__init__.py` line 26) will fail with `UpdateFailed`, surfacing the error — but the entry is already partially created, requiring the user to re-enter settings via options flow.

**`async_get_power` comparison is fragile:**
- Symptoms: `async_get_power` returns `True` only if `pin_value == 1` (integer equality). `async_get_pin_value` returns `int(response.text)` when `response.text.isdigit()`. If the API returns `"1"` this works. However `async_get_mode` then checks `current_power_state != False` (line 116), which is a non-idiomatic truthiness check. More importantly, `async_get_mode` makes a second independent API call to `async_get_power` every time mode is fetched — doubling calls during each coordinator update cycle.
- Files: `custom_components/windmillac/blynk_service.py` (lines 104–110, 112–127)
- Trigger: Every coordinator poll (60 seconds) fetches power twice.

---

## Security Considerations

**Blynk device token transmitted in plaintext query parameters:**
- Risk: The token is appended as a URL query parameter (`?token=<TOKEN>`) on every GET request. Query parameters appear in server-side access logs, browser history, proxy logs, and network monitoring tools.
- Files: `custom_components/windmillac/blynk_service.py` (lines 31–38, 62–66)
- Current mitigation: Requests are made over HTTPS (`https://dashboard.windmillair.com`), so the token is encrypted in transit. However, it is still exposed in any access logs on the server side.
- Recommendations: This is the Blynk API's defined protocol; no practical alternative without changing the upstream API. Ensure token rotation is documented if tokens are ever compromised.

**No request timeout configured:**
- Risk: `requests.get(url)` is called without a `timeout` parameter. A hung or slow API server will block the executor thread indefinitely (until OS TCP timeout, potentially minutes). Under high load, multiple stuck executor threads could exhaust the HA thread pool.
- Files: `custom_components/windmillac/blynk_service.py` (lines 42, 69)
- Current mitigation: None.
- Recommendations: Add `timeout=10` (seconds) to both `requests.get()` calls. Handle `requests.exceptions.Timeout` and `requests.exceptions.ConnectionError` explicitly.

**No HTTPS certificate verification override, but also no explicit enforcement:**
- Risk: Default `requests` behavior verifies TLS certificates (which is correct). No explicit `verify=True` parameter means a future code change could accidentally disable verification.
- Files: `custom_components/windmillac/blynk_service.py` (lines 42, 69)
- Current mitigation: Default `requests` behaviour is safe.
- Recommendations: Add explicit `verify=True` to document intent.

---

## Performance Bottlenecks

**Coordinator fetches 5 sequential API calls per poll cycle (7 including the double power read):**
- Problem: `_async_update_data` calls `async_get_pin_value('V1')`, `async_get_pin_value('V2')`, `async_get_mode()`, `async_get_fan()`, `async_get_power()`. `async_get_mode()` itself calls `async_get_power()` internally, making 6 total network requests each poll. All calls are sequential (each `await` completes before the next begins).
- Files: `custom_components/windmillac/coordinator.py` (lines 27–33), `custom_components/windmillac/blynk_service.py` (lines 112–127)
- Cause: No concurrency (`asyncio.gather` not used); `async_get_mode` has a hidden dependency call to `async_get_power`.
- Improvement path: Use `asyncio.gather()` for independent pin reads. Refactor `async_get_mode` to accept a pre-fetched power state rather than fetching it independently.

**Synchronous `requests` library in async context:**
- Problem: Every HTTP call runs in a thread pool executor (`async_add_executor_job`). Under HA's default thread pool this is acceptable for 1–2 devices, but `requests` itself does not support connection pooling across executor jobs unless a `Session` is shared. Each call creates a new TCP connection.
- Files: `custom_components/windmillac/blynk_service.py` (lines 41–60, 67–78)
- Cause: `requests` is blocking/synchronous; no persistent session or connection reuse.
- Improvement path: Replace `requests` with `aiohttp` (already available in HA environment via `homeassistant.helpers.aiohttp_client.async_get_clientsession`), eliminating the thread pool overhead and enabling true async concurrency.

---

## Fragile Areas

**`BlynkService` pin-to-semantic mapping is undocumented and has no validation:**
- Files: `custom_components/windmillac/blynk_service.py` (lines 14–29, 80–143)
- Why fragile: The mapping from virtual pin numbers (V0–V4) to semantic meaning is known only from code comments and variable names. If the Windmill firmware changes a pin assignment, the integration silently sets wrong values with no error. The fan speed and mode mappings differ in whether they use string vs. integer values with no consistent policy.
- Safe modification: Any change to pin assignments requires updating multiple mapping dicts and the match blocks in `async_get_mode` and `async_get_fan`. Add constants for pin names (e.g., `PIN_POWER = "V0"`) in `const.py`.
- Test coverage: None.

**`async_get_pin_value` response parsing handles only three cases, with a fragile fallback:**
- Files: `custom_components/windmillac/blynk_service.py` (lines 48–56)
- Why fragile: The three-branch parser (digit → int, alpha → string, else → json()[0]) does not cover floats, negative numbers, or multi-element JSON arrays. Any API response format change silently falls into the `json()[0]` branch and either works or raises a `ValueError`/`IndexError`.
- Safe modification: Replace with an explicit parser that tries `float()` first, then falls back to string, with clear handling for JSON arrays.
- Test coverage: None.

**`async_set_mode` lowercases the incoming `hvac_mode` value before dict lookup:**
- Files: `custom_components/windmillac/blynk_service.py` (line 93)
- Why fragile: HA's `HVACMode` enum values are already lowercase strings (`"cool"`, `"fan_only"`, `"auto"`). The `mode_mapping` dict keys are `HVACMode` enum members. `value.lower()` converts the enum to a plain string, but the dict keys are enum objects. Lookup succeeds only because Python compares enum values by their string value in `__eq__`. If HA changes how HVACMode equality works, this lookup will silently return the default `"0"` fallback.
- Safe modification: Remove the `.lower()` call; pass `hvac_mode` directly to `mode_mapping.get()`. The mapping already handles enum keys.
- Test coverage: None.

---

## Scaling Limits

**Single device only — no multi-instance architecture validation:**
- Current capacity: One Windmill AC unit per HA config entry. Multiple entries are theoretically supported by `hass.data[DOMAIN][entry.entry_id]` keying.
- Limit: Each additional device adds 6–7 more sequential HTTP requests per 60-second poll cycle. At 5+ devices this could cause coordinator update timeouts.
- Scaling path: Migrate to `aiohttp` + `asyncio.gather` for concurrent pin reads per device.

---

## Dependencies at Risk

**`requests` library (unpinned version):**
- Risk: `requests` is a blocking synchronous HTTP library. It is an anti-pattern in async HA integrations. HA's own ecosystem is moving toward requiring `aiohttp` for all network I/O. Future HA versions may enforce async-only networking in integrations.
- Impact: If HA deprecates thread-pool network patterns, the entire `BlynkService` HTTP layer must be rewritten.
- Migration plan: Replace both `requests.get()` calls in `blynk_service.py` with `aiohttp` via `async_get_clientsession(hass)`. Remove `requests` from `manifest.json` requirements.

**`blynklib` library (unused, unpinned):**
- Risk: Listed as a requirement but never used. Any breaking change or removal from PyPI triggers an install failure even though the library is not needed.
- Impact: Integration fails to install if `blynklib` is unavailable on PyPI.
- Migration plan: Remove `"blynklib"` from `manifest.json` requirements immediately.

---

## Missing Critical Features

**No token validation on config flow entry:**
- Problem: The config flow creates a config entry without ever testing that the provided token authenticates successfully against the Blynk API.
- Blocks: Users with invalid tokens get a confusing "integration unavailable" error after setup rather than a clear "invalid token" message during setup.
- Fix: Add an API test call in `async_step_user` before `async_create_entry`; surface `errors["base"] = "cannot_connect"` or `errors["base"] = "invalid_auth"` on failure.

**No minimum/maximum temperature bounds exposed:**
- Problem: `WindmillClimate` does not set `_attr_min_temp` or `_attr_max_temp`. HA defaults to 7°C–35°C (44.6°F–95°F) which does not match Windmill AC's actual supported range.
- Files: `custom_components/windmillac/entity.py`
- Blocks: HA UI temperature slider shows incorrect bounds; users can attempt to set temperatures the device does not support.

**No `translation_key` or strings.json for UI localisation:**
- Problem: Entity name and config flow title are hardcoded English strings. No `strings.json` or `translations/en.json` file exists.
- Files: `custom_components/windmillac/config_flow.py`, `custom_components/windmillac/entity.py`
- Blocks: HACS/hassfest may flag this in future validation passes; the integration cannot be localized.

---

## Test Coverage Gaps

**Zero test coverage across all functionality:**
- What's not tested: Every code path — coordinator polling, pin value parsing, mode/fan/power mapping, climate entity state transitions, config flow, error handling.
- Files: All files in `custom_components/windmillac/`
- Risk: Any change to `BlynkService` pin mappings, response parsing, or entity state logic can silently break AC control with no automated detection.
- Priority: High

**The fragile response parser has no edge-case tests:**
- What's not tested: Float temperatures, negative numbers, multi-value JSON arrays, non-200 HTTP responses, network timeouts.
- Files: `custom_components/windmillac/blynk_service.py` (lines 48–56)
- Risk: Temperature reading failures would surface only in production with real hardware.
- Priority: High

**Mode mapping symmetry is untested:**
- What's not tested: That values written to V3 (integer) correctly map back to `HVACMode` strings returned from V3 reads.
- Files: `custom_components/windmillac/blynk_service.py` (lines 21–25, 92–96, 112–127)
- Risk: Silent mode mismatch between set and get paths.
- Priority: High

---

*Concerns audit: 2026-04-22*

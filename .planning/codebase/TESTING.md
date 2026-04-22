# Testing
<!-- Last updated: 2026-04-22 -->

## Overview
**No tests exist in this repository.** The CI pipeline validates HACS/HA compliance only — it does not run any unit or integration tests.

---

## Current State

| Area | Status |
|------|--------|
| Unit tests | None |
| Integration tests | None |
| Test framework config | None |
| Coverage tooling | None |
| Mocking / fixtures | None |

---

## CI Validation (What Exists)

### `.github/workflows/main.yml` — HACS Validation
Validates the repository structure and `hacs.json` manifest against HACS requirements.

### `.github/workflows/hassfest.yml` — HA Manifest Validation
Validates `custom_components/windmill_ac/manifest.json` using Home Assistant's `hassfest` tool. Checks:
- Required manifest fields
- Dependency declarations
- Code owner format

**Neither workflow executes any Python tests.**

---

## Recommendations for Future Testing

- Use `pytest-homeassistant-custom-component` for HA integration testing
- Mock Blynk API calls with `unittest.mock` or `pytest-mock`
- Test coordinator update logic and error handling (`UpdateFailed` paths)
- Test climate entity state transitions (HVAC modes, temperature setting)
- Add `pyproject.toml` with pytest config and coverage thresholds

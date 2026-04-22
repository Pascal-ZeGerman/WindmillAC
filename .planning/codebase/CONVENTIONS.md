# Conventions
<!-- Last updated: 2026-04-22 -->

## Overview
Home Assistant custom component for Windmill AC control. Python codebase following HA integration conventions with some project-specific patterns.

---

## Naming Conventions

| Scope | Convention | Example |
|-------|-----------|---------|
| Files | `snake_case` | `climate.py`, `blynk_service.py` |
| Classes | `PascalCase` | `WindmillClimate`, `WindmillDataUpdateCoordinator` |
| HA async methods | `async_` prefix | `async_update`, `async_set_hvac_mode` |
| HA entity attributes | `_attr_` prefix | `_attr_hvac_mode`, `_attr_current_temperature` |
| Constants | `UPPER_SNAKE_CASE` | `DOMAIN`, `DEFAULT_SCAN_INTERVAL` |
| Private members | `_` prefix | `_blynk`, `_coordinator` |

---

## Import Order
Standard HA convention (stdlib → third-party → HA framework → local relative):

```python
import logging
from datetime import timedelta

import requests

from homeassistant.components.climate import ClimateEntity
from homeassistant.const import TEMP_CELSIUS

from . import DOMAIN
from .blynk_service import BlynkService
```

---

## Logging

Every module defines a module-level logger:

```python
_LOGGER = logging.getLogger(__name__)
```

Non-standard: `_LOGGER.setLevel(logging.DEBUG)` is set explicitly in 3 files. This forces DEBUG output regardless of HA's configured log level — likely leftover from development.

---

## Error Handling

- Coordinator wraps all update errors in `UpdateFailed`
- `blynk_service.py` raises bare `Exception` (not a custom exception class)
- No custom exception hierarchy exists

---

## Async Pattern

Synchronous `requests` library is wrapped via executor job using an inner closure pattern:

```python
result = await hass.async_add_executor_job(fetch)
```

where `fetch` is a locally-defined inner function. This is the correct HA pattern for blocking I/O.

---

## Configuration
- No formatter config (black, isort, ruff) detected
- No linter config (flake8, pylint) detected
- Manifest and HACS validation enforced via CI

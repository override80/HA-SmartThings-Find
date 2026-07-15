# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ Repository archived

This repository is archived and no longer actively maintained by the original author. Treat contributions accordingly — there's no CI beyond HACS validation, and no maintainer bandwidth for large refactors.

## What this is

A Home Assistant custom integration (`custom_components/smartthings_find`) that adds support for Samsung SmartThings Find devices (SmartTags, phones, tablets, watches, earbuds). It works entirely by reverse-engineering the SmartThings Find web app's private HTTP API — there is no official public API, so requests mimic what the website does (same endpoints, same request shapes, cookie-based session).

There is no build step, package manager config, or test suite in this repo. It's a pure HA custom component: install it under `config/custom_components/`, restart Home Assistant, and it's live.

## Validation

The only CI is HACS repository validation (`.github/workflows/validate.yaml`), which runs on push/PR and checks the repo conforms to HACS integration structure (`hacs.json`, `manifest.json`, etc.). There's no way to run it locally beyond eyeballing `hacs.json`/`manifest.json` for correctness.

To manually verify a change, copy `custom_components/smartthings_find/` into a real Home Assistant instance's `config/custom_components/` directory, restart HA, and watch `home-assistant.log` with debug logging enabled:

```yaml
logger:
  default: info
  logs:
    custom_components.smartthings_find: debug
```

## Architecture

### Auth flow (QR-code login, session cookie)

There's no API key/OAuth — auth mimics the manual QR-code login flow used by smartthingsfind.samsung.com, implemented across two stages in `config_flow.py` + `utils.py`:

1. **Stage one** (`do_login_stage_one`): hits the Samsung account pre-signin and QR-signin pages to extract a `signin.samsung.com/key/...` URL, which gets rendered as a QR code (`gen_qr_code_base64`) for the user to scan with their phone.
2. **Stage two** (`do_login_stage_two`): polls `signInWithQrCodeProc` until the user scans the code, then follows a chain of redirects to obtain the `JSESSIONID` cookie that authenticates all subsequent SmartThings Find requests.

The config flow (`SmartThingsFindConfigFlow`) drives this via `async_show_progress`/`async_show_progress_done` steps (`user` → `auth_stage_two` → `finish`) since both stages are long-running async tasks. The **same flow is reused for reauth** (`async_step_reauth`) — when the session dies, HA re-triggers this whole QR flow rather than a separate reauth-specific path.

Only `JSESSIONID` is persisted in the config entry (`CONF_JSESSIONID`); it's injected into a fresh `aiohttp` session's cookie jar on every integration load (`async_setup_entry` in `__init__.py`). No one knows exactly how long this session stays valid (README says "at least several weeks" for some users) — expiry is detected reactively (a 401 or a `'Logout'` response body from the API) and surfaces as `ConfigEntryAuthFailed`, which HA turns into a reauth prompt.

### CSRF token

Every mutating request to smartthingsfind.samsung.com needs a `_csrf` token as a query param, fetched via `fetch_csrf()` and cached in `hass.data[DOMAIN][entry_id]["_csrf"]`. This is re-fetched on integration setup and opportunistically when a request unexpectedly fails (see `button.py`'s ring handler).

### Data flow: coordinator → entities

`__init__.py` sets up one `SmartThingsFindCoordinator` (a `DataUpdateCoordinator`) per config entry. On each poll cycle it calls `get_device_location()` (in `utils.py`) once per device and builds a dict keyed by `dvceID` → location/battery/status data. All entities read from `coordinator.data[self.device_id]` rather than fetching anything themselves.

Entities are split across three platforms, all keyed off the same `devices` list fetched once at setup (`get_devices()`):
- `device_tracker.py` — GPS location (`SmartThingsDeviceTracker`)
- `sensor.py` — battery level (`DeviceBatterySensor`)
- `button.py` — "ring this device" action (`RingButton`), which does its own direct API call rather than going through the coordinator

Devices with `subType == 'CANAL2'` (Galaxy Buds-style earbuds with independent left/right locations) get **two extra** `SmartThingsDeviceTracker` instances (`"left"`/`"right"` sub-devices) in addition to the main tracker entity — see the branching in `device_tracker.py`'s `async_setup_entry` and `get_sub_location()` in `utils.py`, which pulls per-earbud coordinates out of `encLocation`.

### Active vs. passive location updates

Two independently configurable modes (`CONF_ACTIVE_MODE_SMARTTAGS`, `CONF_ACTIVE_MODE_OTHERS`, exposed via the options flow in `config_flow.py`) control whether `get_device_location()` sends a `CHECK_CONNECTION_WITH_LOCATION` "request update now" operation before reading location (active — costs battery, may wake the paired phone) or just reads back whatever was last reported (passive). SmartTags default to active; everything else defaults to passive. This is decided per-device inside `get_device_location()` based on `dev_data['deviceTypeCode']`.

### Location resolution logic

The STF API returns a list of `operation` entries (`ops`) per device, each with an `oprnType` (`LOCATION`, `LASTLOC`, `OFFLINE_LOC`, `CHECK_CONNECTION`, etc.). `get_device_location()` walks all operations and picks the most recent one with usable (non-encrypted, dated) coordinates as `used_loc`/`used_op`. Encrypted `encLocation` entries are skipped unless they're the earbud sub-location case handled separately by `get_sub_location()`. Battery level comes from a `CHECK_CONNECTION` operation's `battery` field, mapped through `BATTERY_LEVELS` in `const.py` (STF sometimes returns a fuzzy string like `MEDIUM` instead of a number).

## Localization

User-facing strings for the config/options flow live in `custom_components/smartthings_find/translations/{en,de}.json`. Keep both in sync when adding/changing config flow steps or fields — HA falls back to `en.json` for missing keys, but `de.json` should not be allowed to silently drift.

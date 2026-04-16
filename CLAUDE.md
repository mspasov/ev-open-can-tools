# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

PlatformIO-only C++/Arduino firmware that intercepts and re-transmits CAN bus frames on supported EVs in real time. The same source builds for several MCUs (RP2040, SAME51, ESP32 variants) and for a host-native test target (`NATIVE_BUILD`). Safety disclaimer in `README.md` applies — this is a private-testing / educational tool.

## Common commands

Build / test / lint are all driven through PlatformIO, plus a single clang-format check. Each command below is what CI runs (`.github/workflows/ci.yml`).

- Run all native unit tests for one env: `pio test -e native` (others: `native_bypass_tlssc_requirement`, `native_log_buffer`, `native_nag`). Each env has its own `test_filter` so tests compile under the right set of `-D` flags — you must pick the matching env, not just `native`.
- Run a single test folder: `pio test -e <env> -f <folder_name>` where `<folder_name>` is one of the `test/test_native_*` directories.
- Lint: `git ls-files '*.cpp' '*.h' '*.hpp' | xargs -r clang-format --dry-run --Werror --style=file` (style config in `.clang-format`: LLVM base, Allman braces, 4-space indent, no column limit, `SortIncludes: false`).
- Build firmware for a board: first select a profile, then run `pio run -e <env>`. Example (ESP32 TWAI, HW4):
  ```
  python scripts/platformio_set_profile.py --driver DRIVER_TWAI --vehicle HW4 --enable EMERGENCY_VEHICLE_DETECTION --enable ENHANCED_AUTOPILOT
  SKIP_DASH_CREDENTIAL_CHECK=1 pio run -e esp32_twai
  ```
- Python helper tests: `python test/test_can_analyzer.py` (no pytest — it's a standalone script paired with `scripts/ev_can_analyzer.py`).
- Release metadata check: `python scripts/check_release_metadata.py` — enforces that `VERSION` is SemVer, that `CHANGELOG.md` has an `[Unreleased]` section and a heading for the current `VERSION`, and that user-facing changes (anything under `include/`, `src/`, `scripts/`, `guides/`, `docs-site/docs/` or in `README.md`/`platformio.ini`/`platformio_profile.h`) also touch `CHANGELOG.md`.

## Profile system — understand before building

`platformio_profile.h` is the single source of truth for board + vehicle + feature selection. It is read at build time by `scripts/platformio_sync_profile.py` (declared as `extra_scripts = pre:` on every hardware env in `platformio.ini`). The sync script:

1. Parses the active `#define`s out of the header (LEGACY vs HW3 vs HW4; one of four `DRIVER_*`; optional feature flags).
2. Cross-checks them against the env's `build_flags` and aborts if the env driver doesn't match the header's driver.
3. For ESP32 dashboard builds with `DRIVER_ESP32_EXT_MCP2515`, translates the vehicle into `-DDASH_DEFAULT_HW=0|1|2` (dashboard picks the handler at runtime from one firmware binary). For other drivers, it appends the vehicle define directly.
4. Injects dashboard credentials (`DASH_SSID` / `DASH_PASS` / `DASH_OTA_USER` / `DASH_OTA_PASS`) as string `CPPDEFINES`, and refuses to build if `DASH_PASS` or `DASH_OTA_PASS` are still `"changeme"` — **set `SKIP_DASH_CREDENTIAL_CHECK=1` for CI / non-deploy builds**.

`scripts/platformio_set_profile.py` is the mechanical way to toggle those defines from the command line or CI (`--driver`, `--vehicle`, repeated `--enable`). Don't hand-edit `platformio_profile.h` in a way that ends up with zero or multiple driver/vehicle defines — the sync script will fail the build.

`env:native*` skips the sync script (uses `platformio_native_env.py` instead, which only fixes libc++ include paths on macOS). Native tests set their feature flags directly in `platformio.ini` `build_flags`.

## Architecture

All logic lives in headers under `include/`; `src/main.cpp` is a thin driver-selection shim. The runtime flow is:

1. **Driver layer** (`include/drivers/`): each target has a `CanDriver` subclass (`MCP2515Driver`, `ESP32_MCP2515Driver`, `SAME51Driver`, `TWAIDriver`, `MockDriver` for tests). Drivers expose `init / setFilters / enableInterrupt / read / send` and a static `kSupportsISR`. `main.cpp` instantiates exactly one based on the `DRIVER_*` macro.
2. **Application loop** (`include/app.h`): templated `appSetup<Driver>` / `appLoop<Driver>` glue. It owns a single `std::unique_ptr<CarManagerBase>` and calls `handleMessage` on every incoming frame, then optionally dispatches to `appPluginProcess` (set by the dashboard). On ESP32 dashboard builds, OTA updates pause the loop via `Update.isRunning()`.
3. **Handlers** (`include/handlers.h`): `LegacyHandler`, `HW3Handler`, `HW4Handler`, `NagHandler` all derive from `CarManagerBase`. Each declares its own `filterIds()` list (the CAN IDs it wants the driver to forward) and implements `handleMessage`, which mutates the incoming frame in place and calls `driver.send(...)` to re-transmit. The active handler is chosen at compile time via `SelectedHandler` in `app.h`; for `ESP32_DASHBOARD` builds, the choice is made from `DASH_DEFAULT_HW` (see profile section above).
4. **Shared state** (`include/shared_types.h`, `can_helpers.h`): `Shared<T>` is `std::atomic<T>` on-device but a plain `T` under `NATIVE_BUILD` so tests stay simple. Feature-flag runtime toggles (`isaSpeedChimeSuppressRuntime`, `nagKillerRuntime`, `enhancedAutopilotRuntime`, etc.) are declared here as `inline Shared<bool>`, default-initialized from the matching `-D` flag. Handlers gate behavior on these runtime values, not on the compile-time macro, so the dashboard can flip features live.
5. **Web dashboard + plugin engine** (`include/web/`, `include/plugin_engine.h`): compiled in only when `ESP32_DASHBOARD` is defined. The dashboard sets `appHandler->onFrame` / `onSend` for live stats, and installs `appPluginProcess` to run user-supplied JSON plugins after the built-in handler. Plugins are stored in SPIFFS; format is documented in `docs/plugins.md`.

Two compile-time conventions to keep in mind when editing handlers:

- **Runtime-vs-build gating**: wrap dashboard-only code paths in `#if defined(...) || defined(ESP32_DASHBOARD)` so the non-dashboard build still compiles with the flag undefined. See `HW4Handler::handleMessage` for the pattern.
- **Filter-list arity**: `filterIds()` returns a pointer + `filterIdCount()` returns the length. Both must agree, and they change with `#if defined(ESP32_DASHBOARD)` / `ISA_SPEED_CHIME_SUPPRESS`. If you add a CAN ID to a handler, update the count branch for every `#if` arm.

## Testing notes

- Tests live under `test/test_native_*/` and use Unity via PlatformIO. `test_filter` in each `env:native*` selects which folders are built — adding a new test folder means either matching an existing filter or declaring a new env.
- `MockDriver` (`include/drivers/mock_driver.h`) collects sent frames in a vector so tests can assert on `driver.sent` after calling `handler.handleMessage(...)`. Prefer this over touching real drivers.
- Native-only defines: `NATIVE_BUILD` disables `<Arduino.h>` includes and swaps `Shared<T>` to raw `T`. If you add an `#include <Arduino.h>` to a header that's reachable from native tests, guard it with `#ifndef NATIVE_BUILD`.

## Release discipline

- Bump `VERSION` (SemVer) and add a dated `## [x.y.z] - YYYY-MM-DD` section in `CHANGELOG.md` when landing user-facing changes. CI fails otherwise.
- Keep an `## [Unreleased]` section at the top of `CHANGELOG.md` for in-flight work (checked by `scripts/check_release_metadata.py`).
- Tags matching `v*.*.*` trigger the `release` + `notify-discord-release` jobs in CI, which upload firmware artifacts built by the `platformio-build` matrix and post the changelog section to Discord.

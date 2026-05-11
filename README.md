[![Release Version](https://img.shields.io/github/release/h2zero/NimBLE-Arduino.svg?style=plastic)
![Release Date](https://img.shields.io/github/release-date/h2zero/NimBLE-Arduino.svg?style=plastic)](https://github.com/h2zero/NimBLE-Arduino/releases/latest/)
<br/>

# NimBLE-Arduino — Crosspoint enhanced fork

> **This is a fork** of [h2zero/NimBLE-Arduino](https://github.com/h2zero/NimBLE-Arduino), maintained at
> [Rballesteros/NimBLE-Arduino-enhanced](https://github.com/Rballesteros/NimBLE-Arduino-enhanced) on branch
> `enhanced`. It is upstream **plus** a small set of patches focused on **ESP32-C3 init/deinit reliability**
> and a few **ergonomic API additions**. If you do not need any of the items below, use upstream — the fork
> is intentionally narrow.

---

## Why this fork exists

Upstream NimBLE-Arduino ships a known issue on **ESP32-C3** where a `NimBLEDevice::deinit() → init()` cycle
can crash the firmware: certain mutex/semaphore handles inside the NimBLE port layer are deleted but never
nulled, so the next `init()` either dereferences a dangling FreeRTOS handle or fires an `assert(handle)`
inside `npl_os_freertos.c`. The cleanup path also races: the host task can still be inside
`nimble_port_run()` when `nimble_port_deinit()` is called, and the static cleanup of `m_pServer`/`m_pScan`
runs even when the stack failed to stop.

This fork addresses both. It also adds three small APIs that arose while building a long-running BLE-HID
central on the ESP32-C3 that needed RAM-tight scanning, stack-based address formatting, and a one-liner for
whitelist filtering.

---

## Enhancements over upstream

All commits live as discrete patches on the `enhanced` branch (so rebasing onto a newer upstream stays
tractable). The list below is exhaustive — every behavioral and API delta vs. upstream is documented here.

### Reliability — re-init / cleanup hardening

#### Host-task-exit wait in `NimBLEDevice::deinit()`
- **File:** `src/NimBLEDevice.cpp`, `src/NimBLEDevice.h`
- **What changed:** A new static flag `m_hostTaskRunning` (`std::atomic<bool>`) is set to `true` on entry to
  `host_task()` and `false` on exit, immediately before `nimble_port_freertos_deinit()`.
  `NimBLEDevice::deinit()` now calls `nimble_port_stop()` and then **polls** that flag with a 1 ms
  `ble_npl_time_delay()` for up to 1 second before invoking `nimble_port_deinit()`.
- **Why:** Eliminates the use-after-free window where `nimble_port_deinit()` could free port resources while
  the host task was still inside `nimble_port_run()`. On timeout, `deinit()` returns `BLE_HS_ETIMEOUT`
  instead of corrupting state.
- **Caveat:** `m_hostTaskRunning` is set inside the task, so a `deinit()` that races a not-yet-scheduled
  task would fall through immediately. In practice `init()` already waits on `m_synced` (set from the host
  task), so the task is guaranteed to have started before any sane `deinit()` call.

#### Gated `clearAll` cleanup
- **File:** `src/NimBLEDevice.cpp`
- **What changed:** The static cleanup block (`delete m_pServer`, etc.) inside `deinit(true)` is now gated
  on a new local `stackStopped` flag that is true only when `nimble_port_stop()` succeeded **and** the host
  task observed exiting **and** `nimble_port_deinit()` (and the legacy ESP-IDF <5.0 HCI deinit) succeeded.
- **Why:** A failed deinit no longer leaves dangling pointers in `m_pServer`, `m_pScan`, etc. that would
  segfault on the next access. Previously the cleanup ran unconditionally on `clearAll=true`.

#### Host-task self-delete safety in `esp_nimble_disable()`
- **File:** `src/nimble/porting/npl/freertos/src/nimble_port_freertos.c`
- **What changed:** `esp_nimble_disable()` now compares `pcTaskGetName(NULL)` against
  `NIMBLE_HOST_TASK_NAME` (= `"nimble_host"`). If the function is being called **from inside the host task
  itself**, it nulls `host_task_h` and self-deletes via `vTaskDelete(NULL)` instead of trying to delete
  itself with a stale handle.
- **Why:** Some teardown paths (notably callbacks dispatched on the host task that subsequently call
  `NimBLEDevice::deinit()`) re-enter the disable path from inside the task they want to delete; without
  this check the original code corrupted FreeRTOS internals.

#### Centralized host-task name
- **File:** `src/nimble/porting/npl/freertos/src/nimble_port_freertos.c`
- **What changed:** A single `#define NIMBLE_HOST_TASK_NAME "nimble_host"` is now used for both
  `xTaskCreatePinnedToCore(..., NIMBLE_HOST_TASK_NAME, ...)` and the self-delete check above.
- **Why:** Avoids the silent regression where renaming one site without the other would break self-delete
  detection.

### Reliability — NPL (NimBLE Port Layer) self-heal

These three coordinated changes in `src/nimble/porting/npl/freertos/src/npl_os_freertos.c` work together to
survive an upstream NimBLE bug where some subsystems do not fully re-initialize their mutexes/semaphores
after a `deinit() → init()` cycle. Rather than asserting and rebooting, the port now self-heals.

#### `mutex_deinit` / `sem_deinit` null the handle
- After `vSemaphoreDelete(handle)`, `mu->handle = NULL` (and the same for `sem->handle`) so a subsequent
  `pend` cannot see a dangling FreeRTOS handle pointer.

#### `mutex_pend` / `sem_pend` lazy-create on a missing handle
- If the handle is `NULL`, `pend` creates one on demand — `xSemaphoreCreateRecursiveMutex()` for mutexes,
  `xSemaphoreCreateCounting(128, 0)` for semaphores — instead of `assert(handle)`. Failure to allocate
  returns `BLE_NPL_ENOENT`.
- **Caveat:** The 128 max-count for lazy-created counting semaphores matches the value used in
  `npl_freertos_sem_init()`. If the original sem was initialized with a smaller bound, that information is
  unrecoverable and a lazy-recreated sem will accept more posts than the stack expects. The warn-log (see
  next section) is the only signal that this code path fired.

#### `mutex_release` / `sem_release` no-op on stale handle
- A release on a `NULL` handle now returns `BLE_NPL_BAD_MUTEX` (mutex) / `BLE_NPL_ERROR` (sem) instead of
  asserting. Combined with the lazy-create on `pend`, the matching release is a real one in normal use; a
  release without a preceding `pend` is logged.

### Diagnostics — `npl_selfheal` log tag

- **File:** `src/nimble/porting/npl/freertos/src/npl_os_freertos.c`
- **What changed:** A new `NPL_SELFHEAL_WARN(fmt, …)` macro emits an `ESP_LOGW` under tag
  `"npl_selfheal"` on every self-heal branch (lazy-create on pend, no-op on stale release). On non-ESP
  platforms the macro is a no-op.
- **Why:** Self-heal is a workaround for an upstream bug, not a fix. Logging at warn level lets you see
  when it fires in production and decide whether each event is harmless recovery or a real bug worth
  reporting upstream.
- **Enabling at runtime:**
  ```cpp
  esp_log_level_set("npl_selfheal", ESP_LOG_WARN);
  ```
  Filter for `npl_selfheal` in your serial monitor to track recurrence.

### New APIs — ergonomic additions

#### `NimBLEAddress::toChars(char (&buffer)[kAddrStrLen]) const`
- **File:** `src/NimBLEAddress.h`, `src/NimBLEAddress.cpp`
- Fills an 18-byte buffer with the canonical printable address (`XX:XX:XX:XX:XX:XX\0`). The
  reference-to-array signature enforces the buffer size **at compile time** — passing a smaller array fails
  to compile.
- A new `static constexpr size_t NimBLEAddress::kAddrStrLen = 18` exposes the required buffer length so
  callers can size storage without magic numbers.
- `operator std::string()` and the deprecated `toString()` now delegate to `toChars()` — no behavioral
  change for existing callers.
- **Use case:** logging on hot paths can write into a stack buffer instead of allocating a `std::string`.
  ```cpp
  char buf[NimBLEAddress::kAddrStrLen];
  device.getAddress().toChars(buf);
  Serial.printf("seen %s\n", buf);
  ```

#### `NimBLEAdvertisedDevice::clearPayload()`
- **File:** `src/NimBLEAdvertisedDevice.h`, `src/NimBLEAdvertisedDevice.cpp`
- Frees the underlying `m_payload` vector (`std::vector<uint8_t>().swap(m_payload)` for guaranteed capacity
  release) and resets `m_advLength` and `m_callbackSent`.
- **Use case:** a long-running scanner that retains references to `NimBLEAdvertisedDevice` instances for
  filtering (e.g. by address) can drop the raw advertisement payload after extracting the fields it cares
  about, recovering tens of bytes per device on the heap.
- **Caveats — read before using:**
  - **Do not call parsing accessors after `clearPayload()`.** Methods such as `getServiceData(uuid)`
    perform `m_payload.size() - 2` arithmetic that underflows on an empty vector. Either call
    `clearPayload()` only after you have already extracted everything you need, or treat the device as
    address-only afterwards.
  - **Not thread-safe vs. concurrent scan callbacks.** The scan callback runs on the NimBLE host task and
    writes `m_payload` via `update()`; calling `clearPayload()` from the app task while scanning is active
    races. Safe call sites are *inside* the scan callback itself or *after* `NimBLEScan::stop()`.

#### `NimBLEScan::setFilterWhitelist(bool)`
- **File:** `src/NimBLEScan.h`, `src/NimBLEScan.cpp`
- Convenience wrapper around `setFilterPolicy()` for the two common filter policies:
  - `setFilterWhitelist(true)`  → `BLE_HCI_SCAN_FILT_USE_WL` (only advertisements from whitelisted MACs)
  - `setFilterWhitelist(false)` → `BLE_HCI_SCAN_FILT_NO_WL`  (all advertisements, default)
- For the directed-advertisement RPA-handling variants (`BLE_HCI_SCAN_FILT_NO_WL_INITA`,
  `BLE_HCI_SCAN_FILT_USE_WL_INITA`) keep using `setFilterPolicy()` directly.
- **Use case:** see `examples/NimBLE_Scan_Whitelist`. Combine with `NimBLEDevice::whiteListAdd(addr)` to
  drop everything except a known set of peripherals at the controller level.

---

## Installing this fork (PlatformIO — recommended)

Pin to a specific commit SHA in your `platformio.ini`:

```ini
lib_deps =
  https://github.com/Rballesteros/NimBLE-Arduino-enhanced.git#<sha>
```

> **PlatformIO cache caveat:** PlatformIO caches dependencies under `.pio/libdeps/<env>/`. Bumping the SHA
> usually invalidates the cache, but if you switch between branch refs (e.g. `#enhanced`) and SHAs you may
> need to clear it manually:
> ```bash
> rm -rf .pio/libdeps && pio run
> ```

To track the moving tip instead of pinning, replace `#<sha>` with `#enhanced`. This is less reproducible
but auto-pulls every push.

## Installing this fork (Arduino IDE)

The Arduino Library Manager will not find this fork — its registry entry points to upstream
`NimBLE-Arduino`. To use this fork in the Arduino IDE:

```bash
git clone -b enhanced https://github.com/Rballesteros/NimBLE-Arduino-enhanced.git \
    ~/Documents/Arduino/libraries/NimBLE-Arduino-enhanced
```

Note that the fork's `library.properties` still uses the upstream library identifier, so do **not** install
upstream NimBLE-Arduino at the same time — Arduino will pick one nondeterministically.

---

## Upstream sync policy

- Branch `enhanced` is the integration branch and is the de-facto default.
- Fork-specific patches are kept as discrete commits so they can be cherry-picked or rebased onto a newer
  upstream tag.
- Bugs in the fork-specific patches: file in this repo's
  [issues](https://github.com/Rballesteros/NimBLE-Arduino-enhanced/issues).
- Bugs in the underlying NimBLE-Arduino code (anything you would also reproduce on upstream): file in
  [h2zero/NimBLE-Arduino](https://github.com/h2zero/NimBLE-Arduino/issues).

---

# Upstream documentation (preserved verbatim from h2zero/NimBLE-Arduino)

The remainder of this README is the upstream documentation. It applies to the fork unmodified.

A fork of the NimBLE stack refactored for compilation in the Arduino IDE.

> [!IMPORTANT]
> Version 2 is now released!
> Check out the [1.x to 2.x Migration Guide](docs/1.x_to2.x_migration_guide.md) and [Release Notes](https://github.com/h2zero/NimBLE-Arduino/releases/latest)

## Supported MCU's
 - Espressif: ESP32, ESP32C3, ESP32S3, ESP32C6, ESP32H2, ESP32C2, ESP32C5
 - Nordic: nRF51, nRF52 series (**Requires** using [n-able arduino core](https://github.com/h2zero/n-able-Arduino))

**Note for ESP-IDF users: This repo will not compile correctly in ESP-IDF. An ESP-IDF component version of this library can be [found here.](https://github.com/h2zero/esp-nimble-cpp)**

This library **significantly** reduces resource usage and improves performance for ESP32 BLE applications as compared with the bluedroid based library. The goal is to maintain, as much as reasonable, compatibility with the original library but but using the NimBLE stack. In addition, this library will be more actively developed and maintained to provide improved capabilities and stability over the original.
<br/>

For Nordic devices, this library provides access to a completely open source and configurable BLE stack. No softdevice to work around, allowing for full debugging and resource management, continuous updates, with a cross platform API.

# Arduino installation
**Arduino Library manager:** Go to `sketch` -> `Include Library` -> `Manage Libraries`, search for NimBLE and install.

**Alternatively:** Download as .zip and extract to Arduino/libraries folder, or in Arduino IDE from Sketch menu -> Include library -> Add .Zip library.

`#include "NimBLEDevice.h"` at the beginning of your sketch.

# Platformio installation
* Open platformio.ini, a project configuration file located in the root of PlatformIO project.  
* Add the following line to the lib_deps option of [env:] section:
```
h2zero/NimBLE-Arduino@^2.1.0
```
* Build a project, PlatformIO will automatically install dependencies.

# Using
This library is intended to be compatible with the original ESP32 BLE functions and classes with minor changes.

If you have not used the original Bluedroid library please refer to the [New user guide](docs/New_user_guide.md).

If you are familiar with the original library, see: [The migration guide](docs/Migration_guide.md) for details about breaking changes and migration.

If you already use this library and need to migrate your code to version 2.x see the [1.x to 2.x Migration Guide.](docs/1.x_to2.x_migration_guide.md)

[Full API documentation and class list can be found here.](https://h2zero.github.io/NimBLE-Arduino/)

For added performance and optimizations see [Usage tips](docs/Usage_tips.md).

More advanced examples highlighting many available features are in examples/ NimBLE_Server, NimBLE_Client.

Beacon examples provided by @beegee-tokyo are in examples/ BLE_Beacon_Scanner, BLE_EddystoneTLM_Beacon.

Change the settings in the `src/nimconfig.h` file to customize NimBLE to your project,
such as increasing max connections, default is 3 for the esp32.
<br/>

# Development Status
This Library is tracking the esp-nimble repo, nimble-1.5.0-idf branch, currently [@e3cbdc0.](https://github.com/espressif/esp-nimble)  
<br/>

# Sponsors
Thank you to all the sponsors who support this project!

<!-- sponsors --><!-- sponsors -->

If you use this library for a commercial product please consider [sponsoring the development](https://github.com/sponsors/h2zero) to ensure the continued updates and maintenance.  
<br/>

# Acknowledgments
* [nkolban](https://github.com/nkolban) and [chegewara](https://github.com/chegewara) for the [original esp32 BLE library](https://github.com/nkolban/esp32-snippets/tree/master/cpp_utils) this project was derived from.
* [beegee-tokyo](https://github.com/beegee-tokyo) for contributing your time to test/debug and contributing the beacon examples.
* [Jeroen88](https://github.com/Jeroen88) for the amazing help debugging and improving the client code.
<br/>

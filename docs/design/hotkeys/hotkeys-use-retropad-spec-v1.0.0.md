# Hotkeys Use RetroPad — Design Spec (v1.0.0)

name: Hotkeys Use RetroPad
slug-token: hotkeys-use-retropad
version: v1.0.0

Date: 2026-04-24
Author: Copilot (working with koalabobo)

One-line summary

Provide a joypad-only mode where hotkeys are stored and evaluated using RetroPad semantic binds instead of device-native button/axis codes. This mode is opt-in (default OFF) and preserves keyboard behavior exactly.

Table of contents

- Goals
- Data model & on-disk format
- Runtime placement & pseudocode
- Caching & invalidation
- Keyboard safety contract & unit tests
- Migration & "Adopt Current Hotkeys" utility
- Diagnostics & overlay
- Test matrix (unit/integration/manual)
- Performance targets & benchmarks
- Rollout plan and gating
- File-by-file implementation map
- PR checklist and acceptance gates

Goals

- Reduce the need for per-controller autoconfig copying when users switch controllers on the same port.
- Preserve existing keyboard behavior exactly.
- Ensure negligible per-frame CPU/latency overhead.
- Provide diagnostic visibility and migration utilities.


Data model & on-disk format

On-disk format
- Location: `retroarch.cfg` section `[hotkeys-use-retropad]` OR a dedicated file `config/hotkeys-use-retropad.cfg` (implementation may choose one; the config system must expose it as a normal setting section) 
- Human-readable key-value pairs using existing hotkey action names and RetroPad semantic IDs.

Example in `retroarch.cfg` section:

[hotkeys-use-retropad]
menu_toggle = "retropad:BUTTON_START"
save_state = "retropad:BUTTON_SELECT"
rewind = "retropad:B_BUTTON"

Internal data structures (C)
- Use a small fixed-size enum for RetroPad semantic ids and a packed struct for hotkey entries.

typedef enum {
  HOTKEY_RP_BTN_A,
  HOTKEY_RP_BTN_B,
  HOTKEY_RP_BTN_X,
  HOTKEY_RP_BTN_Y,
  HOTKEY_RP_BTN_L,
  HOTKEY_RP_BTN_R,
  HOTKEY_RP_BTN_SELECT,
  HOTKEY_RP_BTN_START,
  HOTKEY_RP_DPAD_UP,
  HOTKEY_RP_DPAD_DOWN,
  HOTKEY_RP_DPAD_LEFT,
  HOTKEY_RP_DPAD_RIGHT,
  HOTKEY_RP_MAX
} hotkeys_use_retropad_rp_btn_t;

typedef struct {
  unsigned action_id;
  hotkeys_use_retropad_rp_btn_t rp_btn;
  uint8_t flags; // edge/held, modifiers
} hotkeys_use_retropad_entry_t;

typedef struct {
  hotkeys_use_retropad_entry_t entries[HOTKEY_MAX_COUNT];
  unsigned count;
  unsigned version;
} hotkeys_use_retropad_map_t;


Runtime placement & pseudocode

High-level placement
- Hoist the mode check once per-frame/per-port inside `input_driver_collect_system_input()` before per-hotkey evaluation loops.
- If `input_hotkeys_use_retropad_binds` is enabled and the current port is the hotkey owner port and a joypad is present, use the retropad remap path. Otherwise use existing device-native path.

Pseudocode (per frame, per port):

for each port p:
  hotkey_owner = resolve_hotkey_owner_port()
  if (!config.input_hotkeys_use_retropad_binds) {
    evaluate_hotkeys_device_native(p)
    continue
  }

  if (p != hotkey_owner || !has_joypad(p)) {
    evaluate_hotkeys_device_native(p)
    continue
  }

  remap_table = hotkeys_use_retropad_get_remap_table(p) // small array HOTKEY_RP_MAX -> device_code
  evaluate_hotkeys_remapped(p, remap_table)

Key helper functions
- hotkeys_use_retropad_get_remap_table(port):
  - Returns cached table if device signature matches; otherwise rebuilds from autoconfig mapping for the current device.
- evaluate_hotkeys_remapped(port, remap_table):
  - For each entry in hotkeys_use_retropad_map_t:
    - driver_code = remap_table[entry.rp_btn]
    - if driver_code == INVALID continue
    - if (joypad_state_cache_has_code(port, driver_code, entry.flags)) trigger_action(entry.action_id)


Caching & invalidation

Cache keys
- Cache per-port remap table; key it by device signature (vendor/product/serial/host-assigned-id where available) and autoconfig modification timestamp.

Invalidation triggers
- Device connect/disconnect event on that port
- Autoconfig reload
- Explicit user action: "Adopt Current Hotkeys as Hotkeys Use RetroPad"

Cache size
- One remap table per active port. Small memory footprint (array HOTKEY_RP_MAX ints).


Keyboard safety contract & unit tests

Contract (explicit)
- Keyboard hotkeys are evaluated on the device-native path and are never altered by hotkeys-use-retropad mode.
- The keyboard evaluation function remains unchanged and is always invoked.

Unit test requirement (added)
- Add a unit test suite that runs the entire baseline keyboard hotkey regression set with the new input code compiled in, both with `input_hotkeys_use_retropad_binds = false` and `= true`, and assert identical results for keyboard-only scenarios.
- Test harness API:
  - load_test_config(config_path)
  - emulate_keyboard_event(keycode)
  - capture_hotkey_action_calls()
  - assert_equal(action_calls_before, action_calls_after)


Migration & "Adopt Current Hotkeys" utility

Menu action: "Adopt Current Hotkeys as Hotkeys Use RetroPad"
- Walks per-device binds for the selected hotkey owner port.
- Attempts reverse lookup: device_button_code -> retropad semantic id. If unique mapping exists, convert; otherwise mark as unmappable and prompt the user.
- Writes converted binds into `[hotkeys-use-retropad]` section or config file with version tag.

Rollback
- Toggling `input_hotkeys_use_retropad_binds=false` restores device-native behavior; the on-disk per-device autoconfigs are untouched.


Diagnostics & overlay

Diagnostics mode (menu toggle)
- Show hotkey mode ON/OFF
- Display hotkey owner port and active device signature
- Show example resolved bindings (e.g., Menu Toggle -> retropad:START -> device button 10)
- Show cache age and last invalidation reason

Verbose log hooks (configurable log level)
- When enabled, log each hotkey evaluation in a compact form: timestamp, action_id, source(port), resolved_driver_code, result.


Test matrix (unit/integration/manual)

Unit tests
- Remap table computation and reverse lookup unit tests.
- Keyboard regression unit suite (see above).

Integration tests (CI)
- Controller swap test: Emulate attaching controller A with autoconfig A, binding hotkey, attach controller B on same port, verify hotkey triggers same action.
- Hotplug tests: disconnect/reconnect loop and assert no crash and hotkeys re-evaluate correctly.

Manual QA matrix (sampled configs)
- Android handheld built-in vs Bluetooth controller
- USB controller on Linux with multiple autoconfigs
- Windows with multiple USB controllers attached simultaneously
- Axis-only controllers (verify fallback & diagnostics; not supported in v1)


Performance targets & benchmarks

- Target: added per-frame cost < 50 microseconds on target hardware (ARM64 Android and x86_64 desktop under light load).
- Provide microbenchmark harness in test/benchmarks that runs baseline vs hotkeys-use-retropad path with worst-case 300 hotkeys bound.


Rollout plan and gating

- v1.0.0-dev: Design spec + internals + unit tests + perf harness.
- v1.0.0-rc: Integration tests + Android hotplug tests + diagnostics overlay.
- v1.0.0: Public-ready release when unit + integration + perf gates pass.

Acceptance gates (must pass before merge)
- Keyboard safety unit tests pass (exact match ON/OFF for keyboard-only inputs).
- Perf microbench does not exceed threshold on test platforms.
- Integration tests for controller swap & hotplug pass on CI.
- Documentation files (this spec + research) present in the PR.


File-by-file implementation map

- config.def.h: Add DEFAULT_INPUT_HOTKEYS_USE_RETROPAD false
- configuration.h/.c: Add bool and register the setting
- msg_hash.*: Add labels & sublabels for UI
- menu/menu_setting.c: Add CONFIG_BOOL entry
- menu/cbs/menu_cbs_sublabel.c: Add sublabel callback
- input/hotkeys_use_retropad_map.c/h: New module for remap cache & persistence (parsing/writing)
- input/input_driver.c: Integrate hoisted branch & evaluation path
- tests/: Add unit/integration tests and perf harness


PR checklist

- [ ] Docs: add design spec and research (this PR)
- [ ] Unit tests: remap & keyboard safety suite
- [ ] Integration tests: controller swap & hotplug
- [ ] Perf tests: include baseline numbers
- [ ] Code: no per-hotkey allocations in hot path
- [ ] Menu: add setting and help text
- [ ] Diagnostics: add overlay for debug


Notes
- UI copy and config key name remains `input_hotkeys_use_retropad_binds` and menu label "Hotkeys Use Controller's RetroPad Binds" for clarity in the UI.
- All repo tokens, filenames, and branch names use the canonical slug-token `hotkeys-use-retropad` for discoverability.

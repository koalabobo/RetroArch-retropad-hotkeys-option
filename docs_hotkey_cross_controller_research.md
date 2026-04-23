# Research & Planning: Cross-Controller Hotkeys in RetroArch

_Date: 2026-04-15_

## Decision Log

- **2026-04-15**: v1 for `input_hotkeys_use_retropad_binds` is **joypad-only**.
- **2026-04-15**: Keyboard behavior is protected by a dedicated safety contract (defined in this plan).
- **2026-04-23**: Open Question #2 resolved to **Option B**: implement button-first shared RetroPad hotkeys, then harden/extend axis behavior before shipping.
- **2026-04-23**: Use **semantic versioning language** in planning/milestones (`v1.0.0`, `v1.1.0`, etc.) for this feature track.
- **2026-04-23**: Latency policy lock = **A**: lowest-latency outcome is mandatory; implementation approach remains open to peer review/critique if it improves latency and safety.

## 1) Problem Statement

Users repeatedly report that RetroArch hotkeys are not portable when switching between built-in controls and external controllers (Bluetooth/USB), or between different controller models.

Typical symptoms:
- Hotkeys work on handheld controls but not external controller.
- Hotkeys bound with controller A trigger wrong actions on controller B.
- Reconnects and port changes break expected hotkey behavior.
- Users resort to manual autoconfig editing/copying as a workaround.

## 2) What the current codebase already provides

The current tree has two adjacent knobs:
- `input_hotkey_device_merge` (`Hotkey Device Type Merge`).
- `input_hotkey_follows_player1` (`Hotkeys Follow Player 1`).

These settings help with hotkey gating and port ownership, but they do **not** provide a dedicated controller-agnostic hotkey profile that dynamically follows arbitrary device swaps.

## 3) External evidence (user complaints / demand signals)

### A) Directly matching complaint pattern
1. **Libretro Forums**: “Hotkeys are not remapped when using a different controller (& more)” (thread 43271).
   - Explicitly reports hotkeys bound on one controller becoming inconsistent on another.

2. **Reddit r/RetroArch**: “[Android] How to sort out hotkeys for external controllers?”
   - Reports handheld + external controller hotkey/menu-toggle inconsistency.

3. **Reddit r/RetroArch**: “Hotkeys on Bluetooth controllers”.
   - Reports gameplay inputs working, but hotkeys not working on Bluetooth controller.

### B) Related reliability/friction around hotkeys and controller mapping
4. **Libretro Forums**: “Dual Sense Hotkeys not working?”
   - Demonstrates hotkey binding friction and dependency on bind-hold config state.

5. **GitHub issue #3414**: Android controller reconnect increments port and breaks expected mappings.
   - Older issue, but strongly related to dock/undock and reconnect workflows.

### C) Community workaround patterns (informal)
6. Community guides and comments repeatedly suggest:
   - Editing `retroarch.cfg` + per-controller autoconfig files.
   - Using middleware to normalize controllers to one virtual pad profile.

This indicates unmet demand for a first-class built-in solution.

## 4) Prior upstream product/PR context

- **Accepted precedent**: PR #18353 (“Hotkeys follow player 1”), merged Nov 2025.
  - Validates maintainer acceptance of improvements in this area.

- **Design caution**: PR #18785 (menu toggle bypass hotkey enable) was closed Apr 2026.
  - Maintainer feedback signaled concern about unsustainable, one-off hotkey exceptions.

Implication: a **generalized architecture** is more likely to be accepted than per-hotkey ad hoc exceptions.

## 5) Proposed product direction

Add a new UI boolean option:

### New setting: `input_hotkeys_use_retropad_binds` (bool, default `false`)
- **Menu label**: `Hotkeys Use Controller's RetroPad Binds`
- **Sublabel**: `ON: Hotkeys are bound to RetroPad button/axis codes. Recommended if controllers change frequently. OFF (default): Hotkeys use device-specific button/axis codes.`

### User-facing behavior when ON
- User binds hotkeys once in UI.
- Runtime resolves those hotkeys against active controller RetroPad mapping on selected hotkey port.
- Changing hotkeys does not require editing/copying autoconfig files.
- Switching controllers (same user/port) preserves expected hotkey intent.

## 6) Architecture sketch

### Core idea
Introduce a canonical "shared hotkey profile" independent of per-device autoconfig button numbers.

### High-level data model
- Existing per-device autoconfig remains intact.
- New shared hotkey map stores abstract input intent (e.g., resolved via RetroPad semantics / normalized mapping layer) rather than device-native numeric button IDs.

### Runtime evaluation
During input polling:
1. Identify current hotkey source port (respect existing follow-player1 logic if enabled).
2. If `input_hotkeys_use_retropad_binds = true`, evaluate hotkeys via RetroPad-resolved mapping + current active device mapping.
3. Otherwise use existing device-specific button/axis code behavior.

### Polling order and low-cost integration strategy

- Hotkey polling currently executes in `input_driver_collect_system_input()` via `input_keys_pressed()` for each active user port.
- This runs inside the main frame loop (`runloop_iterate()`) before menu combo post-processing and before the core consumes input callbacks.
- Core/gameplay input resolution is already normalized through `input_state_wrap()`, which includes a per-port joypad mask cache (`joypad_state_cache`) to avoid repeated per-button driver queries.

### Current hotkey source-port semantics (important for Open Question #3)

- Current runtime logic chooses a single `hotkey_port` owner for joypad hotkey evaluation.
- Default owner is port `0`; when `input_hotkey_follows_player1` is enabled, owner becomes `input_remap_port_map[0][0]` (with bounds fallback to `0`).
- `input_keys_pressed()` is called per user port, but hotkey ownership/gating behavior depends on that selected `hotkey_port`.
- In practical terms, "no controller on hotkey source port" means the currently selected owner port has no active/usable joypad input source at that moment, so hotkey trigger behavior becomes ambiguous unless fallback policy is defined.

**Performance-oriented implementation guidance:**
1. Avoid adding expensive work inside the hotkey inner loop (no allocations, parsing, file IO, or per-frame string work).
2. Reuse already-resolved bind tables and/or cached bitmasks wherever possible instead of repeatedly branching on every hotkey bind.
3. Hoist mode checks once per frame (or once per port) and route to a selected evaluation path (function pointer/static inline helper), rather than layering repeated conditionals per hotkey candidate.
4. Keep keyboard path unchanged and out of the new joypad-only branch to preserve the keyboard safety contract.

### Keyboard safety contract (v1)

When `input_hotkeys_use_retropad_binds = true`, keyboard behavior must remain equivalent to baseline:

1. **No keyboard remapping changes**: keyboard hotkey keycodes are read/evaluated exactly as in OFF mode.
2. **No keyboard precedence changes**: existing keyboard-vs-joypad block/unblock semantics remain unchanged.
3. **No keyboard source-port behavior changes**: any existing constraints tied to user/core-port mapping remain unchanged.
4. **No keyboard-only regressions allowed**: keyboard-only hotkey users must pass current behavior checks with the new setting ON and OFF.
5. **Regression observability required**: verbose logs must emit enough context to compare keyboard decision paths between ON/OFF runs.

## 7) Implementation plan

### Phase 1 (MVP)
- Add config bool + menu item + labels/sub-labels.
- Implement shared hotkey storage and runtime path for **button-based hotkeys first**.
- Keep default OFF for compatibility.
- **Performance placement requirement:** integrate the new branch at the earliest shared hotkey evaluation point (single check per frame/port), and avoid adding per-bind conditional work in inner loops. The goal is to minimize (or effectively eliminate) measurable input-latency impact versus OFF mode.

### Phase 2
- Harden/extend axis behavior and edge cases; do not ship/PR until this hardening completes.
- Add migration utility: "Adopt current hotkeys as shared profile".
- Add diagnostics overlay/logging for resolved hotkey source and mapping.

### Phase 3
- Edge-case hardening for axis-only controllers and hotplug/reconnect on Android.
- Expand tests across multiple input drivers and controller families.

### Semantic versioning milestones (feature track)
- **v1.0.0-dev**: Button-first implementation + regression matrix in progress (internal development only).
- **v1.1.0-rc**: Axis hardening and reconnect edge-case closure.
- **v1.1.0**: First public-ready release candidate for the feature (after pass of full matrix + latency checks).

### File-by-file implementation map (what must change)

1. `config.def.h`
   - Add `#define DEFAULT_INPUT_HOTKEYS_USE_RETROPAD_BINDS false`.

2. `configuration.h`
   - Add `bool input_hotkeys_use_retropad_binds;` to settings bools.

3. `configuration.c`
   - Register the new config key with `SETTING_BOOL("input_hotkeys_use_retropad_binds", ...)`.

4. `msg_hash.h`
   - Add enum labels for the new menu setting/value/sublabel identifiers.

5. `msg_hash_lbl_str.h`
   - Add string key define for `input_hotkeys_use_retropad_binds`.

6. `intl/msg_hash_lbl.h`
   - Add label mapping entries so menu can resolve the new setting label symbol.

7. `intl/msg_hash_us.h`
   - Add English label and sublabel text:
     - Label: `Hotkeys Use Controller's RetroPad Binds`
     - Sublabel: `ON: Hotkeys are bound to RetroPad button/axis codes. Recommended if controllers change frequently. OFF (default): Hotkeys use device-specific button/axis codes.`

8. `menu/menu_setting.c`
   - Add the `CONFIG_BOOL(...)` entry in the Input > Hotkeys list.
   - Place near `input_hotkey_device_merge` / `input_hotkey_follows_player1`.

9. `menu/cbs/menu_cbs_sublabel.c`
   - Add/bind sublabel callback macro entry for the new setting.

10. `input/input_driver.c`
    - Add runtime branch for `input_hotkeys_use_retropad_binds` in hotkey evaluation path.
    - Keep keyboard behavior unchanged per safety contract.
    - Add verbose logging hooks for ON/OFF comparison and diagnostics.

11. (Follow-up localization sweep) `intl/msg_hash_*.h`
    - Add translation entries for non-English locales after US text lands.

12. (User docs follow-up) docs.libretro.com input/hotkey guides
    - Document feature semantics, defaults, and recovery/troubleshooting steps.

## 8) Risks and mitigations

1. **Ambiguity across controller layouts**
   - Mitigation: explicit normalization policy + user preview UI.

2. **Keyboard + controller interactions**
   - Mitigation: preserve existing `input_hotkey_device_merge` semantics; scope new mode to joypad hotkeys first.

3. **Regression risk in legacy configs**
   - Mitigation: default remains OFF; feature gated.

4. **Maintainer concern about complexity**
   - Mitigation: keep one generalized mode, avoid per-hotkey exception toggles.

## 9) Success criteria

- Users can swap between at least two different controllers without rebinding/copying hotkey config files.
- Hotkey behavior remains stable after disconnect/reconnect in supported scenarios.
- Editing hotkeys in UI updates shared behavior immediately.
- No regression in default device-specific mode.
- ON mode shows no user-perceptible latency regression, and any CPU overhead in the input path is below measurement noise in frame-time profiling.

## 10) Open questions before implementation

1. ~~Should `input_hotkeys_use_retropad_binds` include keyboard hotkeys or only joypad hotkeys initially?~~
   - **Decision (2026-04-15):** joypad-only in v1.
2. ~~Should shared mapping target logical RetroPad IDs, or a smaller abstract subset first?~~
   - **Decision (2026-04-23):** start with button-first subset, then harden/extend axis behavior before ship/PR.
3. What is expected behavior when hotkey source port has no controller connected?
4. Do we expose per-core overrides in shared mode, or keep global-only for MVP?

## References (external)

- https://forums.libretro.com/t/hotkeys-are-not-remapped-when-using-a-different-controller-more/43271
- https://www.reddit.com/r/RetroArch/comments/1l757g0/android_how_to_sort_out_hotkeys_for_external/
- https://www.reddit.com/r/RetroArch/comments/1h8pd3g/hotkeys_on_bluetooth_controllers/
- https://forums.libretro.com/t/dual-sense-hotkeys-not-working/44259
- https://github.com/libretro/RetroArch/issues/3414
- https://github.com/libretro/RetroArch/pull/18353
- https://github.com/libretro/RetroArch/pull/18785
- https://github.com/libretro/RetroArch/pull/18670

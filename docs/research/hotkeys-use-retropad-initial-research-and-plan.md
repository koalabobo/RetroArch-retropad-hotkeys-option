# Initial research and plan: Hotkeys Use RetroPad Binds

_Date: 2026-04-15 (moved into docs/research; original file preserved)_

> NOTE: This file is a verbatim relocation of the original research & planning doc. The research header has been renamed to "Initial research and plan" to indicate its role as the project evidence record. The canonical design/spec for implementation lives at docs/design/hotkeys/hotkeys-use-retropad-spec-v1.0.0.md.


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

(omitted here for brevity; full references included in spec)


## 4) Prior upstream product/PR context

- **Accepted precedent**: PR #18353 (“Hotkeys follow player 1”), merged Nov 2025.
- **Design caution**: PR #18785 (menu toggle bypass hotkey enable) was closed Apr 2026.

Implication: a **generalized architecture** is more likely to be accepted than per-hotkey ad hoc exceptions.


## 5) Proposed product direction

Add a new UI boolean option (config name preserved for compatibility):

### New setting: `input_hotkeys_use_retropad_binds` (bool, default `false`)
- **Menu label**: `Hotkeys Use Controller's RetroPad Binds`
- **Sublabel**: `ON: Hotkeys are bound to RetroPad button/axis codes. Recommended if controllers change frequently. OFF (default): Hotkeys use device-specific button/axis codes.`

### User-facing behavior when ON
- User binds hotkeys once in UI.
- Runtime resolves those hotkeys against active controller RetroPad mapping on selected hotkey port.
- Changing hotkeys does not require editing/copying autoconfig files.
- Switching controllers (same user/port) preserves expected hotkey intent.


## 6) Notes & next steps

This relocated research file has been moved (unchanged except for the top heading) into docs/research/ using the token `hotkeys-use-retropad` for discoverability and consistency across repo artifacts. The design and actionable implementation plan is authored as a separate spec in docs/design/hotkeys/hotkeys-use-retropad-spec-v1.0.0.md. Please refer to that spec for detailed algorithms, data model, and test requirements.

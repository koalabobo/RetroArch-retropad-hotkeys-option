# Steelman Plan: Cross-Controller Hotkeys via RetroPad Binds

_Date: 2026-04-15_

> **Role of this document:** This is a **challenge/hardening companion** to the official plan.  
> The canonical source of truth for product decisions, requirements, and accepted scope is `docs_hotkey_cross_controller_research.md`.  
> Any newly accepted decisions must be logged in the official plan first.

## Actionable Recommendations (from adversarial review)

1. **Ship as an opt-in boolean, default OFF** (`input_hotkeys_use_retropad_binds = false`) to preserve backwards compatibility and reduce rollback risk.
2. **Scope v1 to joypad hotkeys only**; do not alter keyboard hotkey semantics in v1.
3. **Keep existing settings authoritative** (`input_hotkey_follows_player1`, `input_hotkey_device_merge`) and layer the new option beneath them.
4. **No per-hotkey bypass flags** in v1. Keep this as one generalized switch to avoid settings sprawl.
5. **Instrument resolution path** with debug logging (source port, joy_idx, profile name, resolved RetroPad key) behind verbose logging.
6. **Add conflict detection UX** for unsupported/unresolvable binds (axis-only pads, missing logical mappings).
7. **Build a controller matrix test plan** before merging behavior changes (Android handheld + docked BT pad, xinput + dinput, SDL/udev).
8. **Preserve manual override escape hatch**: users can always turn option OFF and fall back to existing device-specific behavior.
9. **Explicitly define precedence rules** between autoconfig hotkeys, retroarch.cfg hotkeys, and shared RetroPad hotkey evaluation.
10. **Document failure modes** and include a quick recovery checklist in user docs (toggle OFF, reset bind, verify profile, restart).

---

## 1) Steelman hypothesis

A single boolean option, **Hotkeys Use Controller's RetroPad Binds**, can provide a practical cross-controller hotkey experience while keeping current behavior unchanged by default.

When ON, joypad hotkeys are evaluated against logical RetroPad mappings for the active hotkey source port, rather than raw per-device button/axis codes. This should allow users to dock/undock or swap controller models without manually copying hotkey lines into each autoconfig profile.

## 2) Why this is likely to succeed

- It aligns with existing RetroArch architecture that already normalizes controllers to RetroPad via autoconfig.
- It aligns with accepted upstream direction around hotkey ownership (`Hotkeys Follow Player 1`, PR #18353).
- It avoids the maintainability trap maintainers warned about (per-hotkey bypass proliferation, see PR #18785 discussion).
- It addresses repeated user friction where autoconfig and hotkey persistence conflict with controller switching.

## 3) Product definition (v1)

### 3.1 New setting
- Config key: `input_hotkeys_use_retropad_binds`
- Type: bool
- Default: `false`
- Label: `Hotkeys Use Controller's RetroPad Binds`
- Sublabel: `ON: Hotkeys are bound to RetroPad button/axis codes. Recommended if controllers change frequently. OFF (default): Hotkeys use device-specific button/axis codes.`

### 3.2 Behavioral contract
When OFF:
- Existing behavior unchanged.

When ON:
- For joypad-origin hotkeys, evaluate based on logical RetroPad mapping of the hotkey source user/port.
- Preserve existing `Hotkeys Follow Player 1` determination of hotkey source.
- Preserve existing `Hotkey Device Type Merge` block/gating semantics.
- Keyboard hotkeys continue to behave exactly as today (v1 non-goal).

### 3.3 Non-goals (v1)
- No per-hotkey bypass settings.
- No migration wizard UI.
- No automatic rewrite of autoconfig files.

## 4) Technical architecture (v1)

### 4.1 Inputs and sources
- Determine hotkey source port using current logic (`input_hotkey_follows_player1` if enabled).
- Retrieve active joypad device index (`joy_idx`) and associated RetroPad bind map.

### 4.2 Resolution
- Translate configured hotkey intents into logical RetroPad checks (instead of raw button number equality by specific controller profile).
- Evaluate hotkey activation against resolved logical mapping for active device.

### 4.3 Precedence model
1. Hotkey source port selection (existing logic).
2. Device type gating/block behavior (existing merge/enabler logic).
3. If new boolean OFF: current hotkey evaluation path.
4. If new boolean ON: RetroPad-resolved joypad hotkey evaluation path.

### 4.4 Observability
- Add verbose logging for: source port, mapped user, joy_idx, config mode, resolved logical bind, and final action trigger.

### 4.5 Keyboard safety contract (v1)

_Note: This contract is mirrored in the official plan doc and governed there._

When `input_hotkeys_use_retropad_binds = true`, keyboard behavior must remain equivalent to baseline:

1. **No keyboard remapping changes**: keyboard hotkey keycodes are read/evaluated exactly as in OFF mode.
2. **No keyboard precedence changes**: existing keyboard-vs-joypad block/unblock semantics remain unchanged.
3. **No keyboard source-port behavior changes**: any existing constraints tied to user/core-port mapping remain unchanged.
4. **No keyboard-only regressions allowed**: keyboard-only hotkey users must pass current behavior checks with the new setting ON and OFF.
5. **Regression observability required**: verbose logs must emit enough context to compare keyboard decision paths between ON/OFF runs.

## 5) Adversarial review: likely failure modes and gaps

### Gap A — “RetroPad-normalized” is underspecified
**Risk:** ambiguous implementation could accidentally mirror current behavior.
**Mitigation:** define exact mapping algorithm and data structures before coding.

### Gap B — Axis/button heterogeneity across controllers
**Risk:** some controllers expose triggers/special keys as axes; logical match may fail.
**Mitigation:** include axis normalization rules and fallback diagnostics.

### Gap C — Autoconfig precedence confusion
**Risk:** users may still see profile overrides and assume feature is broken.
**Mitigation:** publish explicit precedence rules and troubleshooting in UI/help.

### Gap D — Keyboard regressions via shared path
**Risk:** unintentional keyboard behavior changes.
**Mitigation:** hard-scope v1 to joypad hotkeys only; add targeted tests.

### Gap E — Port churn and reconnect instability
**Risk:** hotplug/reconnect changes effective source user unexpectedly.
**Mitigation:** test against reconnect scenarios and document expected behavior with `Hotkeys Follow Player 1`.

### Gap F — Settings interaction complexity
**Risk:** emergent bugs when combined with `input_hotkey_device_merge`, game focus, and menu controls.
**Mitigation:** define interaction table and add regression tests for each combination.

### Gap G — Debuggability
**Risk:** users cannot tell why a hotkey did/did not fire.
**Mitigation:** add verbose logs and optional on-screen debug notifications.

### Gap H — Maintainership concerns about scope creep
**Risk:** follow-up demands for per-hotkey exceptions.
**Mitigation:** keep one generalized boolean and reject per-hotkey bypasses for v1.

## 6) Internet evidence summary (problem and constraints)

### 6.1 Official docs point users toward autoconfig editing workflows
- Libretro controller autoconfig docs explicitly describe copying hotkey lines from `retroarch.cfg` into per-controller autoconfig files and validating afterward.
- This validates that manual profile edits are currently an endorsed workaround for some cases.

### 6.2 Official docs emphasize profile-per-controller reality
- Input docs recommend saving controller profiles for each different controller.
- This supports the pain pattern when users swap controllers frequently.

### 6.3 Upstream history supports generalized approach
- PR #18353 (merged) legitimizes changing hotkey ownership behavior around player/core port mapping.
- PR #18785 discussion explicitly warns that adding bypasses for individual hotkeys is not sustainable.

### 6.4 User reports repeatedly surface conflicts between config layers
- Community reports describe hotkey changes reverting, autoconfig “Auto” hotkeys overriding UI edits, and different controller models needing separate profile edits.

## 7) Delivery plan

### Phase 0: Design freeze (short)
- Lock behavioral contract, precedence, and non-goals.
- Produce settings interaction table.

### Phase 1: Plumbing PR
- Add boolean setting, menu label/sublabel, config serialization.
- Add no-op/guarded branch and logging scaffolding (feature OFF by default).

### Phase 2: Behavior PR
- Implement RetroPad-resolved joypad hotkey evaluation path.
- Add regression tests for OFF mode equivalence.

### Phase 3: Hardening PR
- Controller matrix verification.
- Edge-case fixes (axis-only and reconnect behavior).
- User-facing troubleshooting doc.

## 8) Test matrix (minimum)

- Platforms: Android, Linux (udev/sdl2), Windows (xinput/dinput), macOS/iOS if available.
- Scenarios:
  1. Built-in handheld controls only.
  2. External controller only.
  3. Switch between built-in and external during same session.
  4. Two different external controllers with different physical layouts.
  5. Reconnect/hotplug and device-order changes.
  6. `Hotkeys Follow Player 1` ON/OFF combinations.
  7. `Hotkey Device Type Merge` ON/OFF combinations.

## 9) Exit criteria for merge

- OFF mode behavior is unchanged in all tested scenarios.
- ON mode demonstrates cross-controller hotkey continuity for supported controllers.
- No keyboard hotkey regressions in v1 scope.
- Clear logs/troubleshooting available for failed resolution cases.

## 10) Source links used for this steelman/adversarial review

- https://docs.libretro.com/guides/controller-autoconfiguration/
- https://docs.libretro.com/guides/input-and-controls/
- https://github.com/libretro/retroarch-joypad-autoconfig
- https://github.com/libretro/RetroArch/pull/18353
- https://github.com/libretro/RetroArch/pull/18785
- https://github.com/libretro/RetroArch/issues/16552
- https://forums.libretro.com/t/hotkeys-are-not-remapped-when-using-a-different-controller-more/43271
- https://www.reddit.com/r/RetroArch/comments/19451g5/different_hotkey_with_different_controllers/
- https://www.reddit.com/r/RetroArch/comments/1646uuo/hotkey_mapping_retropad_vs_actual_gamepad/
- https://www.reddit.com/r/RetroArch/comments/1qmmxpt/cannot_disable_menu_toggle_on_controller_it_keeps/

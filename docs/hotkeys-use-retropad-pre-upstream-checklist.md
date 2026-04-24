# Pre-Upstream Checklist: Hotkeys Use RetroPad

**NO PRs to libretro/RetroArch until every item below is complete and owner (koalabobo) gives explicit approval.**

## 1. Testing & Safety

- [ ] **Keyboard Safety Pass**  
  Keyboard hotkey unit tests must run with `input_hotkeys_use_retropad_binds` both OFF and ON—results must be identical. Known configs from forums/reddit must be covered.
- [ ] **Integration Tests Pass**  
  Controller swap, hotplug, reconnect (esp. Android) regressions are ruled out. Hotkey behavior confirmed on all primary platforms.
- [ ] **Performance Microbenchmark**  
  Measured overhead for hotkeys-use-retropad mode must be negligible (<50 µs target). Compare to baseline.
- [ ] **Axis Handling (Phase 2) Complete**  
  No PR to libretro unless axis support and edge cases are done and regression tested.

## 2. Migration, Rollback, and Documentation
- [ ] **Migration Tool Functional**  
  "Adopt Current Hotkeys as Hotkeys Use RetroPad" works and warns on unmappable binds; rollback guarantees preserved.
- [ ] **Documentation Verified**  
  Spec, decision log, user README, diagnostics docs are updated and accurate.
- [ ] **Diagnostics Overlay Present**  
  Mode status, mapping, and cache must be visible from UI for troubleshooting advanced use.

## 3. PR/Review/Process
- [ ] **Acceptance Gates Satisfied**  
  Above checks are verified, PR is clearly scoped, test evidence included.
- [ ] **Collaborator Approval**  
  Another reviewer signs off in your fork PR.
- [ ] **No Upstream PR Opened**  
  NEVER until explicit approval from koalabobo.

## 4. (Optional) Upstream Maintainer Outreach
- [ ] **Libretro RFC/Issue/Forum Thread Filed**  
  (Optional, but recommended): Test approach with active maintainers before PR is ever opened.

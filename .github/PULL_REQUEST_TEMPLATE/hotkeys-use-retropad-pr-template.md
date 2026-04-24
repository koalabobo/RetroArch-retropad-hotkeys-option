---
name: Hotkeys Use RetroPad Feature PR
about: Complete this checklist BEFORE proposing or merging any PR targeting upstream libretro/RetroArch
labels: [feature, input, hotkeys-use-retropad]
---

# Hotkeys Use RetroPad Pre-Upstream Checklist
<!-- Copy of docs/hotkeys-use-retropad-pre-upstream-checklist.md for PRs and contributors. -->

- [ ] **Keyboard safety unit/regression tests pass.**
- [ ] **Controller swap/hotplug integration tests pass.**
- [ ] **Performance (microbenchmarks) documented and within targets (<50µs/frame).**
- [ ] **Axis/edge-case support (Phase 2) complete (DO NOT open PR with partial axis support).**
- [ ] **Migration/adoption tool tested and working.**
- [ ] **Roll-back tested (toggling mode off returns to per-device config, no data loss).**
- [ ] **Spec & research docs updated and linked in PR body.**
- [ ] **Diagnostics overlay available and screenshots attached if practical.**
- [ ] **Test evidence attached: before/after logs, config examples, screen or vid of pass cases.**
- [ ] **No PR (draft or otherwise) to upstream unless/explicit approval from koalabobo.**

Consult the canonical design spec in docs/design/hotkeys/hotkeys-use-retropad-spec-v1.0.0.md and initial research at docs/research/hotkeys-use-retropad-initial-research-and-plan.md for details.
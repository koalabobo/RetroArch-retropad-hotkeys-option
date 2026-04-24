# Appendix: Pre-Upstream Checklist (Hotkeys Use RetroPad)

For any PR intended for libretro/RetroArch, ALL these conditions MUST be satisfied first (see docs/hotkeys-use-retropad-pre-upstream-checklist.md for the canonical list).

- Keyboard regression safety: all keyboard hotkey behaviors identical with the setting ON and OFF.
- Integration/perf tests: hotplug, config swap, latency measures meet target.
- Axis/edge-case support: complete and tested.
- Diagnostics overlay & user guidance present.
- Migration/rollback safety: all config data recoverable and old behavior restored when OFF.
- No upstream PR submission (even draft) until explicit approval from koalabobo.
- Attach test evidence to PR as practical.

See top-level checklist in docs/hotkeys-use-retropad-pre-upstream-checklist.md.
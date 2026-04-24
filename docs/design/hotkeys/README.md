# Hotkeys Use RetroPad — Design Index

This directory contains design artifacts for the Hotkeys Use RetroPad feature (token: `hotkeys-use-retropad`).

Files:

- `hotkeys-use-retropad-spec-v1.0.0.md` — The canonical design/spec for v1 (button-only). Contains data model, runtime pseudocode, caching/invalidation rules, keyboard safety contract and unit-test requirement, migration utility, diagnostics, test matrix, perf targets, and PR checklist.

- ../research/hotkeys-use-retropad-initial-research-and-plan.md — Original research and planning material (relocated; titled "Initial research and plan"). This file preserves community evidence, issue/PR references, and the decision log.

How to use

- Review `hotkeys-use-retropad-spec-v1.0.0.md` for implementation guidance and the PR acceptance gates.
- Do not change UI text: the menu label remains exactly `Hotkeys Use Controller's RetroPad Binds` and the config key remains `input_hotkeys_use_retropad_binds`.
- Implementation branch: `feat/hotkeys-use-retropad-v1` (docs live there). Code implementation should reference this spec and the token `hotkeys-use-retropad` in commit messages and new filenames for discoverability.

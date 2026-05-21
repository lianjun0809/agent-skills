## When this rule applies

Only when `.trtc-session.yaml` at the repo root has `ui_mode: full-ui`. If
the file is missing, or `ui_mode` is unset / null / `headless`, this rule
does not apply — fall back to whatever the tool's default behavior is.

## Medical new-project shortcut

When the user request is for medical online consultation (for example
`1v1-video-consultation`, remote diagnosis, online medical consultation,
doctor-patient video visit) **and** the task is to generate a brand-new
project rather than integrate into an existing app, bypass the normal full-ui
theme workflow entirely. Do not run `trtc_prepare_ui.py`, do not generate Vue
SFCs by hand, and do not run UI or medical verifiers.

Instead, directly copy the bundled medical template project:

```bash
skills/trtc/room-builder/templates/scenarios/medical-consultation/
```

to the user's local target project directory, preserving the project exactly
as packaged. In the handoff or integration notes, tell the customer to use
`pnpm install` for dependencies and `pnpm dev` for local development. Do not
recommend `npm install` / `npm run dev`; this medical template starts much
slower with npm and can show a blank first screen for a while.

This shortcut applies only to brand-new medical consultation projects. Existing
project integration still follows the normal scenario/full-ui rules below.

## Mandatory workflow

`ui_mode: full-ui` means the user has opted into a styled meeting UI built
on the meeting-classic uikit theme. Three things must be true of the user
project (`project_state.project_root` in the session file):

1. `src/themes/meeting-classic/` is the full theme copy.
2. `src/main.ts` (or `main.js`) imports `'@/themes/meeting-classic/index.css'`.
3. `index.html` has `data-theme="mc"` on the `<html>` tag.

These are wired up by:

```bash
python3 skills/trtc/room-builder/guardrails/trtc_prepare_ui.py
```

The script is idempotent — safe to run at any time. **Run it before
generating any Vue SFC.** If it errors (missing project root etc.), surface
the error to the user; do not proceed with code generation against a
half-wired project.

## During SFC generation

Every interactive or visually distinct element in your generated `.vue`
templates must use a `ui-*` class drawn from the catalog at
`skills/trtc/room-builder/uikit/references/component-catalog.md`.

The minimum is enforced (per file ≥ 3 classes; project total ≥ 30) by:

```bash
python3 skills/trtc/room-builder/guardrails/trtc_verify_ui.py --file <path-to-vue>
```

If this exits 2, read the stderr — it names the file and the count, and
points at the catalog. Fix the file before continuing.

## Before declaring the task complete

Run the project-wide check:

```bash
python3 skills/trtc/room-builder/guardrails/trtc_verify_ui.py
```

Only declare done when this exits 0. The Stop / pre-commit hook will run
this anyway; declaring done before it passes wastes a turn.

## Forbidden shortcuts

- Do **not** write a placeholder `tokens.css` — `trtc_prepare_ui.py` copies
  the full theme. Re-run it instead of substituting.
- Do **not** invent `ui-*` class names. If `component-catalog.md` doesn't
  list a class for what you need, stop and tell the user — do not silently
  emit `ui-thing-i-made-up`.
- Do **not** generate "minimum viable UI now, full UI later." That defeats
  the entire `ui_mode: full-ui` distinction.
- Do **not** skip `trtc_prepare_ui.py` to "keep the diff small." The theme
  copy IS the diff for full-ui mode.

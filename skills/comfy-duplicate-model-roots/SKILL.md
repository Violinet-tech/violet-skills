---
name: comfy-duplicate-model-roots
description: >
  Diagnoses and fixes duplicate/empty/conflicting model-root configs in
  ComfyUI launchers and package managers (ComfyUI Desktop confirmed; Stability
  Matrix and others suspected to have the same class of bug). Use when a
  ComfyUI install is slow to start, when a model/LoRA browser or loader shows
  far fewer files than actually exist or shows files with no preview, when
  models were moved to a new drive and some of them "disappeared", or when
  the same install behaves differently depending on how it was launched.
  Trigger phrases: "models not showing up", "comfy is slow to open", "missing
  previews", "wrong model folder", "duplicate model paths", "stability
  matrix models", "which folder is comfy actually using".
---

# Duplicate/conflicting model-root configs (ComfyUI-family apps)

Several ComfyUI launchers and package managers layer their OWN model-root
list on top of plain ComfyUI's `extra_model_paths.yaml`, and that extra layer
is where this bug actually lives. Confirmed on ComfyUI Desktop 2026-09-29;
almost certainly the same class of problem on Stability Matrix (not yet
verified — see the placeholder section below) and any other manager that
lets several installs/packages "share" one model library.

**The core pattern, regardless of which app**: there is more than one place
a model-root path can be configured, they don't always agree, and nothing
warns you when they don't. Symptoms: the app takes longer to start than it
used to (scanning N roots, some possibly on a slow or since-unplugged
drive), a model/LoRA browser shows a handful of files instead of the real
library, or shows files with no preview image, or "the same install acts
different depending on how you start it" (when in fact it's not the same
config being read both ways).

## Diagnose (works for any manager)

1. Find every place the manager stores a model-root path: its own
   settings/config file (JSON/YAML/TOML), any *generated* derived config it
   writes for the underlying ComfyUI process to read, and the plain
   `extra_model_paths.yaml` inside whichever ComfyUI checkout it's driving.
   A generated file usually says so at the top ("do not edit manually") —
   believe it, and go find what generates it instead of hand-editing it.
2. List every root path across all of those, then **count real files per
   root** — never trust "the folder exists" or "the folder has
   subfolders" as evidence of anything:
   ```bash
   find "<root>/models" -type f \( -iname "*.safetensors" -o -iname "*.gguf" -o -iname "*.ckpt" -o -iname "*.pt" -o -iname "*.bin" \) | wc -l
   ```
3. A root with 0 files is pure overhead: still gets scanned/watched, adds
   nothing. A root with real files stays, even if it isn't the "main"
   library — these apps union every registered root per model kind
   (checkpoints, loras, vae, clip/text_encoders, diffusion_models/unet,
   gguf, clip_vision, embeddings, controlnet, upscale_models, etc.), they
   don't pick one and ignore the rest.

## Fix (works for any manager)

1. Back up the config file you're about to edit first (it's always small).
2. If more than one root has real files, ask the user which is their real
   library before consolidating — a second root with a handful of files may
   be intentional (e.g. kept on a faster local drive on purpose), not a
   mistake to merge away.
3. Edit the *config* to drop empty/duplicate entries and add whatever path
   the user actually wants loaded. Never delete files or folders to "fix"
   this — it's a config problem, not a files problem.
4. The app usually needs a full quit and reopen to pick up a hand-edited
   config file, since it may hold its own copy in memory and overwrite your
   edit on exit before ever re-reading it. If computer-use isn't available
   for that app, say so and ask the person to quit/reopen it themselves
   rather than trying to automate the click.
5. Verify against the ORIGINAL file counts from step 2 of diagnosis, not
   just "something showed up now" — confirm the number of files the browser
   reports actually matches what's really on disk across every kept root.

## Worked example: ComfyUI Desktop (verified 2026-09-29)

Three separate places, only the first is meant to be hand-edited:
- `%APPDATA%\Comfy Desktop\settings.json` -> `modelsDirs` (array). The real,
  user-editable source of truth for Desktop's *extra* shared roots.
- `%APPDATA%\Comfy Desktop\shared_model_paths.yaml` and
  `%APPDATA%\Comfy Desktop\instance-model-paths\inst-*.yaml` — GENERATED
  from `modelsDirs` (+ the install's own `extra_model_paths.yaml`). Literally
  headed "do not edit manually" — true, edit `settings.json` instead.
- The install's own `extra_model_paths.yaml` (inside `.../ComfyUI/`, next to
  `main.py`) — a third, independent source, additive to `modelsDirs`.

Found on one machine: `modelsDirs` listed two D: drive paths, one of them
(`ComfyUI-Shared\models`) completely empty (0 files) after a past drive
reorganization, while the real 99-file library lived on F: via
`extra_model_paths.yaml` the whole time. Removing the empty entry from
`modelsDirs` fixed both the slow startup and the model browser's low count,
without touching a single file.

## Stability Matrix — not yet verified, do this next

Same class of bug is suspected (multiple packages sharing one model library
is Stability Matrix's whole premise, which is exactly the shape that
produces this bug elsewhere), but its actual config file names/locations
have not been checked yet. Before assuming anything about Stability
Matrix's format, apply the generic "Diagnose" steps above to it directly:
find its own settings store, find whatever it generates for each package
(Package Config in Stability Matrix terms) to consume, and count real files
per root the same way. Update this section with the verified specifics once
that's done — do not guess at exact file paths here ahead of checking.

## Related
See the `comfy-mcp-fixes` skill for other ComfyUI Desktop / comfy-mcp specifics.
This skill is scoped to the model-root-duplication bug class, so it applies the
same way to launchers that skill does not cover.

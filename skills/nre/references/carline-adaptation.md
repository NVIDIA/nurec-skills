# Carline Adaptation

Carline adaptation is a local NRE rendering utility for an existing NuRec
USDZ and an already-augmented target vehicle rig JSON. It composes existing
NRE and Harmonizer workflows; it does not introduce another runtime or
command wrapper.

## Workflow

1. Resolve the source USDZ, augmented target rig JSON, output directory, and
   target-rig camera IDs.
2. Follow [Option 2 in `local-render.md`](local-render.md#option-2-custom-rig-trajectory-full-flow):
   - Pass the augmented rig to `export-custom-rig-trajectory` with
     `--rig-json`.
   - Pass the exported JSON to `render` with `--custom-rig-trajectory` and
     render the requested `--camera-id` values.
3. Inspect and preserve the raw rendered frames.
4. Hand the raw frame directories to the
   [`nurec-fixer`](../../nurec-fixer/SKILL.md) skill and run the public
   DiffusionHarmonizer for each rendered camera. That skill owns the setup and
   inference commands. Stop after step 3 only when the user explicitly wants
   raw renders without harmonization.

## Inputs and outputs

- **Inputs:** source USDZ, augmented target rig JSON, requested target-rig
  camera IDs.
- **NRE outputs:** custom rig trajectory JSON and raw rendered frames.
- **Harmonizer output:** enhanced frames produced from the raw render
  directories by `nurec-fixer` while preserving the raw frames.

Example original and augmented rig files remain under
[`rig-json/`](rig-json/). Use the user's augmented rig when one is supplied.

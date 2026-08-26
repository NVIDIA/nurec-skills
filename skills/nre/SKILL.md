---
name: nre
description: >-
  Use to drive NVIDIA Omniverse NuRec / Neural Reconstruction
  Engine (NRE) via the public NGC containers nvcr.io/nvidia/nre/nre
  and nvcr.io/nvidia/nre/nre-tools (NGC_API_KEY required) — train
  3DGUT Gaussian reconstructions from NCore clips, generate aux
  data, adapt an existing USDZ to an augmented target-vehicle rig
  (carline adaptation), render frames or LiDAR sweeps (local or warm
  `serve-grpc`), export PLY/depth/mesh/USDZ, edit actors, and evaluate
  metrics. Do NOT use for per-object asset capture (use
  `asset-harvester`) or sensor-to-NCore conversion (use `ncore`).
version: "0.2.2"
tools:
  - Shell
  - Read
  - Write
license: CC-BY-4.0 AND Apache-2.0
compatibility: >-
  Linux x86_64, 1+ NVIDIA GPU (Ampere A100/A10/A40/RTX A6000, Ada
  L20/L40/L40S, Hopper H100/H20, or Blackwell RTX Pro 6000D) with
  CUDA 12.8 and >= 24 GB VRAM (48+ GB recommended); driver R570+
  recommended (R580+ on Blackwell, R535+ minimum for Fixer-only).
  Docker >= 23.0.1 + NVIDIA Container Toolkit >= 1.13.5, NGC
  account with NGC_API_KEY exported.
dependencies:
  - bash
  - docker
  - python3
metadata:
  author: NVIDIA NRS <nurec-skills@nvidia.com>
  tags:
    - nurec
    - autonomous-vehicles
    - neural-reconstruction
    - rendering
    - container
  product_page: https://www.nvidia.com/en-us/omniverse/nurec/
  ngc_container: nvcr.io/nvidia/nre/nre:latest
  ngc_tools_container: nvcr.io/nvidia/nre/nre-tools:latest
  ngc_fixer_model: https://catalog.ngc.nvidia.com/orgs/nvidia/teams/nre/models/nurec-fixer
  hf_datasets: https://huggingface.co/datasets/nvidia/PhysicalAI-Autonomous-Vehicles-NuRec
  hf_fixer_model: https://huggingface.co/nvidia/Difix3D
  release_date: "2026-04-30"
  time-estimate: "2h"
---

# NRE — NVIDIA Omniverse NuRec (Neural Reconstruction Engine)

## Purpose

Drive the public NVIDIA Omniverse NuRec / Neural Reconstruction
Engine containers (`nvcr.io/nvidia/nre/nre`,
`nvcr.io/nvidia/nre/nre-tools`) to train a 3DGUT/3DGRT Gaussian
reconstruction from an NCore V4 camera+LiDAR clip, render novel
views (locally or via gRPC), generate aux data, export
PLY/depth/mesh/ego-mask/tracks, package Asset Harvester output
into a USDZ, and evaluate rendering metrics.

This skill carries the host-side toolkit around the NRE CLI: NGC
credential resolution, cached-image notes, local render recipes,
MP4 encoding, warm `serve-grpc` boot/teardown scripts, a thin
Python gRPC client for repeated RGB renders, bundled rig JSONs,
pre-baked custom-rig trajectories, and bash / Hydra / OSMO
workflow templates.

## When to Use / When NOT to Use

**Use this skill when** the user has an NCore V4 clip (or a USDZ +
NRE artifact pair) on a Linux x86_64 host with an NVIDIA GPU and
an NGC API key, and wants to train, render, generate aux data,
export artifacts, insert/remove actors, run the gRPC server, or
evaluate metrics. Concrete triggers:

- Train a multi-camera + LiDAR AV clip into a renderable USDZ
  scene with 3DGUT (or 3DGRT ray-traced) Gaussians.
- Generate NuRec auxiliary data (seg, depth, ego mask, DINOv2,
  LiDAR-seg visibility) using `nre-tools`.
- Render frames locally (no server) along the training rig or a
  custom rig + offsets.
- Render novel views via the sensorsim gRPC API (CARLA, Isaac
  Sim, AlpaSim, custom simulator), optionally with Difix
  artifact-removal.
- Render LiDAR sweeps via `render-grpc --lidar`.
- Export PLY / ego masks / depth / Poisson mesh / ground mesh /
  point clouds / cuboid tracks / NCore tracks / custom rig
  trajectories.
- Insert / remove / replace 3D actors with `export-external-assets`
  + `render-grpc --edit-assets`.
- Render the gated HF dataset
  `nvidia/PhysicalAI-Autonomous-Vehicles-NuRec`.
- Upgrade an old USDZ once (`upgrade-artifact`).
- Inspect / evaluate (`export-parsed-config`, `gaussian-statistics`,
  `eval-rendering-metrics`, `compute-metrics`,
  `eval-ground-mesh`).
- Browse a USDZ or PLY in the in-container viewer.

**Do NOT use this skill when:**

- The user still needs to convert raw sensor data into NCore V4
  (use the `ncore` skill first; NRE consumes NCore-formatted
  shards).
- The user wants per-object 3D asset extraction from sparse views
  (use `asset-harvester`; NRE only *consumes* AH outputs via
  `export-external-assets`).
- The user only needs to clean up already-rendered frames using
  the standalone Cosmos-based Fixer (use `nurec-fixer`). NRE's
  inline `--enable-difix` flag is still on this skill's surface,
  but the standalone harmonizer pipeline is owned by
  `nurec-fixer`.

## Inputs

- **dataset_dir** — host directory holding the NCore shards
  (`<NAME>.zarr.itar`, `<NAME>.json`, and any pre-generated
  `<NAME>.aux.*.zarr` auxiliary shards). Required.
- **dataset_name** — basename of the NCore dataset (the part
  before `.zarr.itar`). Required.
- **output_dir** — host directory NRE will fill with checkpoints,
  parsed config, metrics, videos, and USDZ artifacts. Required.
- **camera_ids / lidar_ids** — sensor IDs from the NCore JSON to
  include. Default: all sensors per recipe.
- **config_name** — Hydra config path resolved inside the
  container. Pick by source dataset:
  - **Waymo Open Dataset** →
    `configs/apps/AV/Waymo/3dgut_dynamic.yaml` (and its
    `_mcmc` / `_road_semantic` / `_static` siblings). These are
    Waymo-only — they bake in the Waymo sensor rig and conventions.
  - **NVIDIA Physical AI Autonomous Vehicles (PAI)** →
    `/apps/prod/Hyperion-8.1/car2sim_6cam.yaml` (the Hyperion-8.1
    car2sim 6-camera recipe used by the Maglev PAI pipeline).
    Typically referenced via the small overlay shipped at
    [`references/configs/pai.yaml`](references/configs/pai.yaml),
    which extends `car2sim_6cam.yaml` with PAI's
    `lidar_top_360fov` ID, six-camera validation set, and lidar
    `intensity` supervision; mount it as
    `{nre_config_dir}/external_overrides.yaml` and pass
    `--config-name=external_overrides`.
  - **PandaSet / NVIDIA AV (NV) / Tesla / Alpasim** → see the
    matching `configs/apps/AV/{PandaSet,NV,Tesla}/…` or
    `configs/apps/Alpasim/…` recipes in
    `references/configuration.md`.
  - Do **not** use the Waymo recipes for PAI clips — the
    sensor rig, validation cameras, and lidar IDs differ.
- **mode** — `train`, `val`, or `trainval`. Default: `trainval`.
- **NGC_API_KEY** — required env var. Generate at
  <https://org.ngc.nvidia.com/setup/api-keys>.

## Instructions

1. **Validate prerequisites.** Have the agent execute
   `scripts/validate_setup.py` via its standard script runner
   (`run_script("scripts/validate_setup.py")`, or
   `python scripts/validate_setup.py [--strict]`). It checks
   Docker, NVIDIA Container Toolkit, GPU/driver, and
   `NGC_API_KEY`. Resolve any FAIL line before pulling the image.
2. **Authenticate Docker to NGC + pull the public containers.**
   See [`references/install.md`](references/install.md). Pull both
   `nvcr.io/nvidia/nre/nre:latest` and
   `nvcr.io/nvidia/nre/nre-tools:latest`.
3. **Confirm input layout.** The dataset directory must contain
   `<NAME>.zarr.itar`, `<NAME>.json`, and any `<NAME>.aux.*.zarr`
   shards. If the NCore data is fresh, generate auxiliary data
   first — see `references/aux-data.md`.
4. **Train / validate the reconstruction.** Run the train recipe
   in [`references/cookbook.md`](references/cookbook.md) with the
   chosen Hydra `--config-name`, `mode`, `dataset.path`, and
   `out_dir`. For multi-GPU append `trainer.world_size=<N>
   trainer.num_nodes=<M>` (see Workflow D). Set
   `checkpoint.artifact.enabled=true` if you intend to render or
   serve the result.
5. **Export downstream artifacts.** Use export sub-commands
   (`export-gaussian-plys`, `export-mesh`, `export-ground-mesh`,
   `export-ego-mask`, `export-depth`, `export-sequence-tracks`,
   `export-ncore-tracks`, …) — full surface in
   `references/cli-reference.md`.
6. **Render novel views — pick the backend.**
   - **Local CLI** — `nre render --artifact-path <usdz>` writes
     frames on disk along the training trajectory, with optional
     rig offsets or `--custom-rig-trajectory`. No gRPC server.
     See `references/local-render.md`.
   - **Warm RGB service** — boot `serve-grpc` once with
     `scripts/session_warm_server.sh`, extract protobuf stubs,
     and use
     `references/NRE_RenderClient/scripts/thin_client.py` for
     repeated single-camera or `batch_render_rgb` calls.
   - **Remote CLI / simulator integration** — `serve-grpc` +
     `render-grpc` (or your own client via `nre.grpc.protos`).
     Required for LiDAR rendering, simulator loops, Difix, or
     `--edit-assets`. See `references/grpc-api.md` and
     `references/physical-ai-render.md`.
7. **Edit actors (optional).** Run `export-external-assets` to
   repackage Asset-Harvester output into a new USDZ, then pass
   the produced `edit-assets.json` to `render-grpc --edit-assets`
   (with `serve-grpc --enable-editing-actors`). See
   `references/asset-editing.md`.
8. **Validate the result.** Confirm
   `<output_dir>/<RUN-ID>/usd-out/last.usdz` opens, `metrics.yaml`
   reports a reasonable `test/psnr`, and the generated MP4s
   render. For more thorough metrics use Workflow I (eval) in
   [`references/workflows.md`](references/workflows.md). Tear
   down any gRPC server (`Ctrl-C` or `docker rm -f`).

For any NRE task expected to run 5 minutes or longer (training,
OSMO jobs, multi-clip renders), follow
`references/long-running-tasks.md`: delegate to a subagent /
background job and report compact status at least every 5
minutes.

## Examples

### Example 1 — End-to-end NCore → USDZ → render

Walk Workflow A in
[`references/workflows.md`](references/workflows.md): validate
host → generate aux data → train (cookbook recipe) → export →
local render or `serve-grpc`. Concrete commands live in the
referenced files; this index does not duplicate them.

### Example 2 — Skip training, render the gated Physical AI dataset

Walk Workflow B: download
`nvidia/PhysicalAI-Autonomous-Vehicles-NuRec` from HuggingFace,
then jump to `serve-grpc` + a Python client. Coordinate-frame
conversion code is in `references/physical-ai-render.md`.

### Example 3 — Insert Asset-Harvester actors into a USDZ

Walk Workflow C: run `asset-harvester`, then
`export-external-assets`, edit `edit-assets.json`, and call
`serve-grpc --enable-editing-actors` + `render-grpc
--edit-assets`. Schema lives in `references/asset-editing.md`.

### Example 4 — Warm-server thin-client for repeated RGB renders

Walk the warm-server quick start at the bottom of
[`references/workflows.md`](references/workflows.md). Boot
`scripts/session_warm_server.sh`, render with
`thin_client.py`, tear down with `scripts/session_teardown.sh`.

## Backend Selection

Pick the smallest backend that exposes the requested feature:

- **Local Docker, single command.** Use `nre render`, `render-grpc`,
  or an export sub-command directly. Simplest for one-off renders,
  LiDAR sweeps, actor edits, rolling shutter, in-container video
  export, or exact `--replicate-training-views` behavior. See
  `references/local-render.md`, `references/nre-image-notes.md`,
  and `references/mp4-encoding.md`.
- **Local Docker, warm `serve-grpc` + thin host client.** Use for
  render-heavy RGB sessions where repeated Docker/Python/CUDA
  cold-start dominates latency, or where multiple cameras should
  be rendered through one `batch_render_rgb` RPC. See
  `references/NRE_RenderClient/README.md` and
  `scripts/session_warm_server.sh` / `scripts/session_teardown.sh`.
- **OSMO / cluster workflows.** Use the templates under
  `references/example-workflows/osmo/` for multi-clip fan-out,
  isolation from the local machine, or training jobs that should
  not run on the user's workstation. Follow
  `references/ngc-and-registry.md` for registry credentials and
  `references/long-running-tasks.md` for polling discipline.

## Output Format

Structured deliverables placed under `${output_dir}/${RUN_ID}/`
by the NRE container (no JSON state file required from the
agent):

- `config/parsed.yaml` — Hydra-resolved training config.
- `checkpoints/last.ckpt` (plus periodic snapshots).
- `val/metrics.yaml` — per-frame PSNR / SSIM / LPIPS under
  `test/*`.
- `val/*.mp4`, `val/<frame>/*.png` — depth, opacity, segmentation,
  RGB visualisations.
- `usd-out/last.usdz` — USDZ containing the trained reconstruction,
  `data_info.json`, `rig_trajectories.json`,
  `sequence_tracks.json`, `parsed_config.yaml`, `checkpoint.ckpt`,
  optional `mesh.ply`, and `map.xodr`. Render with `nre render`,
  `serve-grpc` + `render-grpc`, the in-container `viewer`, or hand
  to a downstream simulator (CARLA, AlpaSim, Isaac Sim).
- `*.ply` / `ego_mask/*` / `depth/*` / `sequence_tracks.json` /
  `ncore_tracks.json` / `mesh.ply` / `ground_mesh.ply` — produced
  by the matching export sub-command.

## Scripts

| Script | Purpose | Usage |
|--------|---------|-------|
| `scripts/validate_setup.py` | Verify Docker, NVIDIA Container Toolkit, GPU/driver R570+ (R535+ minimum), NGC login, and `NGC_API_KEY` env var. No network calls. | `run_script("scripts/validate_setup.py")` or `python scripts/validate_setup.py [--strict]` |
| `scripts/session_warm_server.sh` | Idempotently boot a session-scoped `nre serve-grpc` container for the thin Python client. Discovers a cached 26.04+ renderer image, mounts the USDZ root, waits for readiness. | `NRE_GRPC_USDZ_HOST_DIR=/path/to/usdz/root bash scripts/session_warm_server.sh` |
| `scripts/session_teardown.sh` | Stop and remove the warm `serve-grpc` container and clear its state file without racing the next boot. | `bash scripts/session_teardown.sh` |

## References

Read these on demand; keep `SKILL.md` as the routing layer.

- [`references/install.md`](references/install.md) — `docker
  login nvcr.io`, image pull, full prerequisite matrix, and safe
  secret-handling for `NGC_API_KEY` / `HF_TOKEN`.
- [`references/cookbook.md`](references/cookbook.md) — most-used
  `docker run` invocations: train + validate, re-validate with
  shift, local render at quarter or native res, `serve-grpc` boot,
  LiDAR sweep, in-container `--help`.
- [`references/workflows.md`](references/workflows.md) — workflows
  A – I end-to-end, plus the warm-server thin-client quick start.
- [`references/troubleshooting.md`](references/troubleshooting.md)
  — extended error matrix (`OOM`, `wandb` blocking,
  `--artifact-glob` mismatches, deprecated flags, gRPC LiDAR size,
  etc.).
- [`references/teardown.md`](references/teardown.md) — disk
  cleanup, post-teardown verification, ownership-recovery.
- `references/cli-reference.md` — full sub-command surface of the
  NRE container (training, validation, `render`, `serve-grpc`,
  `render-grpc`, `render-novel-trajectory`, every `export-*`,
  `upgrade-config` / `upgrade-artifact`, `gaussian-statistics`,
  `eval-rendering-metrics`, `compute-metrics`, `viewer`,
  `ply_viewer`, `profile-dataloader`, `run-script`, the
  `nre-tools` aux-data + AH entry points).
- `references/configuration.md` — Hydra recipe map for Waymo / NV
  / PandaSet / Tesla / Alpasim, plus override matrix.
- `references/aux-data.md` — `nre-tools` auxiliary-data CLI.
- `references/local-render.md` — host-side `docker run … render`
  recipes for rig offsets and `export-custom-rig-trajectory`.
- `references/carline-adaptation.md` — augmented target-rig trajectory export,
  local render, and optional `nurec-fixer` Harmonizer handoff.
- `references/NRE_RenderClient/README.md` — warm-server thin
  Python gRPC client.
- `references/grpc-api.md` — sensorsim gRPC server flags + Python
  client cookbook.
- `references/nre-image-notes.md` — cached-image discovery, 26.04+
  vs 26.03 vs pre-26.03 flags.
- `references/ngc-and-registry.md` — NGC API key resolution.
- `references/mp4-encoding.md` — host-side ffmpeg recipe.
- `references/asset-editing.md` — `export-external-assets` +
  `edit-assets.json` schema.
- `references/physical-ai-render.md` — recipe for rendering the
  HuggingFace NuRec dataset.
- `references/example-workflows/` — bash, Hydra, and OSMO
  templates.
- `references/rig-json/` — bundled `rig.json` and
  `augmented_rig.json`.
- `references/custom-rig-trajectories/` — pre-baked
  `export-custom-rig-trajectory` outputs.
- `references/long-running-tasks.md` — background-job + 5-minute
  status reporting convention.
- `references/nurec-skill-catalog.md` — routing table for sibling
  NuRec-stack skills.
- Public product page: <https://www.nvidia.com/en-us/omniverse/nurec/>
- HF dataset: <https://huggingface.co/datasets/nvidia/PhysicalAI-Autonomous-Vehicles-NuRec>
- HF Fixer model: <https://huggingface.co/nvidia/Difix3D>
- NGC Fixer model card: <https://catalog.ngc.nvidia.com/orgs/nvidia/teams/nre/models/nurec-fixer>

## Prerequisites

Linux x86_64 + NVIDIA GPU + Docker 23+ + NVIDIA Container Toolkit
1.13+ + `NGC_API_KEY`. Full matrix (driver minimums per arch,
shm-size, file ownership, GPU-tier guidance) lives in
[`references/install.md`](references/install.md). Always verify
via `scripts/validate_setup.py` before pulling the image.

## Limitations

- **Linux x86_64 only.** aarch64 (e.g. Jetson) is not supported.
- **Internal source not redistributable.** Use only the public
  NGC containers and the public NuRec docs.
- **Multi-GPU defaults are conservative.** Set
  `trainer.world_size` / `trainer.num_nodes` explicitly to scale
  out; SLURM is auto-detected when both are `0`. Quality plateaus
  past ~6 GPUs; per release notes, multi-GPU +
  `dataset.aux_data=false` is a known crash combination.
- **`--config-name` paths differ between train and val/export.**
  Training uses container-bundled recipes; validation and exports
  re-pass the `parsed.yaml` written under
  `<output_dir>/<RUN-ID>/config/`.
- **Validation may prompt for `wandb`.** Choose option 3 to skip
  in non-interactive runs, or pass `logger=tensorboard` /
  `logger=dummy`.
- **Render gRPC is data-format-pinned.** Older releases warn /
  reject artifacts that pre-date them; check release notes when
  mixing client / server versions.
- **Asset-Harvester input only.** `export-external-assets`
  requires AH outputs; raw `.ply` files won't carry the
  per-asset cuboid metadata.
- **Difix variants are pluggable.** The container ships both the
  Cosmos Difix variant (default since 25.09 —
  `difix=cosmos_difix`) and the legacy Stable-Diffusion variant
  (`difix=sd_difix`). The newer Cosmos-Predict-based Fixer
  variants live in the `nurec-fixer` skill.
- **`render` ↔ `render-grpc` overlap.** `render` runs
  in-container without a server; `render-grpc` requires an active
  `serve-grpc`. Use `render` for batch novel-view jobs and
  `render-grpc` when you need actor editing, LiDAR rendering, or
  a long-lived service.

## Troubleshooting (top 4)

| Error | Cause | Fix |
|-------|-------|-----|
| `Unable to find image 'nvcr.io/nvidia/nre/nre:latest'` | Docker not authenticated to NGC. | `docker login nvcr.io` with `Username: $oauthtoken`. |
| `OOM Killed` / `CUDA out of memory` during training | Default recipe needs >= 48 GB VRAM. | Reduce `dataset.camera_ids`, lower `trainer.max_epochs`, switch to `trainer.precision=16-mixed`, or use a 48 GB+ GPU. |
| `serve-grpc` fails to find the USDZ | `--artifact-glob` must end in `.usdz` and be quoted. | Use e.g. `--artifact-glob /workdir/output/<RUN-ID>/usd-out/last.usdz`. |
| Edits silently ignored from `render-grpc --edit-assets` | Server started without `--enable-editing-actors`. | Restart `serve-grpc` with that flag. |

Full matrix in
[`references/troubleshooting.md`](references/troubleshooting.md).

## Teardown

Full inventory, ownership-recovery, and post-teardown verification
commands live in [`references/teardown.md`](references/teardown.md).
Headline: stop `serve-grpc` containers, `docker image rm`
nre/nre-tools, `rm -rf ${HOME}/.cache/nre` and your
`<output_dir>/<RUN-ID>/`. Do not revoke `NGC_API_KEY` unless you
suspect it has been leaked.

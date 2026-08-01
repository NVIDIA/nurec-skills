# NuRec Skills

NVIDIA Omniverse NuRec agent skills automate neural reconstruction for autonomous vehicle and robotics simulation, running the reconstruction and rendering workflow end to end.

The canonical home is <https://github.com/NVIDIA/nurec-skills>.

## Support

**Usage questions and discussion:** please post on the
[NVIDIA Developer Forum (Omniverse / NuRec)](https://forums.developer.nvidia.com/c/omniverse/platform/nurec/752).

**Code-level bugs, documentation issues, and feature requests:** file a
[GitHub issue](../../issues/new/choose) using the appropriate template. The
relevant NVIDIA responder is assigned automatically.

**Security vulnerabilities:** use
[NVIDIA's Vulnerability Disclosure Program](https://app.intigriti.com/programs/nvidia/nvidiavdp/detail).
Do not file security issues publicly in this repository.

## What's a skill?

A **skill** is a single Markdown file (plus a few companion files)
that an agent reads on demand to gain task-specific knowledge. Each
skill in this repo follows the [agentskills.io](https://agentskills.io)
convention: a YAML frontmatter block (name, description, trigger
keywords, compatibility, upstream pointer) followed by a hand-curated
recipe. Agents that support the standard — Cursor, Claude Code, Codex,
and others — can resolve a skill by **name**, regardless of where the
file is on disk.

These skills are **thin coordination layers**. They don't redistribute
NVIDIA source; instead, they teach an agent how to drive the public
NGC containers, GitHub repos, and HuggingFace artifacts that make up
the NuRec stack.

## Skills in this repo

Start with [`nurec-index`](./skills/nurec-index/SKILL.md) — it
routes any NuRec task to the right sibling skill below.

| Name | Folder | Pinned upstream | Purpose |
|------|--------|-----------------|---------|
| [`nurec-index`](./skills/nurec-index/SKILL.md) | `skills/nurec-index/` | hand-curated | Router. Read first. Picks the right skill for any NuRec task. |
| [`physical-ai-datasets`](./skills/physical-ai-datasets/SKILL.md) | `skills/physical-ai-datasets/` | hand-curated | Catalog of every NVIDIA `PhysicalAI-*` dataset on Hugging Face — AV, robotics, NuRec scenes, benchmarks. |
| [`ncore`](./skills/ncore/SKILL.md) | `skills/ncore/` | upstream `2026.04` | Convert any sensor recording (cameras, LiDAR, radar, IMU, depth, stereo) into [NCore V4](https://github.com/NVIDIA/ncore) — the format NRE consumes. Includes a converter template. |
| [`nre`](./skills/nre/SKILL.md) | `skills/nre/` | NRE `release_26.04` (`nvcr.io/nvidia/nre/{nre,nre-tools}`) | Train 3DGUT/3DGRT Gaussian reconstructions, render novel views (local or via gRPC), export PLY/mesh/depth, edit actors, evaluate quality. |
| [`asset-harvester`](./skills/asset-harvester/SKILL.md) | `skills/asset-harvester/` | [`NVIDIA/asset-harvester`](https://github.com/NVIDIA/asset-harvester) `main` (Apache-2.0) | Extract per-object 3D Gaussian Splat assets from sparse AV-clip views via SparseViewDiT + TokenGS. |
| [`nurec-fixer`](./skills/nurec-fixer/SKILL.md) | `skills/nurec-fixer/` | [`nvidia/DiffusionHarmonizer`](https://huggingface.co/nvidia/DiffusionHarmonizer) + [`NVIDIA/harmonizer`](https://github.com/NVIDIA/harmonizer) | Post-process, evaluate, or fine-tune novel-view renders with NVIDIA DiffusionHarmonizer, the current public harmonizer for reconstruction artifacts and inserted-object appearance. |

## Repo layout

```text
.agents/skills/  ──►  skills/                # symlink; both paths resolve to the same tree
skills/
├── nurec-index/
│   ├── SKILL.md                             # The router. Read first.
│   └── references/
│       ├── workflows.md
│       ├── teardown.md
│       └── discovery.md
├── physical-ai-datasets/
│   └── SKILL.md
├── ncore/
│   ├── SKILL.md
│   └── ncore_template/                      # Converter scaffold for new sensor formats
├── nre/
│   ├── SKILL.md
│   ├── references/                          # CLI / configuration / cookbook / rig JSONs / etc.
│   └── scripts/                             # validate_setup.py, session_warm_server.sh, …
├── asset-harvester/
│   ├── SKILL.md
│   ├── references/
│   ├── scripts/
│   └── tests.yaml
└── nurec-fixer/
    ├── SKILL.md
    ├── references/
    ├── scripts/
    └── tests.yaml
```

Each skill is a flat folder rooted at `skills/<name>/SKILL.md`. The
`.agents/skills/` path is a symlink onto `skills/`, so cross-skill
links like `../nre/SKILL.md` keep resolving regardless of which
prefix an agent indexes against. Upstream versions (NRE container
tag, Asset Harvester commit, DiffusionHarmonizer release branches)
are recorded in each skill's frontmatter `metadata:` block — bump
those when upstream releases shift.

## Using these skills

Most modern agent runtimes already auto-discover skills under
`skills/`, `.claude/skills/`, `.cursor/skills/`, or
`~/.cursor/skills/`. The two common ways to consume this repo:

### 1. Drop the repo next to your project

Clone into your project (or a parent directory the agent indexes):

```bash
git clone https://github.com/NVIDIA/nurec-skills.git
```

Then ask your agent to do anything in the trigger surface — e.g. "use
NuRec to render this clip", "convert my ROS 2 bag to NCore", "harvest
3D assets from this driving log". The agent picks the right skill via
`nurec-index` and follows the recipe.

### 2. Install a skill into your user-space

Symlink (or copy) one or more skills into your runtime's user-space
skills directory. For Cursor:

```bash
mkdir -p ~/.cursor/skills
ln -s "$(pwd)/skills/nurec-index"          ~/.cursor/skills/nurec-index
ln -s "$(pwd)/skills/physical-ai-datasets" ~/.cursor/skills/physical-ai-datasets
ln -s "$(pwd)/skills/ncore"                ~/.cursor/skills/ncore
ln -s "$(pwd)/skills/nre"                  ~/.cursor/skills/nre
ln -s "$(pwd)/skills/asset-harvester"      ~/.cursor/skills/asset-harvester
ln -s "$(pwd)/skills/nurec-fixer"          ~/.cursor/skills/nurec-fixer
```

Adjust the destination directory (`~/.claude/skills`, etc.) for other
runtimes.

## Prerequisites

These skills drive external NVIDIA infrastructure. Each skill lists
its own prerequisites in detail; the headline ones:

- **OS / arch:** Linux x86_64 with NVIDIA drivers (CUDA 12.x). aarch64
  is not supported by the NRE containers.
- **GPU:** Ampere or newer (compute capability ≥ 8.0). 16 GB VRAM is
  the practical floor for harmonizer inference; 24–48 GB+ is
  recommended for NRE training, and multi-GPU hosts are expected for
  DiffusionHarmonizer training.
- **Containers:** Docker 23+ and the NVIDIA Container Toolkit are
  required for `nre`, `nre-tools`, and `nurec-fixer`. NGC API key is
  required to pull `nvcr.io/nvidia/nre/*` and may also be required for
  `nvcr.io/nvidia/cosmos/*` container pulls.
- **Hugging Face:** an `HF_TOKEN` is required for any gated dataset or
  model (`nvidia/PhysicalAI-Autonomous-Vehicles*`,
  `nvidia/DiffusionHarmonizer`, `nvidia/DiffusionHarmonizer-Dataset`,
  `nvidia/asset-harvester`, …).
- **Python / conda:** required for the Asset Harvester install path
  and for the NCore in-process API (`pip install nvidia-ncore`).

## Third-party dependencies and bundled code

The skills in this repository are Markdown instructions plus lightweight
NVIDIA-authored helper files. They do not require third-party libraries to
be discovered or read, and they do not vendor or redistribute third-party
OSS source code or third-party binary dependencies. Bundled validation
scripts use only the Python standard library and host tools already called
out by the relevant skill.

Some workflows documented by the skills instruct users to install or run
external upstream tools, containers, Python packages, models, or datasets
from their original distribution channels. Those upstream artifacts are
not redistributed by this repository and retain their own licenses. See
[`THIRD_PARTY_NOTICES.md`](./THIRD_PARTY_NOTICES.md) for the repository's
third-party notice statement.

## Upstream sources of truth

Each skill is thin; the canonical artifacts live upstream:

- **NCore** — <https://github.com/NVIDIA/ncore>
  (spec: <https://nvidia.github.io/ncore/data/conventions.html>)
- **NRE / NuRec containers** — `nvcr.io/nvidia/nre/nre`,
  `nvcr.io/nvidia/nre/nre-tools` (NGC); official documentation page
  <https://docs.nvidia.com/nurec/>
- **Asset Harvester** — <https://github.com/NVIDIA/asset-harvester>
  (paper: <https://arxiv.org/abs/2604.18468>; demo:
  <https://huggingface.co/spaces/nvidia/asset-harvester>)
- **DiffusionHarmonizer** — <https://huggingface.co/nvidia/DiffusionHarmonizer>
  (open-source code: <https://github.com/NVIDIA/harmonizer>; paper:
  <https://arxiv.org/abs/2602.24096>)
- **Physical AI datasets** — <https://huggingface.co/nvidia> (filter
  `PhysicalAI-`); curated collection
  <https://huggingface.co/collections/nvidia/physical-ai>

When upstream releases shift, refresh the affected files in the
skill's `references/` and `scripts/` folders and bump the
`metadata:` block (and `version:`) in its `SKILL.md` frontmatter.

## Contributing

This project is not currently accepting external pull requests. Code-level
bugs, documentation issues, and feature requests are welcome through the
[issue templates](../../issues/new/choose). For usage questions and security
reports, use the channels listed in [Support](#support).

- Frontmatter follows the [agentskills.io](https://agentskills.io)
  schema (`name`, `description`, `version`, `license`, `metadata`).
  Trigger keywords belong inside `description:` so the runtime indexes
  them.
- New skills go under `skills/<folder>/` as a flat layout —
  `SKILL.md` at the root, with optional `references/`, `scripts/`,
  and `tests.yaml` siblings. Pin the upstream version inside the
  skill's frontmatter `metadata:` block, not in the folder path.
- After adding or renaming a skill, update the [`nurec-index`](./skills/nurec-index/SKILL.md)
  router so it knows how to route to it.

## License

This repository is released under dual **CC-BY-4.0 AND Apache-2.0**
terms. The full Apache-2.0 license text is distributed in
[`LICENSE`](./LICENSE), and the full Creative Commons Attribution 4.0
International license text is distributed in
[`LICENSE-CC-BY-4.0`](./LICENSE-CC-BY-4.0).

- **Code-only files** (for example, helper scripts, workflow YAML, test
  YAML, environment examples, and the `ncore_template/` Python package)
  are licensed under
  [**Apache-2.0**](https://www.apache.org/licenses/LICENSE-2.0). NVIDIA
  authored source files carry `SPDX-License-Identifier: Apache-2.0`
  headers.
- **Mixed documentation files** (for example, `SKILL.md`, reference
  Markdown files, and this README) are licensed under the repository's
  dual **CC-BY-4.0 AND Apache-2.0** terms. The skill frontmatter records
  this as `license: CC-BY-4.0 AND Apache-2.0`.

The skills only *drive* upstream NVIDIA artifacts (NGC containers,
GitHub repos, Hugging Face models and datasets). Those upstream
artifacts retain their own licenses — see each skill's `metadata:`
block for the upstream pointer.

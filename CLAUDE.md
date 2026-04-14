# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a fork of Google DeepMind's **AlphaFold 3** inference pipeline, modified to be a lightweight, hackable version that runs without Docker or MSA databases. Designed to run a single GPU / laptop workflow driven from `run_alphafold.py` or `run_alphafold.ipynb`.

The upstream package is a mixed Python / C++ codebase built with `scikit-build-core` + CMake. Python 3.11 is required; JAX 0.4.34 + dm-haiku 0.0.13 are pinned.

## Build / Install

This project is **not** `pip install -e .`-friendly because of the C++ extensions. The README's fork-specific install path is:

```sh
mamba create -n alphafold3 python=3.11 -y && mamba activate alphafold3
pip install "jax[cuda12]==0.4.34" --no-cache-dir
pip install absl-py dm-haiku==0.0.13 dm-tree jaxtyping==0.2.34 numpy tqdm typeguard==2.13.3 zstandard
pip install jax-triton==0.2.0 triton==3.1.0
mamba install -c conda-forge rdkit=2024.3.5 -y
mamba install -c salilab dssp
pip install . --no-dependencies   # builds the C++ extension via CMake
build_data                         # generates CCD ligand intermediate data
```

Model weights (`af3.bin.zst`) must be placed in `models/` — they are not in the repo and access requires a separate Google form.

## Running inference

Fork-specific "no MSA" usage (no data pipeline, XLA flash attention):

```sh
export XLA_FLAGS="--xla_gpu_enable_triton_gemm=false"
export XLA_PYTHON_CLIENT_PREALLOCATE=true
export XLA_CLIENT_MEM_FRACTION=0.95
# For V100s instead: XLA_FLAGS="--xla_disable_hlo_passes=custom-kernel-fusion-rewriter"

JAX_TRACEBACK_FILTERING=off python run_alphafold.py \
    --json_path=af_input/fold_input.json \
    --model_dir=models/ \
    --output_dir=af_output/ \
    --norun_data_pipeline \
    --flash_attention_implementation=xla
```

Key flags (see `run_alphafold.py --help` for the full list): `--run_data_pipeline` (CPU-only genetic/template search), `--run_inference` (GPU), `--flash_attention_implementation={triton,xla,cudnn}`. For "no-MSA" input JSONs, set `unpairedMsa: ""`, `pairedMsa: ""`, `templates: []` on each protein sequence.

## Tests

Uses `pytest` (declared in `[project.optional-dependencies].test`) and `absl.testing`.

```sh
pytest run_alphafold_test.py              # end-to-end integration test
pytest run_alphafold_data_test.py         # data pipeline test
pytest src/alphafold3/path/to/test.py     # unit tests live next to modules
pytest path/to/test.py::TestClass::test_x # single test
```

The end-to-end test shells out to `jackhmmer`/`nhmmer`/`hmmalign`/`hmmsearch`/`hmmbuild` via `shutil.which`; tests that need them are skipped if the binaries aren't on `PATH`. A recent repo-level commit (`0f865a6`) pins `jax_threefry_partitionable=False` so numerical comparisons stay stable — don't flip that without reason.

## Architecture

The Python package lives under `src/alphafold3/`. The pipeline is split into three big phases orchestrated by `run_alphafold.py`:

1. **Data pipeline** (`alphafold3.data`): turns a `folding_input.Input` into featurised tensors.
   - `pipeline.py` — top-level data pipeline (MSA search via HMMER tools in `data/tools/`, template search).
   - `msa.py`, `msa_store.py`, `msa_features.py`, `templates.py` — MSA + template handling.
   - `featurisation.py` — final feature assembly for the model.
   - `parsers.py` plus C++ helpers in `data/cpp/` for fast MSA/mmCIF parsing.

2. **Model** (`alphafold3.model`): Haiku modules for the AF3 network.
   - `model.py` + `model_config.py` — top-level Haiku `Model` and its config.
   - `network/` — the core architecture: `evoformer.py`, `diffusion_head.py`, `diffusion_transformer.py`, `confidence_head.py`, `distogram_head.py`, `template_modules.py`, `atom_cross_attention.py`, `featurization.py`, `modules.py`.
   - `components/` — reusable Haiku blocks.
   - `pipeline/` — inference-time batching/pipelining.
   - `scoring/`, `confidences.py`, `confidence_types.py` — output confidence metrics.
   - `params.py` — loads the `.bin.zst` checkpoint.
   - `post_processing.py` — turns raw model output into mmCIF structures.
   - `atom_layout/` — per-atom indexing/layout utilities used by both data and model.

3. **Structure I/O** (`alphafold3.structure`): in-memory structure representation backed by pandas-like `table.py` + `structure_tables.py`, with `mmcif.py` / `parsing.py` for serialization, plus C++ helpers in `structure/cpp/`.

Other top-level subpackages:
- `alphafold3.common` — `folding_input.py` (the canonical Input object parsed from the JSON schema), `resources.py`, `base_config.py`, `testing/`.
- `alphafold3.constants` — chemical constants, atom types, CCD component sets, periodic table.
- `alphafold3.jax` — custom JAX kernels including flash attention variants (`attention`, used via `--flash_attention_implementation`).
- `alphafold3.parsers` — lightweight Python parsers used outside the data pipeline.
- `alphafold3.cpp` — compiled extension module (built from `src/alphafold3/cpp.cc` + the `cpp/` subdirs via top-level `CMakeLists.txt`).
- `alphafold3.scripts` — miscellaneous CLI helpers.
- `alphafold3.build_data` — packaged as the `build_data` console script; generates derived CCD data needed at runtime.

## Input / output format

Input and output JSON/mmCIF formats are stable and documented — consult `docs/input.md` and `docs/output.md` before changing schema-adjacent code. `folding_input.Input` is the single source of truth for parsing/validating input JSON; do not parse the JSON ad-hoc elsewhere.

## Licensing constraints to keep in mind

Source code is **CC-BY-NC-SA 4.0**. Model parameters are under a separate [Weights Terms of Use](WEIGHTS_TERMS_OF_USE.md) and a [Prohibited Use Policy](WEIGHTS_PROHIBITED_USE_POLICY.md); inference outputs are under [Output Terms of Use](OUTPUT_TERMS_OF_USE.md). The weights may only be used if received directly from Google. Keep this in mind if adding features that touch distribution, redistribution, or commercial-looking code paths.

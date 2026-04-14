# Running this fork on RunPod

The cheapest, easiest way to run AF3 with arbitrary ligands when you don't already own an A100.

## Why RunPod over Lambda / Vast.ai

| | RunPod | Lambda Labs | Vast.ai |
|---|---|---|---|
| A100 80GB price | ~$1.19/hr (community) / ~$1.89/hr (secure) | ~$1.29/hr | ~$0.60–1.00/hr |
| Persistent volumes | ✅ Network Volumes ($0.05/GB/mo) | ❌ Reinstall each time | partial |
| Web terminal + Jupyter | ✅ One click | SSH only | varies |
| A100 availability | usually good | frequently sold out | varies wildly |
| Reliability | high | high | variable per host |
| **Best for** | **one-off + occasional** | clean SSH workflow | price-sensitive bulk |

**RunPod is the easiest because Network Volumes let you park the conda env + weights once and skip the 30-min reinstall on every subsequent job.**

## One-time setup (web UI)

1. Create an account at [runpod.io](https://runpod.io). Add credit ($10 is plenty to start).
2. **Storage → Network Volumes → New Network Volume**. ~100 GB, in a region with A100 capacity (e.g. `US-KS-2`, `EU-RO-1`). ~$5/mo.
3. **Pods → Deploy**:
   - GPU: **A100 80GB SXM** (Community Cloud is fine).
   - Template: **RunPod PyTorch 2.4** or any CUDA 12.1+ image.
   - Attach your network volume at `/workspace`.
   - Container disk: 50 GB.
   - Deploy.
4. Open the web terminal (or Connect → SSH).

## Install AF3 onto the network volume

Everything in `/workspace` survives pod deletion. Install mamba and the env there:

```sh
cd /workspace

# Mamba lives on the volume so future pods reuse it.
curl -L -O https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
bash Miniforge3-Linux-x86_64.sh -b -p /workspace/miniforge
source /workspace/miniforge/etc/profile.d/conda.sh

# Env also lives on the volume.
mamba create -p /workspace/envs/af3 python=3.11 -y
mamba activate /workspace/envs/af3

# AF3 install per fork's CLAUDE.md
pip install "jax[cuda12]==0.4.34" --no-cache-dir
pip install absl-py dm-haiku==0.0.13 dm-tree jaxtyping==0.2.34 numpy tqdm typeguard==2.13.3 zstandard
pip install jax-triton==0.2.0 triton==3.1.0
mamba install -c conda-forge rdkit=2024.3.5 -y
mamba install -c salilab dssp -y

# Clone the fork.
cd /workspace
git clone https://github.com/sness23/alphafold3.git
cd alphafold3
pip install . --no-dependencies
build_data
```

Then drop your weights file:

```sh
mkdir -p /workspace/alphafold3/models
# upload af3.bin.zst here via runpodctl, scp, or the web file browser
```

## Run a docking job

Create your input JSON at `/workspace/alphafold3/af_input/fold_input.json` with a `ligand` entity (CCD or SMILES — see [ligand-docking.md](ligand-docking.md)).

```sh
cd /workspace/alphafold3
source /workspace/miniforge/etc/profile.d/conda.sh
mamba activate /workspace/envs/af3

export XLA_FLAGS="--xla_gpu_enable_triton_gemm=false"
export XLA_PYTHON_CLIENT_PREALLOCATE=true
export XLA_CLIENT_MEM_FRACTION=0.95

JAX_TRACEBACK_FILTERING=off python run_alphafold.py \
    --json_path=af_input/fold_input.json \
    --model_dir=models/ \
    --output_dir=af_output/ \
    --norun_data_pipeline \
    --flash_attention_implementation=xla
```

For a small protein + one ligand this finishes in a few minutes. **Stop the pod when done** — billing is per-second.

## Subsequent runs

1. Deploy a new pod, attach the same network volume.
2. Open terminal:
   ```sh
   source /workspace/miniforge/etc/profile.d/conda.sh
   mamba activate /workspace/envs/af3
   cd /workspace/alphafold3
   # run inference
   ```
3. Stop the pod.

Zero reinstall. Compute cost per job: cents to a couple dollars. Idle cost: ~$5/mo for the volume.

## Cost example

| Step | Time | Cost |
|---|---|---|
| First setup (install + first inference) | ~30 min | ~$0.60 + ~$5 first month volume |
| Each subsequent small docking job | ~5 min | ~$0.10 |
| Idle (volume only, no pod running) | — | ~$5/mo |

Compare to ~$15–25 for the same first run on EC2 `p4d.24xlarge` spot.

## Doing it from the CLI

See [runpod-cli.md](runpod-cli.md) for the full CLI reference: installing `runpodctl`, network volume management, GraphQL API, file transfer, cost optimization, and troubleshooting.

The short version, assuming you've already done the one-time setup above and your network volume has the env + weights baked in:

```sh
# 1. Deploy a pod attached to your existing volume
runpodctl create pod \
  --name af3-job \
  --gpuType "NVIDIA A100 80GB PCIe" \
  --gpuCount 1 \
  --imageName runpod/pytorch:2.4.0-py3.11-cuda12.4.1-devel-ubuntu22.04 \
  --containerDiskSize 50 \
  --volumeSize 0 \
  --networkVolumeId YOUR_VOLUME_ID \
  --ports "22/tcp" \
  --communityCloud

# 2. Get the SSH command
runpodctl get pod POD_ID

# 3. Upload your input
scp -P PORT -i ~/.ssh/id_ed25519 fold_input.json root@HOST:/workspace/alphafold3/af_input/

# 4. SSH in and run inference
ssh -p PORT -i ~/.ssh/id_ed25519 root@HOST
# (on the pod)
cd /workspace/alphafold3
source /workspace/miniforge/etc/profile.d/conda.sh
mamba activate /workspace/envs/af3
python run_alphafold.py --json_path=af_input/fold_input.json --model_dir=models/ --output_dir=af_output/ --norun_data_pipeline --flash_attention_implementation=xla
exit

# 5. Download results
rsync -avz -e "ssh -p PORT -i ~/.ssh/id_ed25519" root@HOST:/workspace/alphafold3/af_output/ ./af_output/

# 6. Tear down
runpodctl remove pod POD_ID
```

A `dock.sh` script wrapping all of this end-to-end will be added separately.

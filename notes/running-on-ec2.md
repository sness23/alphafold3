# Running this fork on AWS EC2

## TL;DR

EC2 is awkward for AF3 because **AWS does not rent single A100s** — every A100/H100 instance is the full 8-GPU box. Use EC2 only if you already live in AWS or need batch throughput. Otherwise rent a single A100 from RunPod / Lambda Labs / Vast.ai (see [runpod-setup.md](runpod-setup.md)).

## Officially supported GPUs

From `docs/performance.md`: AF3 is tested on **A100 80GB** and **H100 80GB**. Inference time, compile-free:

| Tokens | A100 80GB | H100 80GB |
|---|---|---|
| 1,024 | 62 s | 34 s |
| 2,048 | 275 s | 144 s |
| 3,072 | 703 s | 367 s |
| 4,096 | 1,434 s | 774 s |
| 5,120 | 2,547 s | 1,416 s |

VRAM caps:

- A100 80GB / H100 80GB: up to 5,120 tokens comfortably.
- A100 40GB: up to 4,352 tokens with unified memory.
- V100 16GB: up to 1,280 tokens with unified memory.
- P100 16GB: up to 1,024 tokens.

A typical small protein + one ligand is a few hundred tokens — every option above works for the model itself. The constraint is dollars and availability.

## EC2 instance comparison

### Officially supported

| Instance | GPU | VRAM | On-demand | Spot (~) | Notes |
|---|---|---|---|---|---|
| `p4d.24xlarge` | 8× A100 | 40 GB | ~$32.77/hr | ~$10–13/hr | 4,352-token cap with unified memory. You pay for 8 GPUs; AF3 uses 1. |
| `p4de.24xlarge` | 8× A100 | 80 GB | ~$40.96/hr | ~$15/hr | The "official target." Comfortable up to 5,120 tokens. |
| `p5.48xlarge` | 8× H100 | 80 GB | ~$98.32/hr | varies | ~2× faster. Painful price for one job. |

A 5-minute job on a small protein still costs ~$3 on-demand for `p4d`, plus EBS, plus setup.

### Unsupported but cheaper

| Instance | GPU | VRAM | On-demand | Caveat |
|---|---|---|---|---|
| `g5.2xlarge` | 1× A10G | 24 GB | ~$1.21/hr | Off the supported path. Will OOM above ~1,500 tokens. May need the V100 fallback flag. |
| `p3.2xlarge` | 1× V100 | 16 GB | ~$3.06/hr | Use `XLA_FLAGS="--xla_disable_hlo_passes=custom-kernel-fusion-rewriter"` per `CLAUDE.md`. Capped near 1,280 tokens. |

For a small protein + one ligand on `g5.2xlarge`, it might just work and is the cheapest sane EC2 option.

## Practical recipe — `p4d.24xlarge` spot

1. **Quota**: request the **"All G and VT Spot Instance Requests"** and **"All P Spot Instance Requests"** quotas in the region you want. New accounts have these at 0. This is the #1 thing that blocks people for days.
2. **Region**: `us-east-1`, `us-west-2`, `eu-west-1` have the deepest p4d capacity. Spot pricing and availability swing hourly.
3. **AMI**: pick the **Deep Learning AMI (Ubuntu 22.04)**. NVIDIA drivers, CUDA, mamba already installed — saves ~1 hr.
4. **EBS**: 300+ GB gp3. Weights are ~1 GB but the conda env, CCD intermediate data, and JAX cache balloon fast.
5. **Install** per the fork's `CLAUDE.md` (mamba env, JAX 0.4.34 cuda12, `pip install . --no-dependencies`, `build_data`).
6. **Weights**: drop `af3.bin.zst` into `models/`. You need Google's access form first.
7. **Run inference** with the env vars from `CLAUDE.md`:
   ```sh
   export XLA_FLAGS="--xla_gpu_enable_triton_gemm=false"
   export XLA_PYTHON_CLIENT_PREALLOCATE=true
   export XLA_CLIENT_MEM_FRACTION=0.95
   ```
8. **Snapshot the EBS volume** when done. Next run skips the install entirely.

## Realistic costs

| Scenario | Cost |
|---|---|
| First run (install + debug + small inference) | $15–25 on p4d spot |
| Subsequent run (volume snapshot, small protein) | $3–5 on p4d spot |
| Same job on RunPod single A100 | ~$1–2 |

## Why I'd skip EC2

For occasional jobs, RunPod / Lambda Labs / Vast.ai rent you exactly one A100 instead of eight, at roughly **10× cheaper per job**. Same fork, same install steps. EC2 only wins if:

- You already have VPC/IAM/S3 infrastructure and your data lives there.
- You need batch throughput across many jobs simultaneously (then `p4d` × 8 GPUs in parallel is actually a deal).
- You have AWS credits to burn.

Otherwise, see [runpod-setup.md](runpod-setup.md).

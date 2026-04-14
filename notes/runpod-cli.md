# RunPod from the command line

A reference for driving RunPod entirely from your terminal — account setup, the official `runpodctl` CLI, the GraphQL API, file transfer, cost control, and troubleshooting.

> If you just want the AF3-specific flow, see [runpod-setup.md](runpod-setup.md). This doc is the deeper RunPod reference that backs it.

## Contents

- [Account and billing setup](#account-and-billing-setup)
- [API keys and SSH keys](#api-keys-and-ssh-keys)
- [Installing runpodctl](#installing-runpodctl)
- [Concepts: pods, templates, volumes, GPU types](#concepts-pods-templates-volumes-gpu-types)
- [Network volumes](#network-volumes)
- [Listing GPUs and pricing](#listing-gpus-and-pricing)
- [Creating pods](#creating-pods)
- [Connecting to pods](#connecting-to-pods)
- [Transferring files](#transferring-files)
- [Stopping and removing pods](#stopping-and-removing-pods)
- [The GraphQL API](#the-graphql-api)
- [Templates and custom images](#templates-and-custom-images)
- [Cost optimization](#cost-optimization)
- [Security](#security)
- [Troubleshooting](#troubleshooting)
- [Reference](#reference)

---

## Account and billing setup

1. Sign up at [runpod.io](https://runpod.io). Email + password or Google OAuth.
2. **Billing → Add Funds**. RunPod is **prepaid** — you load credit, it gets drawn down per second of pod runtime + per hour of volume storage. Start with $10–20.
3. Set up **Auto-pay** (optional) so credit tops up automatically when it falls below a threshold. Useful if you script jobs that might run unattended.
4. **Spending Limits**: under Settings → Spend Limit you can cap daily / weekly / monthly spend. Set a daily limit to a number that would horrify you if you accidentally left a pod running. Mine is $20/day.

There is no monthly fee. You only pay for:

- **Pod compute** — per-second while the pod exists (running OR stopped, see below).
- **Container disk** — included in the pod price; deleted when the pod is removed.
- **Network volume storage** — per-GB-month, persists independently of pods.

### Stopped vs running pods — important gotcha

A **stopped** pod still costs money for its container disk reservation, just at a lower rate (~10% of running). To pay nothing for compute, you must **remove (delete)** the pod, not just stop it. Use a network volume to persist your work across deletes.

---

## API keys and SSH keys

### API key

Settings → **API Keys** → Create. Save it somewhere (1Password, env file). It's shown only once.

```sh
export RUNPOD_API_KEY="rpa_xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

You can scope keys (read-only, full access). For scripts, create a dedicated key per script so you can rotate without breaking everything.

### SSH key

Settings → **SSH Public Keys** → Add. Paste your `~/.ssh/id_ed25519.pub`. RunPod injects this into every pod you create so you can SSH without password setup.

```sh
# If you don't have one yet
ssh-keygen -t ed25519 -C "your@email"
cat ~/.ssh/id_ed25519.pub  # paste into RunPod
```

---

## Installing runpodctl

The official Go-based CLI.

### Linux / macOS

```sh
wget -qO- cli.runpod.net | sudo bash
runpodctl version
```

### Homebrew (macOS)

```sh
brew install runpod/runpodctl/runpodctl
```

### From source

```sh
go install github.com/runpod/runpodctl@latest
```

### Authenticate

```sh
runpodctl config --apiKey "$RUNPOD_API_KEY"
```

This writes `~/.runpod/config.toml`. From now on every `runpodctl` command uses that key.

---

## Concepts: pods, templates, volumes, GPU types

| Term | What it is |
|---|---|
| **Pod** | A running container on a GPU host. Has its own ephemeral container disk. |
| **Template** | A saved (image, env vars, ports, disk size) tuple. RunPod ships many; you can create your own. |
| **Image** | A Docker image. Can be from RunPod's registry or any public Docker Hub / GHCR / your own. |
| **Container disk** | Local SSD on the host, scoped to one pod. **Deleted when the pod is removed.** |
| **Volume disk** | Larger local SSD attached to a pod. Same lifecycle — dies with the pod. Use the `volumeInGb` field. |
| **Network volume** | Persistent storage that survives pod deletion. Lives in one datacenter and can only be attached to pods in that same DC. |
| **GPU type** | NVIDIA model (A100 80GB SXM, H100 PCIe, RTX 4090, etc.). Each has a price and an availability per cloud type. |
| **Cloud type** | `SECURE` (datacenter-grade, audited hosts, SLA-ish, more expensive) vs `COMMUNITY` (vetted hobbyist hosts, cheaper, occasional preemption). |

For one-off ML jobs, **community + network volume** is the sweet spot.

---

## Network volumes

The single most important RunPod feature for serial jobs. Without one, every pod you create starts from zero — fresh image, no env, no weights, no scratch data. With one, you `mamba activate /workspace/envs/af3` and you're back where you left off.

### Creating one

The web UI is more reliable than the CLI for this:

1. **Storage → Network Volumes → New Network Volume**.
2. Pick a **datacenter**. This is permanent — the volume can only attach to pods in this DC. Pick one that has the GPUs you want.
   - `US-KS-2` — Kansas, generally good A100/H100 stock.
   - `EU-RO-1` — Romania, cheap, good A100 stock.
   - `CA-MTL-1` — Montreal, good H100 stock.
3. Size: 100 GB is plenty for AF3 + weights + a handful of outputs. Resize later by creating a new bigger volume and `rsync`-ing.
4. Name it (`af3-vol`).
5. Cost: ~$0.05/GB/month. 100 GB = ~$5/mo whether you use it or not.

### Attaching to a pod

When you create a pod, pass the volume ID and it mounts at `/workspace`. The CLI flag is `--networkVolumeId`. The GraphQL field is `networkVolumeId`.

### Deleting

Storage → Network Volumes → Delete. **Make sure no pods are attached.** Data is gone immediately, no recovery.

---

## Listing GPUs and pricing

```sh
runpodctl get cloud
```

Shows all available GPU types in all regions, with per-hour prices for both Community and Secure cloud. Filter:

```sh
runpodctl get cloud --gpuType "NVIDIA A100 80GB PCIe"
runpodctl get cloud --gpuCount 1 --secureCloud
```

Useful columns: `gpuTypeId`, `secureCloud`, `communityCloud`, `priceCommunity`, `priceSecure`, `lowestStockedRegion`.

For AF3, you want one of:

- `NVIDIA A100 80GB PCIe` — the bog-standard option. Usually ~$1.19/hr community.
- `NVIDIA A100-SXM4-80GB` — slightly faster, usually ~$1.49/hr community.
- `NVIDIA H100 PCIe` — ~2× faster than A100, usually ~$2.39/hr community.
- `NVIDIA H100 80GB HBM3` — SXM H100, fastest, ~$2.99/hr community.

For a small protein + ligand, A100 is plenty — H100 only pays off if you're running thousands of jobs.

---

## Creating pods

### Minimal `runpodctl create pod`

```sh
runpodctl create pod \
  --name af3-job \
  --gpuType "NVIDIA A100 80GB PCIe" \
  --gpuCount 1 \
  --imageName runpod/pytorch:2.4.0-py3.11-cuda12.4.1-devel-ubuntu22.04 \
  --containerDiskSize 50 \
  --volumeSize 0 \
  --networkVolumeId YOUR_VOLUME_ID \
  --ports "22/tcp,8888/http" \
  --communityCloud
```

Common flags:

| Flag | Purpose |
|---|---|
| `--name` | Display name only. |
| `--gpuType` | Exact `gpuTypeId` from `runpodctl get cloud`. |
| `--gpuCount` | 1 for AF3. |
| `--imageName` | Docker image. Defaults to RunPod's PyTorch base. |
| `--containerDiskSize` | GB of ephemeral disk for the container. 50 is fine if env lives on the network volume. |
| `--volumeSize` | GB of pod-local volume disk (separate from network volume). Set to 0 if you're using a network volume. |
| `--networkVolumeId` | Mount your persistent volume at `/workspace`. |
| `--ports` | Format `port/protocol[,port/protocol]`. `22/tcp` for SSH, `8888/http` for Jupyter. |
| `--communityCloud` / `--secureCloud` | Pick one. Community is cheaper. |
| `--env KEY=VALUE` | Set an env var. Repeatable. |
| `--templateId` | Use a saved template instead of specifying image+ports. |

### Output

```
pod "abcd1234" created for $1.19/hr
```

That `abcd1234` is the pod ID — save it, you need it for everything else.

### Spot pods (interruptible, cheaper)

There's also `runpodctl create pod --bidPrice 0.50` for spot bidding. RunPod calls these "spot" or "interruptible" pods. They cost ~30% less but can be killed if your bid is undercut. For AF3 jobs <30 min on small inputs, the math rarely favors spot — the savings vs. the risk of restart are small.

---

## Connecting to pods

### Via runpodctl

```sh
runpodctl get pod abcd1234
```

Prints the pod's status and an `ssh` command like:

```
ssh root@196.4.20.69 -p 31234 -i ~/.ssh/id_ed25519
```

The host and port change per pod. You can paste this directly into your shell.

### Add to ~/.ssh/config

Once the pod is running, add a stanza so you can `ssh af3` instead:

```
Host af3
    HostName 196.4.20.69
    Port 31234
    User root
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

The `StrictHostKeyChecking no` line is necessary because every new pod has a fresh host key on the same IP — without it you'll get warnings.

### Web terminal / Jupyter

The web UI shows a "Connect" button per pod with a one-click web terminal and Jupyter link. Useful for a sanity check, but for real work SSH is faster.

---

## Transferring files

Three options, easiest first.

### 1. `runpodctl send` / `receive` — one-shot, no setup

On your laptop:

```sh
runpodctl send ./af_input/fold_input.json
# → prints a code like "8338-galileo-collect-stratus"
```

On the pod:

```sh
runpodctl receive 8338-galileo-collect-stratus
```

The file is transferred peer-to-peer. Works in both directions. Magic-wormhole-style. Great for ad-hoc transfers, no ssh setup needed.

### 2. SCP / rsync — for many files or scripted use

```sh
# Upload
scp -P 31234 -i ~/.ssh/id_ed25519 fold_input.json root@196.4.20.69:/workspace/alphafold3/af_input/

# Download results
rsync -avz -e "ssh -p 31234 -i ~/.ssh/id_ed25519" \
  root@196.4.20.69:/workspace/alphafold3/af_output/ ./af_output/
```

If you've configured the SSH alias above:

```sh
scp fold_input.json af3:/workspace/alphafold3/af_input/
rsync -avz af3:/workspace/alphafold3/af_output/ ./af_output/
```

### 3. S3 / cloud storage — for big stuff

For weights or large datasets, host them on S3 / Cloudflare R2 / Backblaze B2 once and `aws s3 cp` from inside the pod. RunPod has fast egress.

---

## Stopping and removing pods

```sh
runpodctl stop pod abcd1234     # pause; cheap but not free
runpodctl start pod abcd1234    # resume
runpodctl remove pod abcd1234   # delete; container disk gone, volume safe
```

**For one-off jobs always `remove`, not `stop`.** A stopped pod still costs ~10¢/day for an A100, and it's easy to forget about.

To list everything you have running so you don't leak pods:

```sh
runpodctl get pod
```

I run this at the end of every session. Highly recommend a shell alias:

```sh
alias rp='runpodctl get pod'
```

---

## The GraphQL API

`runpodctl` doesn't expose every feature. For full control (especially scripting pod creation with network volumes in specific regions), use the GraphQL endpoint directly.

### Endpoint

```
POST https://api.runpod.io/graphql
Authorization: Bearer $RUNPOD_API_KEY
Content-Type: application/json
```

### Example: deploy a pod

```sh
curl -X POST https://api.runpod.io/graphql \
  -H "Authorization: Bearer $RUNPOD_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<'JSON'
{
  "query": "mutation Deploy($input: PodFindAndDeployOnDemandInput!) { podFindAndDeployOnDemand(input: $input) { id imageName machineId desiredStatus } }",
  "variables": {
    "input": {
      "cloudType": "COMMUNITY",
      "gpuCount": 1,
      "gpuTypeId": "NVIDIA A100 80GB PCIe",
      "name": "af3-job",
      "imageName": "runpod/pytorch:2.4.0-py3.11-cuda12.4.1-devel-ubuntu22.04",
      "containerDiskInGb": 50,
      "volumeInGb": 0,
      "networkVolumeId": "YOUR_VOLUME_ID",
      "ports": "22/tcp",
      "dockerArgs": ""
    }
  }
}
JSON
```

### Example: query pods

```sh
curl -X POST https://api.runpod.io/graphql \
  -H "Authorization: Bearer $RUNPOD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ myself { pods { id name desiredStatus runtime { uptimeInSeconds ports { ip publicPort privatePort } } } } }"}'
```

### Example: terminate

```sh
curl -X POST https://api.runpod.io/graphql \
  -H "Authorization: Bearer $RUNPOD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { podTerminate(input: { podId: \"abcd1234\" }) }"}'
```

### Schema reference

The full schema is browsable at <https://graphql-spec.runpod.io/>. Useful queries: `gpuTypes`, `myself`, `pod`. Useful mutations: `podFindAndDeployOnDemand`, `podStop`, `podResume`, `podTerminate`.

For Python scripting, the unofficial [`runpod-python`](https://github.com/runpod/runpod-python) SDK wraps these calls.

---

## Templates and custom images

A **template** is a reusable bundle of (image, env vars, ports, disk sizes, startup command). Saves you repeating the same flags on every `create pod`.

### Use an existing template

Browse <https://runpod.io/console/templates> for community templates (there's one for almost every popular ML stack). Each has a `templateId`. Then:

```sh
runpodctl create pod --templateId 12345abc --gpuType "NVIDIA A100 80GB PCIe" --gpuCount 1 --networkVolumeId YOUR_VOLUME_ID
```

### Create your own template

Web UI: Templates → New Template. Fields:

- **Container image**: `your/image:tag` from any public registry.
- **Container disk**: 50 GB (host scratch).
- **Volume disk**: 0 (network volume handles persistence).
- **Volume mount path**: `/workspace`.
- **Expose ports**: `22/tcp,8888/http`.
- **Docker command**: usually leave blank (use the image's default).
- **Environment variables**: e.g. `JUPYTER_PASSWORD=...`.

Then your `create pod` is just `--templateId X --gpuType Y --networkVolumeId Z`.

### Building a custom image for AF3

If you find yourself reinstalling AF3 often, bake it into an image:

```dockerfile
FROM runpod/pytorch:2.4.0-py3.11-cuda12.4.1-devel-ubuntu22.04

RUN apt-get update && apt-get install -y git curl && rm -rf /var/lib/apt/lists/*

RUN pip install "jax[cuda12]==0.4.34" --no-cache-dir && \
    pip install absl-py dm-haiku==0.0.13 dm-tree jaxtyping==0.2.34 numpy tqdm typeguard==2.13.3 zstandard && \
    pip install jax-triton==0.2.0 triton==3.1.0

# rdkit + dssp via conda
RUN curl -L -O https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh && \
    bash Miniforge3-Linux-x86_64.sh -b -p /opt/miniforge && \
    /opt/miniforge/bin/mamba install -n base -c conda-forge rdkit=2024.3.5 -y && \
    /opt/miniforge/bin/mamba install -n base -c salilab dssp -y

RUN git clone https://github.com/sness23/alphafold3.git /opt/alphafold3 && \
    cd /opt/alphafold3 && \
    pip install . --no-dependencies && \
    build_data
```

Push to Docker Hub / GHCR, then reference it in your template. Now every new pod is preinstalled — you just need to attach weights from your network volume.

---

## Cost optimization

| Tip | Savings |
|---|---|
| Use Community Cloud, not Secure | ~30% |
| Use a network volume to skip reinstalls | huge — avoids a $0.60 setup tax per job |
| Pick A100 80GB PCIe over SXM | ~20% |
| Pick A100 over H100 unless you need the speed | ~50% |
| Pick a region with cheaper electricity (EU-RO, US-KS) | 5–15% |
| **Always `remove`, not `stop`, after a job** | the only thing that gets you to $0 compute |
| Set a daily spend limit | prevents disasters |
| Use `runpodctl get pod` at the end of every session | catches leaked pods |
| Bake AF3 into a Docker image once | trades disk image build time for per-job speed |
| Don't oversize the network volume | every GB is $0.05/mo even when idle |

A typical AF3 docking job on a small protein:

| Approach | Cost |
|---|---|
| Fresh pod, install from scratch each time | ~$0.60 + 30 min |
| Network volume with prebuilt env | ~$0.10 + 5 min |
| Custom Docker image + network volume for weights | ~$0.05 + 2 min |

The custom image path takes an hour to set up but pays off after ~10 jobs.

---

## Security

- **Treat your API key like a password.** It can spend money. Don't commit it to git.
- Use **per-script API keys** so you can rotate without breaking everything.
- **SSH key**: use a passphrase-protected key, or use `ssh-agent` so it's not on disk in plain.
- **Pods are not isolated from each other** at the hardware level — same as any cloud. Don't put secrets on a pod that you wouldn't put on a shared Linux box.
- **Network volumes are not encrypted at rest by default.** If you have sensitive data, encrypt it client-side before uploading.
- **Community Cloud hosts are vetted but not audited.** For real PHI/HIPAA workloads use Secure Cloud or another provider.
- The `Authorization: Bearer` header in scripts can leak via shell history and `ps`. Use env vars and avoid passing keys as flags.

---

## Troubleshooting

### "No GPUs available"

Community Cloud capacity comes and goes. Try:

- A different `gpuType` (PCIe vs SXM).
- A different region — `runpodctl get cloud` shows stock per region.
- Secure Cloud (more expensive but more reliable).
- Wait 15 minutes and retry. Don't poll aggressively; capacity unlocks in batches.

### Pod stuck in "starting"

Usually means the host is downloading the Docker image. Big images (10 GB+) can take 5–10 min on first deploy. After that, the host caches the image and subsequent pods on the same host start in seconds. To skip this, use a smaller base image or bake your env into a slim image.

### `ssh: Connection refused`

Pod is "running" but SSH isn't ready yet. The container takes 10–30 seconds after the host marks it running. Wait, then retry.

### Network volume won't mount

The volume is in a different datacenter than the pod. Volumes are DC-pinned. Either pick a GPU in that DC, or create a new volume in the DC where the GPU is and `rsync` your data over.

### Out of disk space mid-job

Container disk is too small. Default is 20 GB. Set `--containerDiskSize 50` or move scratch into `/workspace` (the network volume).

### Can't find your pod's IP and port

```sh
runpodctl get pod abcd1234
```

Or in GraphQL: `myself { pods { id runtime { ports { ip publicPort privatePort } } } }`.

### Pod was killed without warning

You hit your spend limit, ran out of credit, or your community host went offline. Check Settings → Spend Limit and Billing → Balance. Set up auto-pay if this is a problem.

### Inference is slow

- Are you actually on the GPU? `nvidia-smi` inside the pod.
- Is your `gpuType` what you asked for? The host can substitute occasionally.
- AF3-specific: the first run is slow because of XLA compilation. Subsequent runs in the same pod are 5–10× faster. Don't benchmark the first run.

---

## Reference

- [RunPod docs home](https://docs.runpod.io/)
- [runpodctl on GitHub](https://github.com/runpod/runpodctl)
- [GraphQL schema explorer](https://graphql-spec.runpod.io/)
- [runpod-python SDK](https://github.com/runpod/runpod-python)
- [GPU pricing page](https://runpod.io/pricing)
- [Community templates](https://runpod.io/console/templates)

# RunPod account setup — first time

Walk through this once before you try to deploy anything. ~15 minutes of clicking, plus a 1–3 day wait for the AF3 weights approval email (start that step first so it runs in parallel).

## Checklist

- [ ] Create RunPod account
- [ ] Add credit ($10–20)
- [ ] Set a spend limit
- [ ] Create an API key
- [ ] Add SSH public key
- [ ] Submit Google's AF3 weights form
- [ ] Create a network volume
- [ ] Install `runpodctl` on your laptop

---

## 1. Create the account

Go to [runpod.io](https://runpod.io). Sign up with email or Google OAuth. Verify your email.

## 2. Add credit

**Billing → Add Funds**. RunPod is **prepaid** — you load credit, it gets drawn down per-second of pod runtime plus per-GB-month of network volume storage. Start with $10–20. There is no monthly fee.

## 3. Set a spend limit

This is your safety net against leaked pods. The menu name shifts around, but try these in order:

- **Settings → Billing**
- **Billing** in the left sidebar
- **Avatar (top right) → Account / Billing**

Look for **"Spending Limit"**, **"Spend Limit"**, or **"Auto-pay limit"**. Set a daily cap that would horrify you if you hit it (mine is $20/day).

If you genuinely can't find it in the current UI, don't block on it. The practical safety net is to **always run `runpodctl get pod` at the end of every session** to confirm nothing is still running.

## 4. Create an API key

This is what `runpodctl` and any scripts will use to talk to RunPod.

1. Click your **avatar (top right) → Settings**, or find **Settings** in the left sidebar.
2. Find **API Keys** in the settings menu.
3. Click **+ Create API Key** (or "New API Key").
4. Name it something descriptive: `laptop`, `af3-cli`, `dock-script`. Per-purpose keys make rotation painless later.
5. Permissions: **Read & Write** (or "All Permissions"). Read-only cannot create pods.
6. Create. **The key is shown exactly once.** It looks like `rpa_xxxxxxxxxxxxxxxxxxxxxxxxxxxx`. Copy it immediately to a password manager or a local file you don't commit.
7. Export it on your laptop:
   ```sh
   export RUNPOD_API_KEY="rpa_xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
   ```
   Make it persistent by adding that line to `~/.zshrc` (or `~/.bashrc`). Then `source ~/.zshrc`.

If you lose an API key you cannot view it again — delete it and create a new one.

## 5. Add an SSH public key

RunPod injects this into every pod you create so you can SSH in without password setup.

1. If you don't have an SSH key yet:
   ```sh
   ssh-keygen -t ed25519 -C "your@email"
   ```
   Press enter to accept defaults. Use a passphrase if you want extra security.
2. Print your public key:
   ```sh
   cat ~/.ssh/id_ed25519.pub
   ```
3. RunPod web UI: **Settings → SSH Public Keys → Add**. Paste the entire line (starts with `ssh-ed25519`, ends with your email).

## 6. Submit Google's AF3 weights form

**Do this first if you haven't already** — approval takes 1–3 business days, sometimes same-day, and you can do everything else in parallel while you wait.

- Form: <https://forms.gle/svvpY4u2jsHEwWYS6>
- You'll provide your name, affiliation, and agree to the non-commercial terms.
- Approval arrives via email with a download link for `af3.bin.zst` (~1 GB).
- The link comes from a Google / DeepMind address — search your inbox for `alphafold` if you submitted previously.

### Can I download the weights from somewhere else?

**No.** Three reasons:

1. **License**: per `WEIGHTS_TERMS_OF_USE.md` in this repo and the fork's `CLAUDE.md`, *"The weights may only be used if received directly from Google."* Any third-party mirror violates the license. Your `OUTPUT_TERMS_OF_USE.md` rights only attach if you received the weights legitimately.
2. **Integrity**: there is no published checksum from Google to verify a third-party copy against. A mirrored binary could be tampered with.
3. **It's free and fast anyway**: the form is the path of least resistance.

If you submitted the form months ago, search your email for `alphafold` from a Google or DeepMind sender — your approval link may already be in your inbox.

## 7. Create a network volume

This is the persistent storage that lets you skip reinstalling AF3 every time you spin up a pod. See [runpod-cli.md § Network volumes](runpod-cli.md#network-volumes) for the full background.

1. **Storage → Network Volumes → New Network Volume**.
2. **Datacenter**: pick one with good A100 stock. The volume can only be attached to pods in the same DC, so this choice is permanent for this volume.
   - `US-KS-2` — Kansas
   - `EU-RO-1` — Romania (often cheapest)
   - `CA-MTL-1` — Montreal
3. **Size**: 100 GB. Plenty for AF3 + weights + a stack of outputs. Resize later by creating a new bigger volume and `rsync`-ing.
4. **Name**: `af3-vol`.
5. Create. **Save the volume ID** — you'll pass it to every `runpodctl create pod`.

Cost: ~$5/month for 100 GB. You pay this whether you have a pod attached or not.

## 8. Install `runpodctl`

On your laptop:

```sh
# Linux / macOS
wget -qO- cli.runpod.net | sudo bash

# OR Homebrew (macOS)
brew install runpod/runpodctl/runpodctl

# Verify
runpodctl version

# Authenticate
runpodctl config --apiKey "$RUNPOD_API_KEY"
```

Test with:

```sh
runpodctl get pod
```

Should return an empty list (no pods yet) and exit 0. If you get an auth error, double-check the API key.

---

## You're done with one-time setup

Next steps once weights have arrived and the volume exists:

1. Deploy a pod attached to your network volume — see [runpod-setup.md](runpod-setup.md).
2. Install AF3 onto the volume (one-time, ~30 min).
3. Upload weights (one-time, ~1 min).
4. Run docking jobs forever after for cents per job.

For the full CLI reference (deploy flags, file transfer, GraphQL, troubleshooting), see [runpod-cli.md](runpod-cli.md).

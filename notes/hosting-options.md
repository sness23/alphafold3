# Hosted alternatives to running AF3 yourself

If you only need a few jobs, you may not need to run this fork at all. Here are the hosted options as of April 2026.

## ❌ AlphaFold Server (alphafoldserver.com) — does NOT support arbitrary ligands

Google's official free hosted AF3.

- **10 jobs/day**, resets at midnight UTC.
- Web UI only, **no API**.
- Non-commercial use only.
- **Allowlist of ~20 ligands only**: ATP, ADP, AMP, GTP, GDP, FAD, NADP, NADPH, NDP, heme, heme C, myristic/oleic/palmitic/citric acid, chlorophyll A/B, bacteriochlorophyll A/B, plus selected metal ions and standard nucleic acids.
- **You cannot upload an arbitrary SMILES.** This is the key reason to avoid it for novel ligand work.

If your ligand is on the allowlist, this is the easiest option in the world. Otherwise, skip it.

## ✅ Neurosnap — arbitrary SMILES, web UI, credit-based

[neurosnap.ai](https://neurosnap.ai/) hosts several AF3-equivalent models with full arbitrary-ligand support:

- **Boltz-2 (AlphaFold3)** — current best open AF3-quality model, recommended.
- **Boltz-1 (AlphaFold3)** — older, still good.
- **Protenix (AlphaFold3)** — ByteDance's reimplementation.
- **OpenFold3 (AlphaFold3)** — community reimplementation.
- **Chai-1**, **IntelliFold**, **RoseTTAFold3** — alternatives.

All support protein–ligand docking with arbitrary SMILES through the browser, no install. Free tier to start; credits afterwards. You see an estimated credit cost before each job.

**Best pick for ad-hoc arbitrary-ligand docking with no infrastructure work.**

## ✅ Tamarind Bio — free hosted AF3

[tamarind.bio/tools/alphafold3](https://www.tamarind.bio/tools/alphafold3) offers free hosted AlphaFold 3 with a web UI and SMILES support. Worth trying as a free alternative or sanity check against Neurosnap.

## ✅ This fork on your own GPU

Full SMILES and custom CCD support, no rate limits, programmatic access. But you need:

- Model weights from Google's access form.
- An A100/H100-class GPU (rented or owned).
- ~30 min of install time the first time.

See [running-on-ec2.md](running-on-ec2.md) and [runpod-setup.md](runpod-setup.md).

## Decision

| Use case | Pick |
|---|---|
| One job, ligand on the allowlist | AlphaFold Server |
| One-off arbitrary ligand, no install | Neurosnap (Boltz-2) or Tamarind |
| Tens to hundreds of jobs, scripted | This fork on RunPod / Lambda |
| Already in AWS, batch pipeline | This fork on EC2 p4d/p4de |

## Sources

- [AF3 input format docs](https://github.com/google-deepmind/alphafold3/blob/main/docs/input.md)
- [EBI: Using the AlphaFold 3 source code](https://www.ebi.ac.uk/training/online/courses/alphafold/alphafold-3-and-alphafold-server/using-the-alphafold-3-source-code/)
- [AlphaFold Server guides](https://alphafoldserver.com/guides)
- [Neurosnap Boltz-2](https://neurosnap.ai/service/Boltz-2%20(AlphaFold3))
- [Tamarind Bio AlphaFold3](https://www.tamarind.bio/tools/alphafold3)

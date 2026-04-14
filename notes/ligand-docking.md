# Docking arbitrary ligands with this fork

**Yes.** AlphaFold 3 supports arbitrary ligands as a first-class input entity. You don't run a separate docking step — AF3 jointly predicts the protein + ligand complex from sequence and chemistry, placing the ligand itself (no pocket, no starting pose required).

## Three ways to specify a ligand

In your `fold_input.json`, add a `ligand` entity alongside the protein. See `docs/input.md:296` for the full schema.

### 1. CCD code — for known ligands and ions

```json
{
  "ligand": {
    "id": "L",
    "ccdCodes": ["ATP"]
  }
}
```

Works for anything in the [PDB Chemical Component Dictionary](https://www.wwpdb.org/data/ccd), including ions like `MG`. Ions are specified as `ligand` entities — AF3 has no separate "ion" type.

### 2. SMILES — for arbitrary small molecules

```json
{
  "ligand": {
    "id": "L",
    "smiles": "CC(=O)OC1C[NH+]2CCC1CC2"
  }
}
```

This is the path for novel compounds not in CCD. Notes:

- Backslashes must be JSON-escaped: `"CCC[C@@H](O)CC\\C=C\\C=C\\C#CC#C\\C=C\\CO"`.
- Use canonical SMILES, not isomeric, for better generalization (the model was trained on canonical reps from PDB).
- SMILES ligands cannot define covalent bonds to other entities — there are no atom names to bond to.

### 3. User-provided CCD mmCIF — for custom ligands needing covalent bonds

If you need to define a custom ligand *and* covalently bond it to a protein residue, you must supply a CCD mmCIF entry with named atoms. SMILES alone won't work for this case.

## Performance vs. classic docking tools

From the AF3 paper: **76% of AF3's predicted poses land within 2 Å of the experimental structure**, vs. 38% for DiffDock, the previous best specialized docking tool. AF3 is a strong general-purpose docker.

## What you need to run it

- Model weights `af3.bin.zst` in `models/` — requires Google's separate access form.
- An A100/H100-class GPU (see [running-on-ec2.md](running-on-ec2.md) and [runpod-setup.md](runpod-setup.md)).
- The fork's install per `CLAUDE.md`.

For "no MSA" usage (the fork's lightweight path):

```sh
JAX_TRACEBACK_FILTERING=off python run_alphafold.py \
    --json_path=af_input/fold_input.json \
    --model_dir=models/ \
    --output_dir=af_output/ \
    --norun_data_pipeline \
    --flash_attention_implementation=xla
```

In your input JSON, set `unpairedMsa: ""`, `pairedMsa: ""`, `templates: []` on each protein sequence.

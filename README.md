# hupopsi-pymztabm-handson

Hands-on materials for converting RIKEN-LIPIDOMICS-style MS-DIAL alignment TSV
exports into mzTab-M 2.0.0-M files with
[`pymztab-m`](https://github.com/HUPO-PSI/pymzTab-m).

The repository contains:

- Example RIKEN lipidomics TSV input pairs under `RIKEN_LIPIDOMICS/`.
- A reusable converter skill under `skills/rikenlipidomics2mztabm/`.
- Example mzTab-M outputs under `output/`.

Success means the converter builds an in-memory `MzTabM` object and
`MzTabM.validate(...)` reports zero ERROR-level messages. WARNING-level
messages are printed but do not fail the conversion.

## Repository Layout

| Path | Purpose |
|---|---|
| `RIKEN_LIPIDOMICS/*_forMTD.tsv` | Per-sample metadata tables exported for each study |
| `RIKEN_LIPIDOMICS/*_forSMLSMF.tsv` | MS-DIAL alignment exports with SML/SMF source columns and sample abundances |
| `skills/rikenlipidomics2mztabm/` | Converter skill, Python environment, CLI script, and metadata example |
| `output/claudecode/` | Existing Claude Code generated mzTab-M outputs |
| `output/codex/` | Existing Codex generated mzTab-M outputs |

## Input Format

Each study is represented by two TSV files with matching sample names:

- `<study>_forMTD.tsv`: metadata rows for each sample column, including
  `Public/Private`, `Category`, `Tissue/Species`, `Genotype/Background`,
  `Perturbation`, `Diet/Culture`, biological replicate, technical replicate,
  and unit.
- `<study>_forSMLSMF.tsv`: MS-DIAL alignment rows containing fields such as
  `Alignment ID`, `Average Rt(min)`, `Average Mz`, `Metabolite name`,
  `Adduct type`, `Formula`, `SMILES`, `INCHIKEY`, score fields, spectrum
  references, and one abundance column per sample.

By default, columns whose metadata `Category` is `Blank` are excluded from the
converted mzTab-M file. Use `--include-blank` to keep them.

## Setup

Install [`uv`](https://docs.astral.sh/uv/) if needed, then sync the converter
environment:

```bash
cd skills/rikenlipidomics2mztabm
uv sync
```

The environment requires Python 3.10 or newer and installs `pymztabm` from the
pinned Git source in `pyproject.toml`.

## Convert the Example Data

From the repository root:

```bash
mkdir -p output/codex/current

cd skills/rikenlipidomics2mztabm

uv run python scripts/tsv_to_mztabm.py \
  ../../RIKEN_LIPIDOMICS/1_Mouse_Adrenal_gland_1_forMTD.tsv \
  ../../RIKEN_LIPIDOMICS/1_Mouse_Adrenal_gland_1_forSMLSMF.tsv \
  ../../output/codex/current/1_Mouse_Adrenal_gland_1.mztab

uv run python scripts/tsv_to_mztabm.py \
  ../../RIKEN_LIPIDOMICS/2_Mouse_Brain_1_forMTD.tsv \
  ../../RIKEN_LIPIDOMICS/2_Mouse_Brain_1_forSMLSMF.tsv \
  ../../output/codex/current/2_Mouse_Brain_1.mztab
```

Expected result for the current example files:

- `1_Mouse_Adrenal_gland_1.mztab`: zero validation errors, with warnings.
- `2_Mouse_Brain_1.mztab`: zero validation errors, with warnings.

Warnings such as unreferenced `study_variable` rows or recommended but missing
confidence values are currently tolerated. Any ERROR-level validator message
makes the command exit non-zero.

## Converter CLI

```bash
uv run python scripts/tsv_to_mztabm.py \
  <FOR_MTD.tsv> <FOR_SMLSMF.tsv> <OUTPUT.mztab> \
  [--metadata META.json] \
  [--format tsv|json|yaml] \
  [--include-blank]
```

Options:

- `--metadata META.json`: override study-level mzTab-M metadata.
- `--format tsv|json|yaml`: choose output format. TSV is the default.
- `--include-blank`: retain samples marked as `Blank`.

The script prints a validation summary plus the first validator messages.

## Study-Level Metadata

The RIKEN TSV files do not contain complete study-level metadata such as
instrument details, contacts, publications, software versions, scan polarity,
database metadata, or mzML locations. The converter supplies conservative
defaults, but those should be replaced for real submissions.

Use `scripts/example_metadata.json` as a template:

```bash
uv run python scripts/tsv_to_mztabm.py \
  forMTD.tsv forSMLSMF.tsv out.mztab \
  --metadata scripts/example_metadata.json
```

Notes:

- Omit `contact[].email` unless it is a real deliverable address.
- CV terms use objects like
  `{"cv_label": "MS", "cv_accession": "MS:1003082", "name": "MS-DIAL"}`.
- `publication[]` entries can be lists of `"type:accession"` strings, such as
  `["pubmed:12345678", "doi:10.1234/example"]`.

## What the Converter Maps

The converter builds mzTab-M through the `pymztab-m` model classes rather than
hand-writing rows. It currently:

- Converts per-sample metadata into `sample`, `ms_run`, `assay`, and
  `study_variable` metadata rows.
- Converts each usable MS-DIAL alignment row into one
  `SmallMoleculeSummary`, one `SmallMoleculeFeature`, and one
  `SmallMoleculeEvidence`.
- Normalizes MS-DIAL adduct strings such as `[M+H]+` to mzTab-M-compatible
  forms such as `[M+H]1+`.
- Uses `Average Mz` as feature/evidence experimental m/z and assumes charge 1.
- Uses `Average Rt(min)` as retention time after conversion to seconds.

## Adapting the Workflow

`scripts/tsv_to_mztabm.py` is intentionally specific to the
RIKEN-LIPIDOMICS/MS-DIAL TSV layout. For a different vendor export or archive
format, adapt the parsing layer (`_read_tsv`, `parse_mtd_tsv`, and the SML/SMF
column mapping) while keeping `build_mztabm()` as the mzTab-M construction
core.

Any adaptation should preserve the main quality gate: in-memory validation must
report zero ERROR-level messages before the output is considered successful.

## License

See [`LICENSE`](LICENSE).

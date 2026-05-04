# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Nextflow DSL2 metagenomics shotgun sequencing pipeline built on the nf-core framework. It accepts paired-end (or single-end) FASTQ reads, removes host contamination, and runs multiple taxonomic/functional profilers in parallel.

## Running the Pipeline

**Minimal test (uses bundled test data):**
```bash
nextflow run jianhong/shotgun -profile test,docker
```

**Kraken2-only test:**
```bash
nextflow run jianhong/shotgun -profile test_kraken2,docker
```

**Production run:**
```bash
nextflow run jianhong/shotgun \
  -profile docker \
  --input samplesheet.csv \
  --genome hg38 \
  --outdir results
```

**Resume a failed run:**
```bash
nextflow run jianhong/shotgun -profile docker --input samplesheet.csv -resume
```

**Skip individual profilers** (useful during development or when databases are unavailable):
```bash
nextflow run jianhong/shotgun -profile docker --input samplesheet.csv \
  --skip_kraken2 --skip_centrifuge --skip_motus
```

**Lint with nf-core tools:**
```bash
nf-core lint
```

## Samplesheet Format

Input CSV (`--input`) must have these columns:
```
sample,fastq_1,fastq_2
SAMPLE_01,/path/to/R1.fastq.gz,/path/to/R2.fastq.gz
SAMPLE_SE,/path/to/R1.fastq.gz,          # single-end: leave fastq_2 blank
```

Multiple rows with the same `sample` name are treated as separate runs and concatenated by `CAT_FASTQ` before processing. Sample IDs are parsed by stripping the `_T{n}` suffix added by `bin/check_samplesheet.py`.

## Pipeline Architecture

Execution flows through three layers: **entry point → subworkflows → modules**.

### Entry Point
`main.nf` bootstraps genome params via `WorkflowMain`, then delegates to the single named workflow `SHOTGUN` defined in `workflows/shotgun.nf`.

### Core Workflow (`workflows/shotgun.nf`)
Orchestrates all subworkflows in order:

1. **`PREPARE_GENOME`** — Downloads/unpacks all databases (bowtie2 index, metaphlan, kaiju, kraken2, humann nucleotide + protein, motus, centrifuge). If a `--*_db` param is supplied, it uses that path instead of downloading. This is the longest setup step.
2. **`INPUT_CHECK`** — Validates the samplesheet and emits `(meta, reads)` channels. The `meta` map carries `id`, `single_end`, and `reads_length` (rounded to nearest 5 bp, used by Bracken).
3. **`CAT_FASTQ`** — Merges multi-run samples.
4. **`FASTQC`** — Optional raw-read QC (skip with `--skip_fastqc`).
5. **`KNEAD_DATA`** — Host decontamination via KneadData (bowtie2 alignment + trimming + TRF repeat filtering). Outputs both `reads` (single-end merged) and `pairs` (paired-end) channels. Set `--use_single_kneaddata` to force downstream tools to use only the merged single-end output.
6. **Profiling subworkflows** (each independently skippable):
   - `METAPHLAN` — runs on `KNEAD_DATA.out.reads`
   - `KAIJU` — runs on `ch_knead_data` (paired or single depending on `--use_single_kneaddata`)
   - `KRAKEN2` — includes Bracken re-estimation and Krona/Pavian/Sankey visualization
   - `HUMANN` — functional profiling (gene families, pathway coverage, pathway abundance)
   - `CENTRIFUGE`
   - `MOTUS`
7. **`MULTIQC`** — Aggregates all QC reports.

### Subworkflows (`subworkflows/local/`)
Each profiler subworkflow follows the pattern: `*_INSTALL` (database) → `*_RUN` (per sample) → `*_MERGE` (aggregate) → visualization (`KRONA`, `SANKEY`).

### Modules
- `modules/local/` — Custom processes for all tools in this pipeline.
- `modules/nf-core/` — Shared nf-core community modules (bowtie2, blast, fastqc, kraken2, multiqc, etc.).

### Configuration Layers
- `nextflow.config` — Global defaults, profiles (docker/singularity/conda/etc.), and the `check_max()` helper.
- `conf/base.config` — Resource labels: `process_low` (2 CPU/12 GB), `process_medium` (6 CPU/36 GB), `process_high` (12 CPU/120 GB).
- `conf/modules.config` — Per-process `ext.args`, `ext.prefix`, and `publishDir` overrides. **This is where tool CLI flags are set**, not in the module `.nf` files themselves.
- `conf/test*.config` — Named test profiles that supply small databases and skip slow tools.

### Groovy Library (`lib/`)
- `WorkflowShotgun.groovy` — Parameter validation and `getReadsLength()` (reads first FASTQ record to determine approximate read length, rounded to 5 bp).
- `WorkflowMain.groovy` — iGenomes genome attribute resolution.
- `NfcoreSchema.groovy` / `NfcoreTemplate.groovy` — nf-core parameter schema validation and email/summary reporting.

## Key Development Patterns

**Adding a new profiler tool:** Create a module in `modules/local/`, a subworkflow in `subworkflows/local/`, add a `skip_*` param to `nextflow.config`, wire it in `workflows/shotgun.nf`, and add `publishDir` + `ext.args` entries to `conf/modules.config`.

**Changing tool arguments:** Edit `conf/modules.config` using `withName: PROCESS_NAME { ext.args = '...' }`. Never hardcode arguments inside module `.nf` files.

**Database paths:** All databases are managed by `PREPARE_GENOME`. Pass a local path via `--*_db` to skip auto-download. Databases that are tarballs (`.tgz`/`.tar.gz`) are unpacked by `UNTAR` before being passed downstream.

**Output directory structure:** Results land under `--outdir` (default `./results`), organized as `<tool>/{bySamples,merged,krona,db,...}`. Pipeline execution reports (timeline, DAG, trace) go to `results/pipeline_info/`.

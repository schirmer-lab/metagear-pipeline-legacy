# Workflows

The `schirmer-lab/metagear` pipeline groups its work into five entry-point workflows, selected at run time with the `--workflow` parameter:

| Workflow | Purpose | Input | Output | Cost |
|---|---|---|---|---|
| [download_databases](download_databases.md) | One-time install of KneadData, MetaPhlAn, and HUMAnN references | — | Reference databases at the paths you supply | Disk + network |
| [qc_dna](qc_dna.md) | Adapter/quality trimming and host decontamination of DNA reads | Raw DNA FASTQ | Clean paired reads + QC report | Medium |
| [qc_rna](qc_rna.md) | Same flow as `qc_dna`; intended for metatranscriptomic input | Raw RNA FASTQ | Clean paired reads + QC report | Medium |
| [microbial_profiles](microbial_profiles.md) | Reference-based taxonomic and functional profiling | Clean reads | MetaPhlAn4 species table + HUMAnN3 gene-family and pathway tables | Medium–High |
| [gene_analysis](gene_analysis.md) | De novo assembly, gene catalog, MSP analysis | Clean reads | Gene/protein catalogs, abundance matrices, MSPs | High |

## Recommended order

Run `download_databases` once per machine, then quality-control the raw reads, then choose whichever downstream workflows match your scientific question. The two analysis workflows (`microbial_profiles`, `gene_analysis`) are complementary and can run on the same QC'd reads.

```bash
# 1. One-time database setup
nextflow run schirmer-lab/metagear -profile docker \
  --workflow download_databases --outdir databases/

# 2. QC the raw reads
nextflow run schirmer-lab/metagear -profile docker \
  --workflow qc_dna --input raw.csv --outdir qc/

# 3. Pick one or both analyses
nextflow run schirmer-lab/metagear -profile docker \
  --workflow microbial_profiles --input clean.csv --outdir profiles/

nextflow run schirmer-lab/metagear -profile docker \
  --workflow gene_analysis --input clean.csv --outdir genes/
```

## Picking a workflow

- **Which organisms are present, and what functions can they perform?** → `microbial_profiles` (reference-based, no assembly, fast).
- **What genes are in this sample, including novel ones?** → `gene_analysis` (assembly-based, captures genes absent from reference databases, builds metagenomic species pangenomes).

## Input samplesheet

All workflows except `download_databases` take a CSV samplesheet via `--input`. Columns: `sample`, `fastq_1`, `fastq_2` (R2 optional for single-end). See [usage.md](../usage.md) for the full samplesheet contract.

## Wrapper CLI

The [`metagear-tools`](https://github.com/schirmer-lab/metagear-tools) CLI wraps these workflows so that day-to-day invocations look like `metagear qc_dna --input samples.csv`. The pipeline runs identically with or without the wrapper; the Nextflow invocations shown here are the canonical form.

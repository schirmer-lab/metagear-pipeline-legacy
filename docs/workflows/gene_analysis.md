# gene_analysis

De novo gene-centric analysis: assemble reads, call genes, build a non-redundant gene catalog, quantify it back across all samples, and group genes into metagenomic species pangenomes (MSPs). Use this when you need a sample-specific gene catalog or want to characterize organisms that are not in the MetaPhlAn reference.

## What it does

1. **Assembly** — MEGAHIT assembles each sample into contigs.
2. **Gene calling** — Prodigal (meta mode, `-p meta`) predicts ORFs on the contigs; results are filtered to retain confidently-called genes. All per-sample gene sets are concatenated (VAMB merge, minimum length 10) into one set.
3. **Gene clustering** — CD-HIT-EST clusters the concatenated genes at 95% identity / 90% length coverage (`-aS 0.90 -aL 0.90 -c 0.95`) into a non-redundant gene catalog.
4. **Abundance quantification** — reads from each sample are aligned to the gene catalog with BWA, then CoverM produces three tables: raw counts, RPKM, and TPM. CoverM uses `--min-read-percent-identity 95 --min-read-aligned-percent 75` for counts and additionally `--min-covered-fraction 20` for RPKM/TPM.
5. **Protein translation** — gene catalog is translated to proteins, then clustered with CD-HIT at 90% identity / 80% coverage (`-aS 0.80 -aL 0.80 -c 0.90`) to give a non-redundant protein catalog.
6. **Protein annotation** — proteins are split into chunks of 50 000 (Seqkit) and run through InterProScan with `-appl Pfam`. A merged annotation table is produced.
7. **Optional MetaPhlAn profiling** — if `--metaphlan_profiles` is unset (or `false`), MetaPhlAn runs to produce per-sample taxonomic profiles used by MSP analysis. If a file is supplied instead, MetaPhlAn is skipped and that file is reused.
8. **MSP analysis** — MSPminer groups co-abundant genes into metagenomic species pangenomes; results are linked to GTDB-Tk taxonomy.

## Inputs

- `--input` — samplesheet of QC'd reads.
- `--metaphlan_db` — MetaPhlAn 4 database (used when MetaPhlAn is run for MSP taxonomy).
- `--gtdb_tk_db` — GTDB-Tk reference for taxonomy. **This is not installed by [download_databases](download_databases.md)**; install GTDB-Tk separately and point this parameter at the result.

## Parameters

| Parameter              | Type            | Default      | Controls                                                                                                                      |
| ---------------------- | --------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `--input`              | path            | _(required)_ | Samplesheet of clean reads.                                                                                                   |
| `--outdir`             | path            | _(required)_ | Result directory.                                                                                                             |
| `--metaphlan_db`       | path            | —            | MetaPhlAn 4 database (used when MetaPhlAn runs fresh for MSP taxonomy).                                                       |
| `--gtdb_tk_db`         | path            | —            | GTDB-Tk reference for prokaryotic taxonomic placement of MSPs.                                                                |
| `--metaphlan_profiles` | path \| `false` | `false`      | If a path: pre-computed MetaPhlAn profiles file used as input to MSP, MetaPhlAn is skipped. If `false`: MetaPhlAn runs fresh. |

## Output

| Path (relative to `--outdir`)                                  | Content                                                                              |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `prodigal/raw/`                                                | Raw Prodigal output per sample (nucleotide FASTA, GFF).                              |
| `prodigal/`                                                    | Filtered per-sample gene sequences.                                                  |
| `<catalog dir>/merged.nr9590.fa`                               | **Non-redundant gene catalog** (CD-HIT 95% identity, 90% coverage).                  |
| `translated/`                                                  | Protein-translated gene catalog.                                                     |
| `<catalog dir>/<catalog>.prot90.faa`                           | Non-redundant protein catalog (CD-HIT 90% identity, 80% coverage).                   |
| `coverm/gene/bam/`                                             | Per-sample BWA alignments against the gene catalog.                                  |
| `coverm/gene/`                                                 | Per-sample CoverM tables (intermediate).                                             |
| `coverm/gene_abundance_count.tsv`                              | **Gene-by-sample raw read count matrix.**                                            |
| `coverm/gene_abundance_rpkm.tsv`                               | **Gene-by-sample RPKM matrix.**                                                      |
| `coverm/gene_abundance_tpm.tsv`                                | **Gene-by-sample TPM matrix.**                                                       |
| `interproscan/raw/`                                            | Per-chunk InterProScan output.                                                       |
| `interproscan/protein_catalog.tsv`                             | **InterProScan / Pfam annotations** on the protein catalog.                          |
| `interproscan/protein_catalog.FG_IPS_Pfam.tsv`                 | Functional-group annotations.                                                        |
| `metaphlan/individual_profiles/<sample>_microbial_profile.txt` | Per-sample MetaPhlAn profiles (only when MetaPhlAn runs).                            |
| `metaphlan/microbial.txt`                                      | Merged MetaPhlAn matrix (only when MetaPhlAn runs).                                  |
| `msp/`                                                         | MSPminer output: MSP definitions, pangenome sequences, abundance, MetaPhlAn linkage. |
| `pipeline_info/gene_analysis_multiqc_report.html`              | Consolidated MultiQC.                                                                |

The bolded rows are the typical analytical deliverables: the gene catalog, its annotations, the three abundance matrices, and the MSP set.

## Example

End-to-end run, letting MetaPhlAn run fresh:

```bash
nextflow run schirmer-lab/metagear -profile docker \
  --workflow gene_analysis \
  --input clean.csv \
  --outdir genes/ \
  --metaphlan_db /data/metagear/metaphlan \
  --gtdb_tk_db /data/metagear/gtdb_tk
```

Reusing a MetaPhlAn profile from a prior `microbial_profiles` run (skips MetaPhlAn):

```bash
nextflow run schirmer-lab/metagear -profile docker \
  --workflow gene_analysis \
  --input clean.csv \
  --outdir genes/ \
  --metaphlan_profiles prior_run/metaphlan/microbial.txt \
  --gtdb_tk_db /data/metagear/gtdb_tk
```

## Notes

- **Compute footprint.** Assembly and clustering dominate; expect tens to hundreds of CPU-hours per sample, depending on diversity.
- **What "non-redundant" means.** The CD-HIT-EST catalog clusters genes at ≥95% identity and ≥90% length coverage. Representatives are the longest sequence per cluster — this is your effective gene unit for downstream stats.
- **Abundance filtering.** CoverM uses `--min-read-percent-identity 95 --min-read-aligned-percent 75` for counts and additionally `--min-covered-fraction 20` for RPKM/TPM. These thresholds are deliberately strict to limit cross-mapping between closely related genes.
- **InterProScan is run with `-appl Pfam` only.** If you need other InterPro signature databases, customise the args in `conf/metagear/protein_annotation.config`.
- **MSPminer needs MetaPhlAn.** If you skip MetaPhlAn by supplying `--metaphlan_profiles <file>`, MSPminer uses that file to link MSPs to taxonomy.
- **`--contig_catalog`** appears in `workflow_definitions.json` for compatibility with older CLI invocations but is not currently wired into the `gene_analysis` workflow — assembly runs every time. To skip assembly, the right pattern today is to feed pre-assembled FASTQ-derived contigs upstream and run the pipeline from QC'd reads.

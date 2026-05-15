# qc_dna

Quality control for shotgun-metagenomic DNA reads. Produces decontaminated, adapter-trimmed paired reads that are ready for downstream analysis.

## What it does

1. **FastQC** on the raw reads — pre-QC report.
2. **Trim Galore** for adapter and quality trimming. Process arg: `--phred33 --quality 0 --stringency 5 --length 10` (light defaults; quality trimming is delegated to KneadData).
3. **Optional FASTX header rewrite** — only runs when `--fix_fastq_header true`. Use this if your reads have non-standard or Illumina-incompatible headers that confuse downstream tools.
4. **KneadData** for host decontamination. Process arg: `--trimmomatic-options 'HEADCROP:15 SLIDINGWINDOW:4:15 MINLEN:50' --reorder --remove-intermediate-output --bypass-trf`. Reads are aligned against the host genome(s) supplied via `--kneaddata_refdb` and host-matching pairs are dropped.
5. **FastQC** again on the decontaminated reads — post-QC report.
6. **Summary plot** of read counts across stages (raw → trimmed → host-removed → final).

`qc_dna` and `qc_rna` share this exact pipeline (the dispatcher in `workflows/metagear.nf` routes both via `params.workflow.startsWith("qc_")`). See [qc_rna](qc_rna.md) for the metatranscriptomic framing.

## Inputs

- `--input` — CSV samplesheet (`sample,fastq_1,fastq_2`). See [usage.md](../usage.md) for the contract.
- `--kneaddata_refdb` — one or more Bowtie2-indexed host references for decontamination. Set this to the database installed by [download_databases](download_databases.md).

## Parameters

| Parameter            | Type    | Default      | Controls                                                                                                                           |
| -------------------- | ------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| `--input`            | path    | _(required)_ | Samplesheet of raw FASTQ pairs.                                                                                                    |
| `--outdir`           | path    | _(required)_ | Where to write results.                                                                                                            |
| `--kneaddata_refdb`  | array   | `[""]`       | Host genome(s) for read filtering (e.g. `[/data/kneaddata/Homo_sapiens]`).                                                         |
| `--fix_fastq_header` | boolean | `false`      | Rewrite FASTQ headers via FASTX before KneadData. Enable if your reads come from a non-standard source and KneadData rejects them. |

## Output

| Path (relative to `--outdir`)                | Content                                                                    |
| -------------------------------------------- | -------------------------------------------------------------------------- |
| `fastqc/<sample>_fastqc.{html,zip}`          | Pre-QC FastQC per raw FASTQ.                                               |
| `trimgalore/<sample>_*_val_{1,2}.fq.gz`      | Adapter-trimmed reads (intermediate).                                      |
| `trimgalore/<sample>_*_trimming_report.txt`  | Trim Galore log.                                                           |
| `kneaddata/<sample>_paired_{1,2}.fastq.gz`   | **Final clean reads.** Use these as input to downstream workflows.         |
| `kneaddata/<sample>_kneaddata.log`           | Per-sample KneadData log.                                                  |
| `kneaddata/stats/`                           | Per-sample read-count CSV and the QC summary plot (`qc_summary_plot.png`). |
| `fastqc/<sample>_clean_fastqc.{html,zip}`    | Post-QC FastQC per cleaned FASTQ.                                          |
| `pipeline_info/qc_dna_multiqc_report.html`   | Consolidated MultiQC report across all samples.                            |
| `pipeline_info/qc_dna_samplesheet.valid.csv` | Validated samplesheet as the pipeline saw it.                              |

The "final clean reads" row is the only output you need to keep long-term; the rest are diagnostic.

## Example

```bash
nextflow run schirmer-lab/metagear -profile docker \
  --workflow qc_dna \
  --input raw_samples.csv \
  --outdir qc/ \
  --kneaddata_refdb /data/metagear/kneaddata/Homo_sapiens
```

After this, build a `clean.csv` pointing at the `kneaddata/<sample>_paired_{1,2}.fastq.gz` files and feed it to `microbial_profiles` or `gene_analysis`.

## Notes

- **Trim Galore is intentionally light** (`--quality 0`); the heavier quality trimming is done by KneadData's internal Trimmomatic step with `SLIDINGWINDOW:4:15 MINLEN:50`. Don't tighten Trim Galore unless you know why.
- **`HEADCROP:15` removes the first 15 bases** of each read inside KneadData. This is appropriate for typical Illumina libraries with biased initial bases; if your library doesn't have that artifact you may want to drop it via a custom config.
- **Multiple host references.** Pass an array to `--kneaddata_refdb` to filter against several genomes in one pass (e.g. human + mouse for xenograft samples).
- **Single-end input** is accepted — leave `fastq_2` empty in the samplesheet.
- **`--fix_fastq_header`** is off by default. Turn it on only if KneadData errors out on header parsing.

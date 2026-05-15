# qc_rna

Quality control intended for metatranscriptomic (RNA-seq) reads. Mechanically the same pipeline as [qc_dna](qc_dna.md) — same modules, same defaults — but documented separately because the inputs and the appropriate `--kneaddata_refdb` choice differ.

## What it does

Identical to [qc_dna](qc_dna.md):

1. FastQC raw reads.
2. Trim Galore adapter/quality trimming.
3. Optional FASTX header fix (`--fix_fastq_header`).
4. KneadData decontamination using the references in `--kneaddata_refdb`.
5. FastQC post-QC reads.
6. Summary plot of read counts across stages.

The dispatcher in `workflows/metagear.nf` routes both `qc_dna` and `qc_rna` through the same `QUALITY_CONTROL` subworkflow (`params.workflow.startsWith("qc_")`). The difference is which references you pass:

- For DNA: a host genome (e.g. `Homo_sapiens`).
- For RNA: a host genome **plus an rRNA reference** (e.g. SILVA), so ribosomal reads are removed alongside host reads. Both go into the `--kneaddata_refdb` array; KneadData runs against each in turn.

If you don't pass an rRNA reference, the workflow still runs but rRNA reads will dominate the output.

## Inputs

- `--input` — CSV samplesheet (`sample,fastq_1,fastq_2`). See [usage.md](../usage.md).
- `--kneaddata_refdb` — array of Bowtie2-indexed references. For metatranscriptomics, include at least one host genome **and** an rRNA reference.

## Parameters

| Parameter | Type | Default | Controls |
|---|---|---|---|
| `--input` | path | _(required)_ | Samplesheet of raw FASTQ pairs. |
| `--outdir` | path | _(required)_ | Where to write results. |
| `--kneaddata_refdb` | array | `[""]` | Host genome + rRNA reference(s) for filtering. |
| `--fix_fastq_header` | boolean | `false` | Rewrite FASTQ headers via FASTX before KneadData. |

## Output

Same shape as `qc_dna`. Output paths are identical; only the MultiQC report and validated samplesheet are prefixed with `qc_rna_`:

| Path (relative to `--outdir`) | Content |
|---|---|
| `kneaddata/<sample>_paired_{1,2}.fastq.gz` | **Final clean reads** — host- and rRNA-depleted. |
| `kneaddata/<sample>_kneaddata.log` | Per-sample log (one block per reference). |
| `kneaddata/stats/qc_summary_plot.png` | Visual breakdown of read losses by step. |
| `fastqc/`, `trimgalore/` | Intermediate FastQC and Trim Galore output. |
| `pipeline_info/qc_rna_multiqc_report.html` | Consolidated MultiQC report. |

Expect the final read count to drop substantially compared with DNA — that's the rRNA filtering working.

## Example

```bash
nextflow run schirmer-lab/metagear -profile docker \
  --workflow qc_rna \
  --input raw_rna_samples.csv \
  --outdir qc_rna/ \
  --kneaddata_refdb '[/data/metagear/kneaddata/Homo_sapiens,/data/metagear/kneaddata/SILVA_128_LSURef]'
```

## Notes

- **Expect heavy attrition.** A typical RNA library can be 70–90% rRNA; the cleaned output may be only a fraction of the raw input. This is by design.
- **No mRNA enrichment is performed** beyond rRNA depletion. If you have residual host mRNA you don't want, add the host transcriptome to `--kneaddata_refdb`.
- **Downstream compatibility.** The output FASTQs are interchangeable with `qc_dna` output — you can feed them into `microbial_profiles` or `gene_analysis`. Just remember that abundance from RNA reflects activity, not presence.

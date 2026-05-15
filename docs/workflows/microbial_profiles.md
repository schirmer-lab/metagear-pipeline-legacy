# microbial_profiles

Reference-based taxonomic and functional profiling of microbial communities. Use this when you want to know "which species are here and what can they do" without committing to a de novo assembly.

## What it does

1. **MetaPhlAn 4** maps reads against species-specific marker genes and produces a relative-abundance profile per sample. The pipeline always runs with `--profile_vsc`, so a parallel viral-marker profile is emitted alongside the bacterial profile.
2. **MetaPhlAn merge** combines per-sample profiles into a single abundance table across all samples.
3. **HUMAnN 3** uses the MetaPhlAn output to pick species-specific pangenomes from ChocoPhlAn, aligns reads against them, and falls back to UniRef90 for unmapped reads. Output is per-sample gene-family and pathway-abundance tables (normalized to CPM via `--units cpm`).
4. **HUMAnN merge** combines per-sample gene-family and pathway tables.

## Inputs

- `--input` — samplesheet of QC'd reads.
- `--metaphlan_db` — MetaPhlAn 4 marker database (from [download_databases](download_databases.md)).
- `--humann3_nucleo` — HUMAnN ChocoPhlAn nucleotide database.
- `--humann3_uniref90` — HUMAnN UniRef90 protein database.

## Parameters

| Parameter | Type | Default | Controls |
|---|---|---|---|
| `--input` | path | _(required)_ | Samplesheet of clean reads. |
| `--outdir` | path | _(required)_ | Result directory. |
| `--metaphlan_db` | path | — | MetaPhlAn 4 database. |
| `--humann3_nucleo` | path | — | ChocoPhlAn DB. |
| `--humann3_uniref90` | path | — | UniRef90 DB. |

## Output

| Path (relative to `--outdir`) | Content |
|---|---|
| `metaphlan/individual_profiles/<sample>_microbial_profile.txt` | Per-sample MetaPhlAn species table (relative abundance). |
| `metaphlan/individual_profiles/<sample>_viral_profile.txt` | Per-sample viral marker profile (always produced — `--profile_vsc` is on by default). |
| `metaphlan/microbial.txt` | **Species-by-sample abundance matrix** across the whole study. |
| `humann/<sample>_genefamilies.tsv` | Per-sample gene-family abundance (UniRef90 IDs, CPM-normalized). |
| `humann/<sample>_pathabundance.tsv` | Per-sample MetaCyc pathway abundance (CPM-normalized). |
| `humann/gene_families.tsv` | **Gene-family-by-sample abundance matrix.** |
| `humann/path_abundances.tsv` | **Pathway-by-sample abundance matrix.** |
| `pipeline_info/microbial_profiles_multiqc_report.html` | Consolidated MultiQC. |

The two `humann/` merged tables and the `metaphlan/microbial.txt` table are the typical inputs to downstream statistical analysis.

## Example

```bash
nextflow run schirmer-lab/metagear -profile docker \
  --workflow microbial_profiles \
  --input clean.csv \
  --outdir profiles/ \
  --metaphlan_db /data/metagear/metaphlan \
  --humann3_nucleo /data/metagear/humann/chocophlan \
  --humann3_uniref90 /data/metagear/humann/uniref90_diamond
```

## Notes

- **MetaPhlAn 4 is reference-based.** Species that are not in the marker database will not be reported, even if they are abundant in your sample. For novel-organism discovery, complement with [gene_analysis](gene_analysis.md).
- **HUMAnN memory.** HUMAnN scales with diversity; 16 GB+ per sample is typical and large/diverse samples (e.g. soil) may need much more.
- **CPM units.** `humann_renorm_table` is run with `--units cpm`. If you need RPK or relative abundance, post-process the merged tables with the HUMAnN utilities.
- **Viral profiles are produced by default** via MetaPhlAn's VSC mode. The per-sample `*_viral_profile.txt` files contain viral-marker hits; these are useful as a quick check but are reference-limited (only viruses in MetaPhlAn's marker set).

# download_databases

One-time setup of the reference databases consumed by `qc_dna`, `qc_rna`, and `microbial_profiles`. Run this once per installation (and re-run when you want to refresh a database).

## What it does

Downloads and prepares three references, sized at tens of gigabytes each:

- **KneadData host reference** — the human genome (Bowtie2 index), used for host-read decontamination during QC.
- **MetaPhlAn 4 marker database** — used for taxonomic profiling.
- **HUMAnN 3 ChocoPhlAn** (nucleotide, full) and **UniRef90** (Diamond-formatted protein) databases — used for functional profiling.

Each tool is downloaded by its native installer wrapped in a Nextflow module, then exported to the destination path you supply in the corresponding `*_db` parameter.

## Inputs

This workflow does not read a samplesheet — there is no `--input` argument. Instead, you tell it where to put each database via the path parameters below. The `metagear-tools` CLI manages these paths for you via its config; if you run Nextflow directly you must pass them yourself (typically through a `-params-file`).

## Parameters

| Parameter            | Type  | Default      | Controls                                                                                                                     |
| -------------------- | ----- | ------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| `--outdir`           | path  | _(required)_ | Output directory for execution metadata (the database files themselves land at the destinations below).                      |
| `--metaphlan_db`     | path  | —            | Destination for the MetaPhlAn 4 marker database.                                                                             |
| `--humann3_nucleo`   | path  | —            | Destination for the HUMAnN 3 ChocoPhlAn nucleotide database.                                                                 |
| `--humann3_uniref90` | path  | —            | Destination for the HUMAnN 3 UniRef90 Diamond protein database.                                                              |
| `--kneaddata_refdb`  | array | `[""]`       | Destination for the KneadData host reference. Single-element array; the pipeline writes the human-genome Bowtie2 index here. |

## Output

Each tool writes into the directory you supplied for its `*_db` parameter. The pipeline does not republish those files into `<outdir>` (the modules' `publishDir` is intentionally suppressed) — the databases stay where the installers placed them.

Typical layout (paths are whatever you passed):

```
<metaphlan_db>/                      # MetaPhlAn 4 marker DB
<humann3_nucleo>/                    # ChocoPhlAn pangenomes
<humann3_uniref90>/                  # UniRef90 Diamond DB
<kneaddata_refdb[0]>/                # Bowtie2 host genome index

<outdir>/pipeline_info/              # Nextflow execution metadata
```

## Example

```bash
nextflow run schirmer-lab/metagear -profile docker \
  --workflow download_databases \
  --outdir databases/ \
  -params-file db-paths.yaml
```

`db-paths.yaml`:

```yaml
metaphlan_db: /data/metagear/metaphlan
humann3_nucleo: /data/metagear/humann/chocophlan
humann3_uniref90: /data/metagear/humann/uniref90_diamond
kneaddata_refdb: [/data/metagear/kneaddata/Homo_sapiens]
```

## Notes

- **Disk and time.** Total downloads are around 50–70 GB and can take several hours, mostly bound by external download speed.
- **GTDB-Tk is not downloaded by this workflow.** `gene_analysis` needs a GTDB-Tk database for its MSP taxonomic assignment step — install it separately and point `--gtdb_tk_db` at the resulting directory. See the [GTDB-Tk installation docs](https://ecogenomics.github.io/GTDBTk/installing/index.html).
- **HUMAnN config.** The HUMAnN installer is invoked with `--update-config no`; HUMAnN's global config is not modified on the host.
- **Wrapper convenience.** [`metagear-tools`](https://github.com/schirmer-lab/metagear-tools) ships templates that wire all of these paths into a single user config — recommended over hand-writing the params file above.

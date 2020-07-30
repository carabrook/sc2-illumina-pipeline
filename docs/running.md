# Running the pipeline
<!-- MarkdownTOC -->

- [Introduction](#introduction)
- [Running the pipeline](#running-the-pipeline)
	- [Updating the pipeline](#updating-the-pipeline)
- [Main arguments](#main-arguments)
	- [`-profile`](#-profile)
	- [`--reads`](#--reads)
	- [`--primers`](#--primers)
	- [`--ref`](#--ref)
	- [`--ref_host`](#--ref_host)
	- [`--kraken2_db`](#--kraken2_db)
	- [`--exclude_samples`](#--exclude_samples)
	- [`--single_end`](#--single_end)
	- [`--skip_trim_adapters`](#--skip_trim_adapters)
	- [`--prefilter_host_reads`](#--prefilter_host_reads)
	- [`--maxNs`](#--maxns)
	- [`--minLength`](#--minlength)
	- [`--no_reads_quast`](#--no_reads_quast)
	- [`--ercc_fasta`](#--ercc_fasta)
	- [`--save_sars2_filtered_reads`](#--save_sars2_filtered_reads)
	- [`--outdir`](#--outdir)
- [Primer trimming](#primer-trimming)
	- [`--samQualThreshold`](#--samqualthreshold)
- [Consensus calling](#consensus-calling)
	- [`--ivarQualThreshold`](#--ivarqualthreshold)
	- [`--ivarFreqThreshold`](#--ivarfreqthreshold)
	- [`--minDepth`](#--mindepth)
- [AWS Batch specific parameters](#aws-batch-specific-parameters)
	- [`--awsqueue`](#--awsqueue)
	- [`--awsregion`](#--awsregion)
- [Variant calling](#variant-calling)
	- [`--bcftoolsCallTheta`](#--bcftoolscalltheta)
	- [`--joint_variant_calling`](#--joint_variant_calling)
- [Other command line parameters](#other-command-line-parameters)
	- [`-name`](#-name)
	- [`-resume`](#-resume)
	- [`-c`](#-c)
	- [`--max_memory`](#--max_memory)
	- [`--max_time`](#--max_time)
	- [`--max_cpus`](#--max_cpus)
	- [`--multiqc_config`](#--multiqc_config)

<!-- /MarkdownTOC -->

## Introduction

Nextflow handles job submissions on SLURM or other environments, and supervises running the jobs. Thus the Nextflow process must run until the pipeline is finished. We recommend that you put the process running in the background through `screen / tmux` or similar tool. Alternatively you can run nextflow within a cluster job submitted your job scheduler.

It is recommended to limit the Nextflow Java virtual machines memory. We recommend adding the following line to your environment (typically in `~/.bashrc` or `~./bash_profile`):

```bash
NXF_OPTS='-Xms1g -Xmx4g'
```

## Running the pipeline

For generating consensus genomes from reads:

```{sh}
nextflow run czbiohub/sc2-illumina-pipeline -profile artic,docker \
    --reads '[s3://]path/to/reads/*_R{1,2}_001.fastq.gz*' \
    --kraken2_db '[s3://]path/to/kraken2db' \
    --outdir '[s3://]path/to/outdir'
```

The kraken2db can be downloaded from https://genexa.ch/sars2-bioinformatics-resources/.

### Updating the pipeline

The above command will automatically pull the code from GitHub and store it as a cached version. This cached version (found in `.nextflow/assets/`) will be used for subsequent runs in available. To make sure you are running the latest version of the pipeline, update the cached version:

```bash
nextflow pull czbiohub/sc2-illumina-pipeline
```

## Main arguments

### `-profile`

Use this parameter to choose a configuration profile. Profiles can give configuration presets for different compute environments. Note that multiple profiles can be loaded, for example: `-profile test,docker` - the order of arguments is important!

If `-profile` is not specified at all the pipeline will be run locally and expects all software to be install and available on the `PATH`.

* `conda`
  * a generic configuration profile to be used with [conda](https://conda.io/docs/)
  * pulls most software from [Bioconda](https://bioconda.github.io/)
* `docker`
  * A generic configuration profile to be used with [Docker](http://docker.com/)
  * Pulls software from dockerhub: [`czbiohub/sc2-msspe`](http://hub.docker.com/r/czbiohub/sc2-msspe/)
* `singularity`
  * A generic configuration profile to be used with [Singularity](http://singularity.lbl.gov/)
  * Pulls software from DockerHub: [`czbiohub/sc2-msspe`](http://hub.docker.com/r/czbiohub/sc2-msspe/)
* `test`
  * A profile with a complete configuration for automated testing
* `artic`
  * A profile to run the pipeline for ARTIC V3 data.
* `msspe`
  * A profile to run the pipeline for MSSPE data.

### `--reads`

Use this to specify the location of your input SARS-CoV-2 read files. For example:

```bash
--reads 'data/*_{R1,R2}_001.fastq.gz'
```

Please note the following requirements:

1. The path must be enclosed in quotes
2. The path must have at least one `*` wildcard character

The path can pull directly from S3 if the system has the appropriate credentials.

### `--primers`

Use this to specify the BED file with the enrichment/amplification primers used in the library prep. The repository comes with two sets of primers corresponding to the MSSPE ([`SARS-COV-2_spikePrimers.bed`](../data/SARS-COV-2_spikePrimers.bed)) and ARTIC V3 ([`nCoV-2019.bed`](../data/nCoV-2019.bed)) protocols.

```bash
--primers data/nCoV-2019.bed
```

### `--ref`

Use this to specify the path to the SARS-CoV-2 reference genome, which will be used for alignment and downstream consensus-calling and variant-calling.

```bash
--ref data/MN908947.3.fa
```

By default, the pipeline will use the genome [`data/MN908947.3.fa`](../data/MN908947.3.fa), corresponding to strain Wuhan-Hu-1.

### `--ref_host`

Use this to specify the path to the host reference genome.

```bash
--ref_host data/human_chr1.fa
```

By default, the human genome is pulled from http://hgdownload.cse.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz.

### `--kraken2_db`

Use this to specify the path to the folder containing the Kraken2 database. This is required to filter the reads to only SARS-CoV-2 reads.

```bash
--kraken2_db kraken2_h+v_20200319/
```

### `--exclude_samples`

Use this with a comma-separated list of samples to exclude by name.

```bash
--exclude_samples 'Undetermined_S0,sample1'
```

You can exclude certain samples in the reads argument that do not need to be assembled.

### `--single_end`

Use this to specify that the reads are single-end.

### `--skip_trim_adapters`

Use this to skip trimming with Trim Galore. This is useful if you are running on a set of reads that have already been adapter-trimmed. This will __not__ skip primer trimming.

### `--prefilter_host_reads`

Prefilter host reads, and save the resulting fastqs. This is useful when you need fastqs that have been filtered of human reads, but still contain reads from other pathogens besides SARS-CoV-2. By default, this is disabled when `-profile artic`, but enabled when `-profile msspe`.

### `--maxNs`

Specify an integer for maximum N bases allowed for a consensus genome to pass filtering.

```bash
--maxNs 1000
```

By default, this is set to `100`.

### `--minLength`

Specify an integer for the minimum non-ambiguous (ACTG) bases needed for a consensus genome to pass filtering.

```bash
--minLength 29000
```

By default, this is set to `25000`.

### `--no_reads_quast`

Run QUAST without aligning reads.

### `--ercc_fasta`

Specify the path to the FASTA file of ERCC sequences. This is necessary to quantify ERCC reads. This is included in the repo and will default to [`data/ercc_sequences.fasta`](../data/ercc_sequences.fasta).

### `--save_sars2_filtered_reads`

Use this to publish SARS-CoV-2 reads to `filtered-sars2-reads` after Kraken2 filtering. This is false by default.

### `--outdir`

Specify the output directory. This is required and can be an S3 path given the system has the appropriate credentials.

```bash
--outdir results/
```

or

```bash
--outdir s3://bucket/results/
```
## Primer trimming

### `--samQualThreshold`

Minimum quality score for a read to be included in the alignment. This is `20` by default.

## Consensus calling

The defaults for these parameters normally do not need to be adjusted.

### `--ivarQualThreshold`

Minimum alignment quality for a read to contribute to an allele call using `ivar consensus`. This is set to `20` by default.

### `--ivarFreqThreshold`

Minimum allele frequency to call a consensus at a position using `ivar consensus`. This is set to `0.9` by default, so the dominant allele must have 90% frequency to be called. Otherwise, an ambiguous base (IUPAC codes) will be called at that position.

### `--minDepth`

Minimum depth at a position for an allele to be called. This is set to `10` by default, so if less than 10 reads align to a position, `N` will be called at that position.

## Variant calling

### `--bcftoolsCallTheta`

Mutation rate for `bcftools call`. This is .0006 by default.

### `--joint_variant_calling`

Create a joint VCF. This is off by default since it takes a long time.

## Other command line parameters

### `-name`
Name for the pipeline run. If not specified, Nextflow will automatically generate a random mnemonic.

**NB:** Single hyphen (core Nextflow option)

### `-resume`
Specify this when restarting a pipeline. Nextflow will used cached results from any pipeline steps where the inputs are the same, continuing from where it got to previously.

You can also supply a run name to resume a specific run: `-resume [run-name]`. Use the `nextflow log` command to show previous run names.

**NB:** Single hyphen (core Nextflow option)

### `-c`
Specify the path to a specific config file (this is a core NextFlow command).

**NB:** Single hyphen (core Nextflow option)

Note - you can use this to override pipeline defaults.

### `--max_memory`
Use to set a top-limit for the default memory requirement for each process.
Should be a string in the format integer-unit. eg. `--max_memory '8.GB'`

### `--max_time`
Use to set a top-limit for the default time requirement for each process.
Should be a string in the format integer-unit. eg. `--max_time '2.h'`

### `--max_cpus`
Use to set a top-limit for the default CPU requirement for each process.
Should be a string in the format integer-unit. eg. `--max_cpus 1`

### `--multiqc_config`
Specify a path to a custom MultiQC configuration file.


-----------

Documentation adapted from `nf-core`(https://github.com/nf-core/tools).

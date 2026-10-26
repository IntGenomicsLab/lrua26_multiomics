# Fiber-seq hands-on: from HPRC CRAM to m6A, nucleosomes and QC (chr20, GM12878)

This is a short tutorial that aims to quickly demonstrate how to process and visualise Fiber-seq data. We take a public PacBio Fiber-seq dataset (GM12878, HPRC v2), cut it down to
chromosome 20 at ~20X coverage, and runs the core [fibertools-rs](https://fiberseq.github.io/fibertools/fibertools.html)
steps: m6A calling, nucleosome calling and QC.

## Overview

| Step | Tool | Output |
|------|------|--------|
| 1. Download the chr20 ~20X subset | curl, samtools | `GM12878_PM00001.chr20.20x.bam` (+ `.bai`) |
| 2. Strip existing calls | samtools | `GM12878_PM00001.chr20.20x.strip.bam` |
| 3. Predict m6A (skipped here) | `ft predict-m6a` | `...strip.m6a.bam` |
| 4. Call nucleosomes | `ft add-nucleosomes` | `...strip.nuc.bam` (+ `.bai`) |
| 5a. QC | `ft qc` | `sample_qc.tab` |
| 5b. QC | `fibertools_qc_check.ipynb` | visualisations |
| 6. FIRE | `ft fire`| `...strip.nuc.fire.bam` |

## Requirements

- [conda](https://docs.conda.io/) (we recommend [Miniforge](https://github.com/conda-forge/miniforge))or [pixi](https://pixi.prefix.dev/latest/), on Linux or macOS
- The conda environment in [environment.yml](environment.yml), which contains everything used in this tutorial:
  [SAMtools](http://www.htslib.org/) 1.24, [fibertools-rs](https://github.com/fiberseq/fibertools-rs) 0.13.1,
  and Python + JupyterLab for the QC notebook.

## 0. Setup

Clone this repository 

```bash
git clone https://github.com/IntGenomicsLab/lrua26_multiomics.git
cd lrua26_multiomics
```

Next, create the environment using [conda](#01-create-and-start-conda-environment) or alternatively [pixi](#02-create-and-start-pixi-environment)
## 0.1. Create and start conda environment

Once you have conda installed, create the environment from the `environment.yml` file:

```bash
conda env create -f environment.yml
```

Activate it every time you start working

```bash
conda activate fiberseq-tutorial
```

## 0.2. Create and start pixi environment

Alternatively to using conda, you can use [pixi](https://pixi.prefix.dev/latest/) to create and manage the environment.

Create the environment from the `pixi.lock` (or `pixi.toml`) file:

```bash
pixi install
```

Start a shell in the environment:

```bash
pixi shell
```

## 0.3. Set up the working directory

Finally, set up the working directory and a few variables. Adjust `WORKDIR` and `THREADS` to your own system - this runs fine on a laptop, but is faster on an [HPC](https://en.wikipedia.org/wiki/High-performance_computing) cluster.

```bash
WORKDIR=results
THREADS=4
PREFIX=GM12878_PM00001.chr20
DATA_URL=https://zenodo.org/records/23596422/files

mkdir -p $WORKDIR
cd $WORKDIR
```

## 1. Download the data

We start from a public PacBio Fiber-seq dataset of GM12878 from HPRC v2. The full file is very large, so for demonstration purposes we provide a subset: chromosome 20 only, downsampled from ~120X to ~20X coverage (~1.2 GB). See [How the subset was made](#how-the-subset-was-made) if you want to recreate it.

```bash
curl -L -O $DATA_URL/$PREFIX.20x.bam
```

Index the BAM and check the coverage (about 20X):

```bash
samtools index $PREFIX.20x.bam
samtools coverage $PREFIX.20x.bam
```

## 2. Remove existing nucleosome and accessibility calls

The HPRC BAMs already contain fibertools annotations. For demonstration purposes, we remove the nucleosome
(`ns`, `nl`) and MTase sensitive patches (`as`, `al`) tags so we can recompute them ourselves.

```bash
samtools view -b -x ns,nl,as,al -@ $THREADS \
  -o $PREFIX.20x.strip.bam $PREFIX.20x.bam
```

## 3. Predict m6A (skip for this tutorial)

> [!NOTE]
> This step does not work on the file we use in this demonstration: `ft predict-m6a` predicts m6A from the HiFi kinetics (`fi`, `fp`, `ri`, `rp` tags), and these are no longer present in the HPRC BAM. If you run it, you will see many warnings (`More than 5% (100.00%) of reads were not predicted on. Are HiFi kinetics missing from this file?`), and the existing m6A calls are passed through unchanged. So **skip this step** and continue with step 4 - it is shown here because you may need to run it on your own data. This step is also not needed for ONT data as Dorado modified basecalling does this, and for any new PacBio data, as m6A are called internally by Jasmine.

```bash
ft predict-m6a -t $THREADS \
  $PREFIX.20x.strip.bam \
  $PREFIX.20x.strip.m6a.bam
```

## 4. Call nucleosomes and index

```bash
ft add-nucleosomes -t $THREADS \
  $PREFIX.20x.strip.bam \
  $PREFIX.20x.strip.nuc.bam

samtools index -@ $THREADS $PREFIX.20x.strip.nuc.bam
```

## 5. Quality control

First and foremost, you sequencing has to be good (think coverage, read N50, Q30..). Once that is checked, we go and QC our Fiber-seq data. 

`--acf` adds the m6A autocorrelation, which shows the nucleosome repeat pattern - very imporant for QC checking.

`--acf-max-lag 2000` - allows us to get longer autocorrelation lag - very helpful for some QC metric plotting and calculation

```bash
ft qc --acf --acf-max-lag 2000 $PREFIX.20x.strip.nuc.bam > sample_qc.tab
```

We will now plot some of the outputs from `sample_qc.tab`, but you may want to check out following repository for Fibertools comprehensive report: https://github.com/fiberseq/fiberseq-qc. 

### QC plotting

`sample_qc.tab` holds a lot of information, and to make some sense of it, we will plot it. You can navigate and check out [fibertools_qc_check.ipynb](fibertools_qc_check.ipynb). 

The notebook runs in the same conda/pixi environment. From the repository folder, start JupyterLab:

```bash
jupyter lab fibertools_qc_check.ipynb
```

Set `QC_FILE` in the notebook to the path of your `sample_qc.tab` (e.g. `/path/to/workdir/sample_qc.tab`) and run all cells.

Focus on the methylation rate, the sizes of nucleosomes and methylation sensitive patches (MSPs), and autocorrelation curve. 

## IGV visualisation
Important part is data visualisation. Install a recent [IGV](https://igv.org/doc/desktop/#DownloadPage/) version and visualise your bam file. Make sure you index it first. 

To look at modified bases, right click on the left panel with bam file name, select `color alignments by` and then `base modifications (6mA)` - or 5mC.

We recommend to check
-  modification patterns at TSS of some highly expressed genes - you should see high accessibility, 
- imprinted regions - you should see haplotype specific methylation patterns
- CTCF sites (you can load ENCODE ChIP data directly from URL)

## FIRE elements

FIRE stands for Fiber-seq Inferred Regulatory Elements and you can think of them as a Fiber-seq equivalent of peaks from ATAC-seq. 

To call FIRE elements and to create BigWig tracks, you need whole genome data. Since we are only working with one chromosome here, we will run `ft fire` to annotate our reads with fire calls. 

```bash
ft fire -t $THREADS \
  $PREFIX.20x.strip.nuc.bam $PREFIX.20x.strip.nuc.fire.bam
```

To see how to install the FIRE Snakemake pipeline environment and how to run the full pipeline, you can follow the tutorial here: https://fiberseq.github.io/fire/run.html

Luckily for us, FIRE developers share the results for the GM12878 cell line we are looking as at the moment, so let's visualise the pipeline outputs! 

Download all the FIRE elements, and haplotype specific FIRE calls for GM12878:

```bash
FIRE_URL=https://s3.kopah.uw.edu/stergachis/public/FIRE/broadly-consented/GM12878
curl -L -O $FIRE_URL/GM12878-fire-v0.1-peaks.bed.gz
curl -L -O $FIRE_URL/GM12878-fire-v0.1-hap-differences.bed.gz
```

There are multiple ways to visualise FIRE pipeline outputs

1. **IGV** <br>
You load `GM12878-fire-v0.1-hap-differences.bed.gz` and `GM12878-fire-v0.1-hap-differences.bed.gz` to IGV together with your CRAM or BAM file. This will allow you to observe FIRE regions visually

2. **UCSC tracks**
As output from FIRE pipeline, you also get UCSC track that you can load to UCSC via My data > Track hubs. 
More info on that here: https://github.com/fiberseq/UCSC-Fiber-seq-hub 

## Full script

Run inside the activated `fiberseq-tutorial` environment.

```bash
#!/usr/bin/env bash
set -euo pipefail

WORKDIR=results
THREADS=4
PREFIX=GM12878_PM00001.chr20
DATA_URL=https://zenodo.org/records/23596422/files
FIRE_URL=https://s3.kopah.uw.edu/stergachis/public/FIRE/broadly-consented/GM12878

mkdir -p $WORKDIR
cd $WORKDIR

curl -L -O $DATA_URL/$PREFIX.20x.bam
samtools index $PREFIX.20x.bam
samtools coverage $PREFIX.20x.bam

samtools view -b -x ns,nl,as,al -@ $THREADS \
  -o $PREFIX.20x.strip.bam $PREFIX.20x.bam
# step 3 (ft predict-m6a) is skipped: this file has no HiFi kinetics
ft add-nucleosomes -t $THREADS $PREFIX.20x.strip.bam $PREFIX.20x.strip.nuc.bam
samtools index -@ $THREADS $PREFIX.20x.strip.nuc.bam
ft qc --acf --acf-max-lag 2000 $PREFIX.20x.strip.nuc.bam > sample_qc.tab
ft fire -t $THREADS $PREFIX.20x.strip.nuc.bam $PREFIX.20x.strip.nuc.fire.bam

curl -L -O $FIRE_URL/GM12878-fire-v0.1-peaks.bed.gz
curl -L -O $FIRE_URL/GM12878-fire-v0.1-hap-differences.bed.gz
```

## How the subset was made

You do not need to run this - it is how the chr20 ~20X file in step 1 was created.
The HPRC v2 Fiber-seq CRAMs are public, so no credentials are needed (`--no-sign-request`).
This needs [aws-cli](https://aws.amazon.com/cli/), which is not part of the tutorial environment
(install it with e.g. `conda install -c conda-forge awscli`).

Download the CRAM and its index:

```bash
aws s3 cp --no-sign-request --endpoint-url https://s3.kopah.uw.edu \
  's3://stergachis/public/HPRCv2/FIRE-bams/GM12878_PM00001_PacBio.shared.ref.cram' .
aws s3 cp --no-sign-request --endpoint-url https://s3.kopah.uw.edu \
  's3://stergachis/public/HPRCv2/FIRE-bams/GM12878_PM00001_PacBio.shared.ref.cram.crai' .
```

Extract chromosome 20 and check the starting coverage (about 120X):

```bash
samtools view -b -@ $THREADS -o $PREFIX.bam GM12878_PM00001_PacBio.shared.ref.cram chr20
samtools coverage $PREFIX.bam
```

Subsample 16% of reads and index. In `-s 42.16`, the integer part (`42`) is the random seed and the
fractional part (`.16`) is the fraction of reads kept, so ~120X × 0.16 ≈ 20X.

```bash
samtools view -b -@ $THREADS -s 42.16 -o $PREFIX.20x.bam $PREFIX.bam
samtools index $PREFIX.20x.bam
```

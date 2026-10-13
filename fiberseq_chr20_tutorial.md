# Fiber-seq hands-on: from HPRC CRAM to m6A, nucleosomes and QC (chr20, GM12878)

This is a short tutorial that aims to quickly demonstrate how to process and visualise Fiber-seq data. WE take a public PacBio Fiber-seq dataset (GM12878, HPRC v2), cuts it down to
chromosome 20 at ~20X coverage, and runs the core [fibertools-rs](https://fiberseq.github.io/fibertools/fibertools.html)
steps: m6A calling, nucleosome calling and QC.

## Overview

| Step | Tool | Output |
|------|------|--------|
| 1. Download CRAM | aws-cli | `GM12878_PM00001_PacBio.shared.ref.cram` |
| 2. Extract chr20 | samtools | `GM12878_PM00001.chr20.bam` |
| 3. Downsample to ~20X | samtools | `GM12878_PM00001.chr20.20x.bam` |
| 4. Strip existing calls | samtools | `GM12878_PM00001.chr20.20x.strip.bam` |
| 5. Predict m6A | `ft predict-m6a` | `...strip.m6a.bam` |
| 6. Call nucleosomes | `ft add-nucleosomes` | `...strip.m6a.nuc.bam` (+ `.bai`) |
| 7a. QC | `ft qc` | `qc.tab` |
| 7b. QC | `fibertools_qc_check.ipynb` | visualisationsß |
| 8. FIRE | `ft fire`| `...strip.m6a.nuc.fire.bam` |

## Requirements

- [Apptainer](https://apptainer.org/) (or Singularity)
- Container images:
  - aws-cli: `apptainer pull aws-cli_latest.sif docker://amazon/aws-cli:latest`
  - [SAMtools](http://www.htslib.org/) 1.24:
    `apptainer pull samtools_1.24--h9dcdb79_1.sif docker://quay.io/biocontainers/samtools:1.24--h9dcdb79_1`
  - [fibertools-rs](https://github.com/fiberseq/fibertools-rs) 0.13.1:
    `apptainer pull fibertools-rs_0.13.1--h270025b_0.sif docker://quay.io/biocontainers/fibertools-rs:0.13.1--h270025b_0`

## 0. Setup

Adjust these paths to your own system. Here, we assume you are working on [HPC](https://en.wikipedia.org/wiki/High-performance_computing) cluster.

```bash
WORKDIR=/path/to/workdir
AWS_SIF=aws-cli_latest.sif
SAM_SIF=samtools_1.24--h9dcdb79_1.sif
FT_SIF=fibertools-rs_0.13.1--h270025b_0.sif
THREADS=20

cd $WORKDIR
```

## 1. Download the data

The HPRC v2 Fiber-seq CRAMs are public, so no credentials are needed (`--no-sign-request`).

```bash
apptainer run -B $WORKDIR $AWS_SIF \
  s3 cp --no-sign-request --endpoint-url https://s3.kopah.uw.edu \
  's3://stergachis/public/HPRCv2/FIRE-bams/GM12878_PM00001_PacBio.shared.ref.cram' .
```

## 2. Extract chromosome 20

As the file is very large, for demonstration purposes, we will look at a single chromosome only.

```bash
apptainer run -B $WORKDIR $SAM_SIF \
  samtools view -b -o GM12878_PM00001.chr20.bam GM12878_PM00001_PacBio.shared.ref.cram chr20
```

## 3. Downsample to ~20X

Check the starting coverage (about 120X):

```bash
apptainer run -B $WORKDIR $SAM_SIF \
  samtools coverage GM12878_PM00001.chr20.bam
```

Subsample 16% of reads. In `-s 42.16`, the integer part (`42`) is the random seed and the
fractional part (`.16`) is the fraction of reads kept, so ~120X × 0.16 ≈ 20X.

```bash
apptainer run -B $WORKDIR $SAM_SIF \
  samtools view -@ $THREADS -s 42.16 -o GM12878_PM00001.chr20.20x.bam GM12878_PM00001.chr20.bam
```

## 4. Remove existing nucleosome and accessibility calls

The HPRC BAMs already contain fibertools annotations. For demonstration purposes, we remove the nucleosome
(`ns`, `nl`) and MTase sensitive patches (`as`, `al`) tags so we can recompute them ourselves.

```bash
apptainer run -B $WORKDIR $SAM_SIF \
  samtools view -b -x ns,nl,as,al -@ $THREADS \
  -o GM12878_PM00001.chr20.20x.strip.bam GM12878_PM00001.chr20.20x.bam
```

## 5. Predict m6A (Optional)

> [!NOTE]
> This step is not needed for the files we use in this demonstration, but you may need to run this on your data. This step is not be needed for ONT data as Dorado modified basecalling does this, and for any new PacBio data, as m6A are called internally by Jasmine.

```bash
apptainer run -B $WORKDIR $FT_SIF \
  ft predict-m6a -t $THREADS \
  GM12878_PM00001.chr20.20x.strip.bam \
  GM12878_PM00001.chr20.20x.strip.m6a.bam
```

## 6. Call nucleosomes and index

```bash
apptainer run -B $WORKDIR $FT_SIF \
  ft add-nucleosomes -t $THREADS \
  GM12878_PM00001.chr20.20x.strip.m6a.bam \
  GM12878_PM00001.chr20.20x.strip.m6a.nuc.bam

apptainer run -B $WORKDIR $SAM_SIF \
  samtools index -@ $THREADS GM12878_PM00001.chr20.20x.strip.m6a.nuc.bam
```

## 7. Quality control

First and foremost, you sequencing has to be good (think coverage, read N50, Q30..). Once that is checked, we go and QC our Fiber-seq data. 

`--acf` adds the m6A autocorrelation, which shows the nucleosome repeat pattern - very imporant for QC checking.

`--acf-max-lag 2000` - allows us to get longer autocorrelation lag - very helpful for some QC metric plotting and calculation

```bash
apptainer run -B $WORKDIR $FT_SIF \
  ft qc --acf --acf-max-lag 2000 GM12878_PM00001.chr20.20x.strip.m6a.nuc.bam > sample_qc.tab
```

We will now plot some of the outputs from `sample_qc.tab`, but you may want to check out following repository for Fibertools comprehensive report: https://github.com/fiberseq/fiberseq-qc. 

### QC plotting

`sample_qc.tab` holds a lot of information, and to make some sense of it, we will plot it. You can navigate and check out [fibertools_qc_check.ipynb](fibertools_qc_check.ipynb). 

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

```
apptainer run -B $WORKDIR $FT_SIF ft fire -t $THREADS \
  $PREFIX.20x.strip.m6a.nuc.bam $PREFIX.20x.strip.m6a.nuc.fire.bam
```

To see how to install the FIRE Snakemake pipeline environment and how to run the full pipeline, you can follow the tutorial here: https://fiberseq.github.io/fire/run.html

Luckily for us, FIRE developers share the results for the GM12878 cell line we are looking as at the moment, so let's visualise the pipeline outputs! 

Download all the FIRE elements, and haplotype specific FIRE calls for GM12878:

```
apptainer run -B $WORKDIR $AWS_SIF s3  cp  --no-sign-request --endpoint-url https://s3.kopah.uw.edu 's3://stergachis/public/FIRE/broadly-consented/GM12878/GM12878-fire-v0.1-peaks.bed.gz' GM12878-fire-v0.1-peaks.bed.gz

apptainer run -B $WORKDIR $AWS_SIF s3  cp  --no-sign-request --endpoint-url https://s3.kopah.uw.edu 's3://stergachis/public/FIRE/broadly-consented/GM12878/GM12878-fire-v0.1-hap-differences.bed.gz' GM12878-fire-v0.1-hap-differences.bed.gz
```

There are multiple ways to visualise FIRE pipeline outputs

1. **IGV** <br>
You load `GM12878-fire-v0.1-hap-differences.bed.gz` and `GM12878-fire-v0.1-hap-differences.bed.gz` to IGV together with your CRAM or BAM file. This will allow you to observe FIRE regions visually

2. **UCSC tracks**
As output from FIRE pipeline, you also get UCSC track that you can load to UCSC via My data > Track hubs. 
More info on that here: https://github.com/fiberseq/UCSC-Fiber-seq-hub 

## Full script

```bash
#!/usr/bin/env bash
set -euo pipefail

WORKDIR=/path/to/workdir
AWS_SIF=aws-cli_latest.sif
SAM_SIF=samtools_1.24--h9dcdb79_1.sif
FT_SIF=fibertools-rs_0.13.1--h270025b_0.sif
THREADS=20
PREFIX=GM12878_PM00001.chr20

cd $WORKDIR

apptainer run -B $WORKDIR $AWS_SIF \
  s3 cp --no-sign-request --endpoint-url https://s3.kopah.uw.edu \
  's3://stergachis/public/HPRCv2/FIRE-bams/GM12878_PM00001_PacBio.shared.ref.cram' .

apptainer run -B $WORKDIR $SAM_SIF samtools view -b -o $PREFIX.bam \
  GM12878_PM00001_PacBio.shared.ref.cram chr20
apptainer run -B $WORKDIR $SAM_SIF samtools coverage $PREFIX.bam
apptainer run -B $WORKDIR $SAM_SIF samtools view -@ $THREADS -s 42.16 \
  -o $PREFIX.20x.bam $PREFIX.bam
apptainer run -B $WORKDIR $SAM_SIF samtools view -b -x ns,nl,as,al -@ $THREADS \
  -o $PREFIX.20x.strip.bam $PREFIX.20x.bam
apptainer run -B $WORKDIR $FT_SIF ft predict-m6a -t $THREADS \
  $PREFIX.20x.strip.bam $PREFIX.20x.strip.m6a.bam
apptainer run -B $WORKDIR $FT_SIF ft add-nucleosomes -t $THREADS \
  $PREFIX.20x.strip.m6a.bam $PREFIX.20x.strip.m6a.nuc.bam
apptainer run -B $WORKDIR $SAM_SIF samtools index -@ $THREADS $PREFIX.20x.strip.m6a.nuc.bam
apptainer run -B $WORKDIR $FT_SIF ft qc --acf $PREFIX.20x.strip.m6a.nuc.bam > sample_qc.tab
apptainer run -B $WORKDIR $FT_SIF ft fire -t $THREADS \
  $PREFIX.20x.strip.m6a.nuc.bam $PREFIX.20x.strip.m6a.nuc.fire.bam

apptainer run -B $WORKDIR $AWS_SIF s3  cp  --no-sign-request --endpoint-url https://s3.kopah.uw.edu 's3://stergachis/public/FIRE/broadly-consented/GM12878/GM12878-fire-v0.1-peaks.bed.gz' GM12878-fire-v0.1-peaks.bed.gz
apptainer run -B $WORKDIR $AWS_SIF s3  cp  --no-sign-request --endpoint-url https://s3.kopah.uw.edu 's3://stergachis/public/FIRE/broadly-consented/GM12878/GM12878-fire-v0.1-hap-differences.bed.gz' GM12878-fire-v0.1-hap-differences.bed.gz
```
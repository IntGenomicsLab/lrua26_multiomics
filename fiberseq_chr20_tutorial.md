# Fiber-seq hands-on: from HPRC CRAM to m6A, nucleosomes and QC (chr20, GM12878)

This tutorial takes a public PacBio Fiber-seq dataset (GM12878, HPRC v2), cuts it down to
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
| 7. QC | `ft qc` | `qc.tab` |

## Requirements

- [Apptainer](https://apptainer.org/) (or Singularity)
- Container images:
  - aws-cli: `apptainer pull aws-cli_latest.sif docker://amazon/aws-cli:latest`
  - [SAMtools](http://www.htslib.org/) 1.24:
    `apptainer pull samtools_1.24--h9dcdb79_1.sif docker://quay.io/biocontainers/samtools:1.24--h9dcdb79_1`
  - [fibertools-rs](https://github.com/fiberseq/fibertools-rs) 0.13.1:
    `apptainer pull fibertools-rs_0.13.1--h270025b_0.sif docker://quay.io/biocontainers/fibertools-rs:0.13.1--h270025b_0`

## 0. Setup

Adjust these paths to your own system. The values below are for the VSC (KU Leuven) cluster.

```bash
WORKDIR=/scratch/leuven/338/vsc33857
STAGING=/staging/leuven/stg_00096
AWS_SIF=$STAGING/home/apancik/software/singularity/aws-cli_latest.sif
SAM_SIF=samtools_1.24--h9dcdb79_1.sif
FT_SIF=fibertools-rs_0.13.1--h270025b_0.sif
THREADS=20

cd $WORKDIR
```

## 1. Download the data

The HPRC v2 Fiber-seq CRAMs are public, so no credentials are needed (`--no-sign-request`).

```bash
apptainer run -B $VSC_SCRATCH -B $STAGING $AWS_SIF \
  s3 cp --no-sign-request --endpoint-url https://s3.kopah.uw.edu \
  's3://stergachis/public/HPRCv2/FIRE-bams/GM12878_PM00001_PacBio.shared.ref.cram' .
```

## 2. Extract chromosome 20

```bash
apptainer run -B $VSC_SCRATCH $SAM_SIF \
  samtools view -b -o GM12878_PM00001.chr20.bam GM12878_PM00001_PacBio.shared.ref.cram chr20
```

## 3. Downsample to ~20X

Check the starting coverage (about 120X):

```bash
apptainer run -B $VSC_SCRATCH $SAM_SIF \
  samtools coverage GM12878_PM00001.chr20.bam
```

Subsample 16% of reads. In `-s 42.16`, the integer part (`42`) is the random seed and the
fractional part (`.16`) is the fraction of reads kept, so ~120X × 0.16 ≈ 20X.

```bash
apptainer run -B $VSC_SCRATCH $SAM_SIF \
  samtools view -@ $THREADS -s 42.16 -o GM12878_PM00001.chr20.20x.bam GM12878_PM00001.chr20.bam
```

## 4. Remove existing nucleosome and accessibility calls

The HPRC BAMs already contain fibertools annotations. This step removes the nucleosome
(`ns`, `nl`) and actuated/accessible element (`as`, `al`) tags so we can recompute them ourselves.

```bash
apptainer run -B $VSC_SCRATCH $SAM_SIF \
  samtools view -b -x ns,nl,as,al -@ $THREADS \
  -o GM12878_PM00001.chr20.20x.strip.bam GM12878_PM00001.chr20.20x.bam
```

## 5. Predict m6A

```bash
apptainer run -B $VSC_SCRATCH $FT_SIF \
  ft predict-m6a -t $THREADS \
  GM12878_PM00001.chr20.20x.strip.bam \
  GM12878_PM00001.chr20.20x.strip.m6a.bam
```

## 6. Call nucleosomes and index

```bash
apptainer run -B $VSC_SCRATCH $FT_SIF \
  ft add-nucleosomes -t $THREADS \
  GM12878_PM00001.chr20.20x.strip.m6a.bam \
  GM12878_PM00001.chr20.20x.strip.m6a.nuc.bam

apptainer run -B $VSC_SCRATCH $SAM_SIF \
  samtools index -@ $THREADS GM12878_PM00001.chr20.20x.strip.m6a.nuc.bam
```

## 7. Quality control

`--acf` adds the m6A autocorrelation, which shows the nucleosome repeat pattern.

```bash
apptainer run -B $VSC_SCRATCH $FT_SIF \
  ft qc --acf GM12878_PM00001.chr20.20x.strip.m6a.nuc.bam > qc.tab
```

## Full script

```bash
#!/usr/bin/env bash
set -euo pipefail

WORKDIR=/scratch/leuven/338/vsc33857
STAGING=/staging/leuven/stg_00096
AWS_SIF=$STAGING/home/apancik/software/singularity/aws-cli_latest.sif
SAM_SIF=samtools_1.24--h9dcdb79_1.sif
FT_SIF=fibertools-rs_0.13.1--h270025b_0.sif
THREADS=20
PREFIX=GM12878_PM00001.chr20

cd $WORKDIR

apptainer run -B $VSC_SCRATCH -B $STAGING $AWS_SIF \
  s3 cp --no-sign-request --endpoint-url https://s3.kopah.uw.edu \
  's3://stergachis/public/HPRCv2/FIRE-bams/GM12878_PM00001_PacBio.shared.ref.cram' .

apptainer run -B $VSC_SCRATCH $SAM_SIF samtools view -b -o $PREFIX.bam \
  GM12878_PM00001_PacBio.shared.ref.cram chr20
apptainer run -B $VSC_SCRATCH $SAM_SIF samtools coverage $PREFIX.bam
apptainer run -B $VSC_SCRATCH $SAM_SIF samtools view -@ $THREADS -s 42.16 \
  -o $PREFIX.20x.bam $PREFIX.bam
apptainer run -B $VSC_SCRATCH $SAM_SIF samtools view -b -x ns,nl,as,al -@ $THREADS \
  -o $PREFIX.20x.strip.bam $PREFIX.20x.bam

apptainer run -B $VSC_SCRATCH $FT_SIF ft predict-m6a -t $THREADS \
  $PREFIX.20x.strip.bam $PREFIX.20x.strip.m6a.bam
apptainer run -B $VSC_SCRATCH $FT_SIF ft add-nucleosomes -t $THREADS \
  $PREFIX.20x.strip.m6a.bam $PREFIX.20x.strip.m6a.nuc.bam
apptainer run -B $VSC_SCRATCH $SAM_SIF samtools index -@ $THREADS $PREFIX.20x.strip.m6a.nuc.bam

apptainer run -B $VSC_SCRATCH $FT_SIF ft qc --acf $PREFIX.20x.strip.m6a.nuc.bam > qc.tab
```

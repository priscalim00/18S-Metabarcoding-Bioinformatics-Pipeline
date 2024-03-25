# 18S-Metabarcoding-Bioinformatics-Pipeline

This pipeline has been designed for 18s amplicon sequencing data but should be adaptable to 16s or other forms of amplicon data.

## Step 1: Demultiplexing of reads & trimming of primers 
If sequencing facility provided demultiplexed reads, this step can be skipped

## Step 2: QIIME quality filtering

## Step 3: Denoising using DADA2

## Step 4: Merge sequences 
Necessary if multiple separate sequencing runs were performed (skip if only working with one run)

## Step 5: Assigning taxonomy using PR2

# 18S-Metabarcoding-Bioinformatics-Pipeline

This pipeline has been designed for 18S amplicon sequencing data but should be adaptable to 16S or other forms of amplicon data.

All data processing will be done in QIIME2. On the UNC Longleaf cluster, QIIME2 is activated using: `module load qiime2`

This pipeline assumes that you are familiar with submitting jobs to the Longleaf cluster. Refer: https://help.rc.unc.edu/submitting-jobs/

## Step 1: Demultiplexing of reads & trimming of primers 
If sequencing facility provided demultiplexed reads, this step can be skipped. 
Multiplexed reads will be returned as `Undetermined_R1_001.fastq.gz` and `Undetermined_R2_001.fastq.gz`

### Import multiplexed reads into QIIME2
1. Generate a tab-delimited metadata file containing `sample-id`, `i7-index` & `i5-index`
    - i7-index should be kept in forward sequence
    - i5-index should be reverse-complemented (i.e. if i5-index is `TATAGCCT`, input `AGGCTATA`)
   
   For details regarding metadata formatting, see: https://docs.qiime2.org/2024.2/tutorials/metadata/

   Example metadata file:
  
    ```
    sample-id  i7-index  i5-index
    sample-01  ATTACTCG  AGGCTATA
    sample-02  TCTCCGGA  GCCTCTAT
    ...
    ```
2. Copy FASTQ files + metadata file into one directory (highly recommend copying instead of moving to preserve the original files)
   
    ```
    mkdir raw_sequences  
    cp Undetermined_R*_001.fastq.gz raw_sequences
    ```
    If using MobaXterm (Windows), you can directly upload the metadata file into your directory.
    For Linux/Mac users, refer to https://its.unc.edu/research-computing/transferring-files-2/ regarding file transfer

3. Import FASTQ files into QIIME
    Follow **multiplexed paired-end FASTQ with barcodes in sequence** protocol: 
    https://docs.qiime2.org/2024.2/tutorials/importing/#sequence-data-with-sequence-quality-information-i-e-fastq
    
    To import `MultiplexedPairedEndBarcodeInSequence` into QIIME, it requires files named `forward.fastq.gz` and `reverse.fastq.gz`.
    As such, I would simply rename your files prior to importing:
    
    ```
    mv Undetermined_R1_001.fastq.gz forward.fastq.gz
    mv Undetermined_R2_001.fastq.gz reverse.fastq.gz 
    
    mkdir QZA
    qiime tools import \
      --type MultiplexedPairedEndBarcodeInSequence \
      --input-path raw_sequences \
      --output-path QZA/multiplexed-seqs.qza
    ```

### Demultiplexing samples 
  Demultiplexing is performed using QIIME's cutadapt plugin
     `qiime cutadapt demux-paired`: https://docs.qiime2.org/2024.2/plugins/available/cutadapt/demux-paired/
  
   ```
   qiime cutadapt demux-paired \
    --i-seqs QZA/multiplexed-seqs.qza \
    --m-forward-barcodes-file barcodes.txt \
    --m-forward-barcodes-column i7-index \
    --m-reverse-barcodes-file barcodes.txt \
    --m-reverse-barcodes-column i5-index \
    --o-per-sample-sequences QZA/demux-seqs.qza \
    --o-untrimmed-sequences QZA/untrimmed-seqs.qza
   ```
  You can now use `demux-seqs.qza` in subsequent steps!

## Step 2: Trimming adaptors and QC
  If sequencing facility returns demultiplexed reads, you should have `R1_001.fastq.gz` and `R2_001.fastq.gz` files for *each* sample. 

### Importing demultiplexed reads into QIIME
  To import these files into QIIME, follow the **Casava 1.8 paired-end demultiplexed fastq** protocol: https://docs.qiime2.org/2024.2/tutorials/importing/#sequence-data-with-sequence-quality-information-i-e-fastq

1. Start by creating a directory and copying all the raw files over
  ```
  mkdir raw_sequences
  cp *.fastq.gz raw_sequences 
  ```
2. Import files into QIIME
  ```
  mkdir QZA
  qiime tools import \
    --type 'SampleData[PairedEndSequencesWithQuality]' \
    --input-path raw_sequences \
    --input-format CasavaOneEightSingleLanePerSampleDirFmt \
    --output-path QZA/demux-seqs.qza
  ```
### Trimming adaptors
  You'll need to remove the sequencing adaptors from the 3' end. For more info, see: https://knowledge.illumina.com/software/general/software-general-reference_material-list/000002905

  Trimming can be performed using `qiime cutadapt trim-paired`: https://docs.qiime2.org/2024.2/plugins/available/cutadapt/trim-paired/

  ```
  qiime cutadapt trim-paired \
  --i-demultiplexed-sequences QZA/demux-seqs.qza \
  --p-adapter-f AGATCGGAAGAGCACACGTCTGAACTCCAGTCAC \
  --p-adapter-r AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT \
  --o-trimmed-sequences QZA/trimmed-demux-seqs.qza
  ```
  Note: Adaptor sequences depend on the primers you used for library prep. Generally it is the P7 adaptor on Read 1 and the reverse-complemented P5 adaptor on Read 2.

### Quality control 
1. Generate a quality report:
 ```
 mkdir QZV
 qiime demux summarize \
  --i-data QZA/trimmed-demux-seqs.qza \
  --o-visualization QZV/demux-quality.qzv
 ```

  The resultant `demux-quality.qzv ` file can be downloaded and viewed on https://view.qiime2.org/. You can use this to determine truncation length in DADA2 (sequence base at which quality begins to drop off significantly)

## Step 3: Denoising using DADA2
Now we will use DADA2 to denoise paired-end sequences, dereplicate them and filter out chimeras. DADA2 will merge paired-end sequences during this process, so there is no need to merge R1 and R2 files prior to this step.

Reference: https://docs.qiime2.org/2024.2/plugins/available/dada2/denoise-paired/

This step requires a significant amount of computing power, so request more cores (-n) and time (-t) when submitting your job

Adjust  `--p-trunc-len` according to your quality report. 

```
qiime dada2 denoise-paired \
    --i-demultiplexed-seqs QZA/trimmed-demux-paired.qza \
    --p-trunc-len-f 0 \
    --p-trunc-len-r 0 \
    --o-representative-sequences QZA/seqs.qza \
    --o-table QZA/table.qza \
    --o-denoising-stats QZA/denoising-stats.qza
```

Outputs:
  - `table.qza` is the feature (ASV) table containing read counts for each feature per sample
  - `seqs.qza` is the DNA sequence for each feature, this is later used to assign taxonomy
  - `denoising-stats.qza` contains information about the number of reads that passed the denoising process.

To visualize these files, we must convert them into `.qzv` files. You can optionally include a sample metadata file containing information about your samples. 

```
qiime feature-table summarize \
  --i-table QZA/table.qza \
  --o-visualization QZV/table.qzv \
  --m-sample-metadata-file QZA/metadata.txt
 
qiime feature-table tabulate-seqs \
  --i-data QZA/seqs.qza \
  --o-visualization QZV/seqs.qzv
 
qiime metadata tabulate \
  --m-input-file QZA/denoising-stats.qza \
  --o-visualization QZV/denoising-stats.qzv
```

Example metadata file:
```
sample-id  station  year
sample-01  A  2022
sample-02  B  2022
...
```
## Step 4: Assigning taxonomy
Taxonomic assignments are only as good as their reference databases, so it is important to choose a high quality reference database. I've included some recommendations but please do your own research prior as taxonomy is constantly changing and being updated!

For phytoplankton 18S rRNA, the PR2 database is recommended: https://pr2-database.org/

For bacterial 16S rRNA, the Greengenes2 database is likely most up-to-date: https://greengenes2.ucsd.edu/

Regardless of what database you choose to use, you'll need to acquire a reference sequences file (.fasta)  and a taxonomy file (usually a .csv or .txt) from the database website. Put these files in your main working directory. 

### Import reference database into QIIME2
```
qiime tools import \
  --type 'FeatureData[Sequence]' \
  --input-path {database name & version}.fasta \
  --output-path QZA/{database name & version}.qza

qiime tools import \
  --type 'FeatureData[Taxonomy]' \
  --input-format HeaderlessTSVTaxonomyFormat \
  --input-path {database name & version}_taxonomy.txt \
  --output-path QZA/ref-taxonomy.qza
```

### Extract reference reads & train classifer
This step is another computationally heavy step, so request nodes and time accordingly!
1. First, you must extract reference reads from your database to obtain just the region of the target sequences that was sequenced. This is done by specifying the primers you used during library prep. In this case, I'm using the 18S V4 region primers

Reference: https://docs.qiime2.org/2024.2/plugins/available/feature-classifier/extract-reads/

```
qiime feature-classifier extract-reads \
  --i-sequences QZA/{database name & version}.qza \
  --p-f-primer CCAGCASCYGCGGTAATTCC \
  --p-r-primer ACTTTCGTTCTTGAT \
  --o-reads QZA/ref-seqs.qza
```
2. Next, you'll need to train a Naive Bayes classifier using the reference sequences we just extracted

Reference: https://docs.qiime2.org/2024.2/plugins/available/feature-classifier/fit-classifier-naive-bayes/

```
qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads QZA/ref-seqs.qza \
  --i-reference-taxonomy QZA/ref-taxonomy.qza \
  --o-classifier QZA/classifier.qza
```
Once you've completed this step, you can use the resultant `ref-seqs.qza` and `classifier.qza` files to assign taxonomy to any set of samples. You'll only have to retrain the classifier if 
1) you want to update the version of the database you initially used, or
2) you want to extract a different target sequence (e.g. instead of the V4 region, you want to use the V9 region)

### Assigning taxonomy
Now you can assign taxonomy to your representative sequences using `qiime feature-classifier classify-sklearn`: https://docs.qiime2.org/2024.2/plugins/available/feature-classifier/classify-sklearn/

```
qiime feature-classifier classify-sklearn \
  --i-classifier QZA/classifier.qza \
  --i-reads QZA/seqs.qza \
  --o-classification QZA/assigned-taxonomy.qza
```

The resultant taxonomic assignments can be visualized and plotted as shown. The metadata file here is the same file used in Step 3 for `qiime feature-table summarize`
```
qiime metadata tabulate \
  --m-input-file QZA/assigned-taxonomy.qza\
  --o-visualization QZV/assigned-taxonomy.qzv

qiime taxa barplot \
 --i-table QZA/table.qza \
 --i-taxonomy QZA/assigned-taxonomy.qza \
 --m-metadata-file QZA/metadata.txt \
 --o-visualization QZV/taxa-bar-plots.qzv
```

## Step 5: OTU picking & merging sequences 

### OTU picking
The feature table generated by DADA2 returns Amplicon Sequence Variants (ASVs). If you decide you'd rather use Operational Taxnomic Units (OTUs), you can perform clustering of your features using `qiime vsearch cluster-features-open-reference`: https://docs.qiime2.org/2024.2/plugins/available/vsearch/cluster-features-open-reference/

Note: ASVs are generally preferred as they represent the actual sequence. However, clustering features at a 97-99% similarity can account for sequencing errors and is often sufficient for taxonomic analysis. The choice is yours!

```
qiime vsearch cluster-features-open-reference \
  --i-table QZA/table.qza \
  --i-sequences QZA/seqs.qza \
  --i-reference-sequences QZA/PR2-ref-seqs.qza \
  --p-perc-identity 0.97 \
  --o-clustered-table QZA/OTU_table.qza \
  --o-clustered-sequences QZA/OTU_seqs.qza \
  --o-new-reference-sequences QZA/OTU_ref-seqs.qza
```

### Merging samples
If multiple separate sequencing runs were performed, you can merge the feature tables and representative sequences using `qiime feature-table merge-seqs`: https://docs.qiime2.org/2024.2/plugins/available/feature-table/merge-seqs/ & `qiime feature-table merge`: https://docs.qiime2.org/2024.2/plugins/available/feature-table/merge/

This is useful if you want to include data from a previous sequencing run, but don't want to redo the entire QIIME pipeline OR if you don't have access to old raw sequence files but have the relevant .qza files. 

```
qiime feature-table merge-seqs \
    --i-data seqs1.qza seqs2.qza \
    --o-merged-data merged-seqs.qza

qiime feature-table merge \
    --i-tables table1.qza table2.qza \
    --o-merged-table merged-table.qza
```

After OTU picking/sample merging, you'll have to go back and assign taxonomy again as your representative sequences are now different. 

## Step 6: Exporting QIIME artifacts 
After obtaining the feature table, representative sequences and taxonomic assignments, I prefer to export these files so I can perform downstream analysis in R as it allows for greater customization 

QIIME2 has multiple built-in plug ins that supports diversity and differential abundance analysis, so you can definitely perform these analyses natively in QIIME2. I will not be going into these in detail, but here are some links.

`qiime diversity`: For alpha and beta diversity https://docs.qiime2.org/2024.2/plugins/available/diversity/#diversity

`qiime composition`: For differential abundance analysis using ANCOM-BC https://docs.qiime2.org/2024.2/plugins/available/composition/#composition

1. The table.qza file is exported as a `feature-table.biom` file so it must be converted to .tsv
```
mkdir export
qiime tools export \ 
--input-path QZA/table.qza \
--output-path export

biom convert \
-i export/feature-table.biom \
-o export/table.tsv \
--to-tsv

cd export
sed -i '1d' table.tsv
sed -i 's/#OTU ID//' table.tsv
cd ../
```
2. The representative sequences (seq.qza) will be exported as `dna-sequences.fasta` and taxonomy (assigned-taxonomy.qza) will be exported as .tsv
```
qiime tools export \
  --input-path QZA/seqs.qza \
  --output-path export

qiime tools export \
  --input-path QZA/assigned-taxonomy.qza \
  --output-path export
```
These datafiles can then be loaded into R for visualization/further analyses.

3. If you want to export separate FASTQ files for each sample, you can simply use `qiime tools export`:
```
qiime tools export \
    --input-path QZA/trimmed-demux-seqs.qza \
    --output-path export
```
This will generate R1 and R2 files for each sample.

If you instead want merged paired end sequences, you should first merge your reads using `qiime vsearch merge-pairs`: https://docs.qiime2.org/2024.2/plugins/available/vsearch/merge-pairs/
```
qiime vsearch merge-pairs \
    --i-demultiplexed-seqs QZA/trimmed-demux-seqs.qza \
    --o-merged-sequences QZA/merged-demux-seqs.qza \
    --o-unmerged-sequences QZA/unmerged-demux-seqs.qza

qiime tools export \
    --input-path QZA/merged-demux-seqs.qza \
    --output-path export
```

   



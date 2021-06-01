# 16S/12S Bioinformatics Processing

This worksheet describes bioinformatics processing of 16S and 12S amplicons for the following manuscript: "Optimizing in-cartridge bead beating extractions of microbial and fish environmental DNA" (in review)

Sean R. Anderson and Luke R. Thompson

June 2021

## 16S rRNA gene

Activate QIIME 2 environment, enable tab delineation, and navigate to the 16S folder. 

```python
conda activate qiime2-2020.8
source tab-qiime
cd Sterivex_16S_testing
```

Load paired-end 16S FASTQ files into QIIME 2. FASTQ files are stored in a subfolder (paired_reads). This will result in a .qza file with information on 16S sequences.

```python
qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' --input-path paired_reads --input-format CasavaOneEightSingleLanePerSampleDirFmt --output-path demux-paired-end-16S.qza
```

Summarize and visualize the quality of forward and reverse reads. This will also give a summary of the number of sequence reads per sample. 

```python
qiime demux summarize --i-data demux-paired-end-16S.qza --o-visualization demux-paired-end-16S.qzv
qiime tools view demux-paired-end-16S.qzv
```

### Primer trimming

Run Cutadapt on the demultiplexed 16S reads to remove V4-V5 primers. View the trimmed sequences. 

```python
qiime cutadapt trim-paired --i-demultiplexed-sequences demux-paired-end-16S.qza --p-front-f GTGYCAGCMGCCGCGGTAA --p-adapter-f AAACTYAAAKRAATTGRCGG --p-front-r CCGYCAATTYMTTTRAGTTT --p-adapter-r TTACCGCGGCKGCTGRCAC --verbose --p-error-rate 0.5  --o-trimmed-sequences demux-trimmed-16S.qza

# Summarize and view cutadapted reads
qiime demux summarize --i-data demux-trimmed-16S.qza --o visualization
demux-trimmed-16S.qzv
```

Export trimmed reads back to FASTQ file format in a new folder (16S-trimmed-reads). These reads can now be loaded into Tourmaline. 

```python
qiime tools export --input-path demux-trimmed-16S.qza --output-path 16S-trimmed-reads
```

### Tourmaline run

Tourmaline is a program that uses QIIME 2 commands and the Snakemake workflow management system to process amplicon sequencing data. Step-by-step instructions on using Tourmaline, including steps for installation, execution, and trouble-shooting are provided on GitHub (https://github.com/aomlomics/tourmaline/wiki). Only core steps and commands are reproduced here. 

Clone new version of Tourmaline into 16S folder.

```python
git clone https://github.com/aomlomics/tourmaline.git
```

Rename the folder with Tourmaline executable files. 

```
mv tourmaline tourmaline-16S-12092020
```

Navigate to the 01-imported folder and import the reference taxonomy files. Taxonomy files are kept in a separate folder (databases) and symbolic links are created for SILVA 138.1 reference sequences (refseqs.qza) and reference taxonomy (reftax.qza) files.	

```python
cd tourmaline-16S-12092020/01-imported
ln -s ~/Reference_databases/silva-138-99-seqs.qza refseqs.qza
ln -s ~/Reference_databases/silva-138-99-tax.qza reftax.qza
```

Create manifest files from the FASTQ files. This script is provided with Tourmaline. 

```python
scripts/create_manifest_from_fastq_directory.py /Users/sean.r.anderson/Sterivex_testing_16S/tourmaline-16S-12092020/00-data/fastq 00-data/manifest_pe.csv 00-data/manifest_se.csv
```

Execute Tourmaline. Parameters for DADA2, taxonomy assignment, and filtering can be modified in the config.yaml file.

```python
snakemake dada2_pe_denoise # Run paired-end DADA2

# Check out the number of reads that made it through DADA2 based on your parameters in the config.yaml file
cd 02-output-dada2-pe-unfiltered/00-table-repseqs/
qiime metadata tabulate --m-input-file dada2_stats.qza --o-visualization dada2_stats.qzv
qiime tools view dada2_stats.qzv

snakemake dada2_pe_taxonomy_unfiltered # Run unfiltered taxonomy

snakemake dada2_pe_diversity_unfiltered # Run diversity metrics

snakemake dada2_pe_report_unfiltered # Final report
```

### Extract 16S chloroplasts and assign taxonomy

Filter out the chloroplast sequences and ASV count table from the Tourmaline output files (repseqs.qza, taxonomy.qza, and table.qza files).

```python
qiime taxa filter-seqs --i-sequences 02-output-dada2-pe-unfiltered/00-table-repseqs/repseqs.qza --i-taxonomy 02-output-dada2-pe-unfiltered/01-taxonomy/taxonomy.qza --p-include Chloroplast --o-filtered-sequences 02-output-dada2-pe-unfiltered/00-table-repseqs/repseqs_chloroplast.qza

qiime taxa filter-table --i-table 02-output-dada2-pe-unfiltered/00-table-repseqs/table.qza --i-taxonomy 02-output-dada2-pe-unfiltered/01-taxonomy/taxonomy.qza --p-include Chloroplast --o-filtered-table 02-output-dada2-pe-unfiltered/00-table-repseqs/table_chloroplast.qza
```

Upload Protist Ribosomal Reference (PR2) database files for annotating 16S plastid reads (https://github.com/pr2database/pr2database/releases/tag/v4.12.0). Taxonomy files are originally from PhytoRef and have been formatted with the PR2 taxonomy framework. 

```python
# Import PR2 sequences using stand-alone QIIME 2
qiime tools import --type 'FeatureData[Sequence]' --input-path pr2_version_4.12.0_16S_mothur.fasta --output-path pr2_plastid_seqs.qza

# Import PR2 taxonomy
qiime tools import --type 'FeatureData[Taxonomy]' --input-format HeaderlessTSVTaxonomyFormat --input-path pr2_version_4.12.0_16S_mothur.tax --output-path pr2_plastid_tax.qza

# Extract PR2 sequences to the 16S V4-V5 primer region
qiime feature-classifier extract-reads --i-sequences pr2_plastid_seqs.qza --p-f-primer GTGYCAGCMGCCGCGGTAA --p-r-primer CCGYCAATTYMTTTRAGTTT --o-reads ref-seqs-pr2-plastid.qza

# Train the naive bayes classifier
qiime feature-classifier fit-classifier-naive-bayes --i-reference-reads ref-seqs-pr2-plastid.qza --i-reference-taxonomy pr2_plastid_tax.qza --o-classifier classifier_bayes_plastid.qza

# Fit classifier to chloroplast reads - results in taxonomy file
qiime feature-classifier classify-sklearn --i-classifier classifier_bayes_plastid.qza --i-reads repseqs_chloroplast.qza --o-classification asv_plastid.qza
```

Estimate unrooted and rooted tree for chloroplasts. 

```
qiime phylogeny align-to-tree-mafft-fasttree --i-sequences repseqs_chloroplast.qza --o-alignment aligned-rep-seqs-chl.qza --o-masked-alignment masked-aligned-rep-seqs-chl.qza --o-tree unrooted-tree-chl.qza --o-rooted-tree rooted-tree-chl.qza
```

Plastid ASV count, taxonomy table, and rooted tree files can now be uploaded in R for analysis of 16S chloroplast reads. 

## 12S rRNA gene

Navigate to the 12S folder. Input FASTQ files into QIIME 2. Summarize and visualize the quality of forward and reverse reads.

```python
cd Sterivex_testing_12S

qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' --input-path paired_reads --input-format CasavaOneEightSingleLanePerSampleDirFmt --output-path demux-paired-end-12S.qza

qiime demux summarize --i-data demux-paired-end-12S.qza --o-visualization demux-paired-end-12S.qzv
qiime tools view demux-paired-end-12S.qzv
```

### Primer trimming

Run Cutadapt on the 12S FASTQ files to remove MiFish-U primers. View the trimmed sequences. 

```python
qiime cutadapt trim-paired --i-demultiplexed-sequences demux-paired-end-12S.qza  --p-front-f GCCGGTAAAACTCGTGCCAGC --p-adapter-f CAAACTGGGATTAGATACCCCACTATG --p-front-r CATAGTGGGGTATCTAATCCCAGTTTG --p-adapter-r GCTGGCACGAGTTTTACCGGC --p-error-rate 0.5 --verbose --o-trimmed-sequences demux-trimmed-12S.qza

qiime demux summarize --i-data demux-trimmed-12S.qza --o-visualization demux-trimmed-12S.qzv
qiime tools view demux-trimmed-12S.qzv
```

### DADA2

Run paired-end DADA2 with the trimmed 12S reads. 

```python
qiime dada2 denoise-paired --i-demultiplexed-seqs demux-trimmed-12S.qza --p-n-threads 0 --p-trunc-len-f 187 --p-trunc-len-r 172 --p-max-ee-f 2 --p-max-ee-r 2 --p-n-reads-learn 1000000 --p-chimera-method consensus --p-trim-left-f 32 --p-trim-left-r 32 --verbose --o-representative-sequences rep-seqs-12S-vsearch.qza --o-table table-12S-vsearch.qza --o-denoising-stats stats-12S.qza
```

Check out the stats to see how many reads pass through DADA2 (~70%).

```python
qiime metadata tabulate --m-input-file stats-12S.qza --o-visualization stats-12S.qzv
qiime tools view stats-12S.qzv
```

### Assign taxonomy

Assign taxonomy using MitoHelper (https://github.com/aomlomics/mitohelper), which provides ready-to-use .qza reference sequences and taxonomy files of combined 12S, 16S, and 18S reads from NCBI, SILVA, MitoFish, and Fishes of the world. 

Here we will run the consensus-vsearch classifier against MitoHelper reference taxonomy and sequence files (March 2021 release; https://zenodo.org/record/4589660#.YKkOlpNKg_U).

```python
qiime feature-classifier classify-consensus-vsearch --i-query rep-seqs-12S-vsearch.qza --i-reference-reads 12S-16S-18S-seqs_March2021.qza --i-reference-taxonomy 12S-16S-18S-tax_March2021.qza --verbose --o-classification 12S-class-vsearch.qza
```

View taxonomy bar plots.

```python
qiime taxa barplot --i-table table-12S-vsearch.qza --i-taxonomy 12S-class-vsearch.qza --m-metadata-file Sampleinfo_12S.txt --o-visualization bar-plots-12S-vsearch.qzv
qiime tools view bar-plots-12S-vsearch.qzv
```

Estimate unrooted and rooted tree for 12S ASVs. 

```python
qiime phylogeny align-to-tree-mafft-fasttree --i-sequences rep-seqs-12S-vsearch.qza --o-alignment aligned-rep-seqs-vsearch.qza --o-masked-alignment masked-aligned-rep-seqs-vsearch.qza --o-tree unrooted-tree-vsearch.qza --o-rooted-tree rooted-tree-vsearch.qza
```


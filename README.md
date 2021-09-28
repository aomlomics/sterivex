## Enclosed Sterivex extractions of aquatic environmental DNA (eDNA)

Associated code for bioinformatics processing, data analysis, and figure generation for the following manuscript:

Anderson, S.R., & Thompson, L.R. Optimizing an enclosed bead beating extraction method of microbial and fish environmental DNA. Now published online in [Environmental DNA](https://onlinelibrary.wiley.com/doi/full/10.1002/edn3.251).

### Project workflow

#### Summary

Sterivex cartridge filters are commonly used to filter eDNA from marine and other aquatic habitats, though extracting DNA from a filter that is encased in plastic has been a challenge. We expand on recent enclosed bead beating techniques and test how the use of different bead sizes or combinations impact eDNA extraction, as well as the influence of using magnetic bead kits that are compatible with 96-well plate automated extractions. Code presented here represents the analysis of amplicon sequencing data (16S and 12S rRNA) to characterize the impact of enclosed bead treatments on population dynamics of microbes and bony fishes from a tropical lagoon (Bear Cut, Biscayne Bay, FL). Raw sequence data are available in NCBI SRA under BioProject ID PRJNA728349.

#### Bioinformatics processing

Code for processing 16S and 12S amplicons is available in _amplicon-proc.rmd_.

* 16S (V4-V5) and 12S (MiFish-U) primers removed with Cutadapt
* 16S reads processed using [Tourmaline](https://github.com/aomlomics/tourmaline)
* 12S reads processed with stand-alone QIIME 2
* 16S/12S ASVs inferred with paired-end DADA2
* Taxonomy assigned for 16S (SILVA; v.138.1), 16S chloroplasts (PR2; v.4.12), and 12S [(Mitohelper)](https://github.com/aomlomics/mitohelper)

#### R data visualization

Code for data analysis and visualization in R is provided as a Jupyter notebook (_data-viz.ipynb_). Plots were generated for each marker gene region with respect to bead content and sampling day. All files needed to reproduce plots are available in separate folders per marker region. 

* QIIME 2 taxonomy, ASV count table, and phylogenetic tree files uploaded to R using [qiime2R](https://github.com/jbisanz/qiime2R)
* Rarefaction curves, species richness (# of ASVs), Shannon diversity, and PCoA plots (weighted UniFrac)  
* Stacked taxonomy bar plots (class or family level) 
* Dot plots of the top 20 most relatively abundant ASVs
* Box plots of top 20 most abundant fish ASVs (used to poll fish experts in the region)
* Percent of all ASVs shared between bead treatments


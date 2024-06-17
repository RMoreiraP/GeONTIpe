# GeONType
A Snakemake workflow for genotype inversions and discovery of structural variants using Oxford Nanopore Long Reads 

## Pipeline Overview
GeONType is a Snakemake workflow that consists of various steps starting from downloading to genotyping inversions mediated by inverted repeats using Oxford Nanopore long reads. The pipeline consists of different steps performed by various software and also by custom scripts. The basic concept in genotyping is capturing reads that span the breakpoints of the inversions while characterizing the orientation by blasting probes. 
Despite the pipeline's initial purpose, it can also be used for:
  - Genotyping inversions not mediated by inverted repeats
  - Using PacBio reads (adjusting parameters to improve performance)
  - Using assemblies (adjusting parameters to improve performance)

For the proper functioning of this workflow, a config.yaml file is included for users to adjust each parameter, as well as a yml file for use with conda, which includes all the necessary software. The pipeline can be used complete or only one section depending if you need to download to mapp your reads or only to genotype inversions. 

## Installation
To run snakefile locally you must have execute:
```
git clone https://github.com/RMoreiraP/GeONType
cd GeONType
conda env create -f environment/Inversiones.yml
conda activate Inversiones
```
## Inputs
In case the complete pipeline needs to be executed (from downloading to genotype), the inputs are as follows:
  - allcoords.txt: This file specifies the inversions to be tested. The separator for each column must be a tab and should include the following information: Inv_ID chromosome BP1_start BP1_end BP2_start BP2_end (chromosome should be 1,2,...,X,Y).
  - ListaTodos.txt: This file contains all individuals with their respective URLs for downloading the necessary data in fastq or fasta format. It must be tab-separated with the format: Individual URL (If there are multiple URLs per individual, each URL should be associated with the same individual).
  - gender: This file provides the sex of each sample, separated by a tab: Sample sex (Women/Men).
  - conversion.txt: A file for converting chromosome names in the following style: chr1, chr2... (This can be modified depending on the reference genome used and the chromosome names it provides).
  - References: Std_ref.fa (reference genome to use, e.g., hg38), t2t_ref.fa (T2T or other reference genome), and All_snp.vcf.gz (SNPs to be used for verification). The names and paths can be modified in the config.yaml, although it is recommended to at least keep the path to avoid possible issues.

If the mapped files are already available, either locally or on an FTP server, allcoords.txt and ListaTodos.txt are not required, but we must have:
  - Individuals.txt: If mapping is not necessary, ListaTodos.txt can be replaced by this file, which lists all the samples to be tested.
  - gender: This file provides the sex of each sample, separated by a tab: Sample sex (Women/Men).
  - conversion.txt: A file for converting chromosome names in the following style: chr1, chr2... (This can be modified depending on the reference genome used and the chromosome names it provides).
  - References: Std_ref.fa (reference genome to use, e.g., hg38), t2t_ref.fa (T2T or other reference genome), and All_snp.vcf.gz (SNPs to be used for verification). The names and paths can be modified in the config.yaml, although it is recommended to at least keep the path to avoid possible issues.

All these outputs should be placed in the Infor directory, and all necessary references in Infor/Reference.

## Output
If you decide to execute the entire workflow, multiple files will be generated throughout each step. To track the workflow steps, you can enter the tracking_pipeline directory, where files for each output will be created. The main outputs will be found in the Genotyping directory and in the directories of each analyzed sample.

In the Genotyping directory, for each analyzed inversion, you will find all the generated probes. Note that if fewer than three probes are found, the specific directory for that inversion will not be created. This information can be checked in tracking_pipeline/created_probes.txt.

Within each individual’s directory, there will again be a Genotyping directory, containing each inversion in its own directory. Inside each inversion directory, you will find:

inversion.png: An image displaying the reads and the generated probes.
inversion_Genotype.txt: A file with information on each read and its respective genotype.
Both outputs are shown here as examples.


## Usage
To execute the workflow, the first step is to access the config file to modify the main directory paths:

```
{
    ## Directories
    "directory": "/main/route",
    "Ref": "/main/route/Infor/Reference/Std_ref.fa",
    
    ## Split of files
    "number_of_splits": 30,
    
    ## Filtering reads on statistics
    "Quality_reads": 7,
    "Read_length": 5000,
    
    ## Mapping of reads
    "Kind_reads": "map-ont",
    "Inversion_detection": "400,100",
    "SV_detection": "100,1000",
    "Mapq_filter": 20,
    
    ## Creating input to genotypes and probes
    "flanking_region": 10000,
    "extra_region": 500000,
    "size_probes": 300,
    "overlap_probes": 275,
    "seq_ref": "/main/route/Infor/Reference/chm13v2.0.fa",
    "extra_route": "/scratch/rmoreira/Infor/Reference",
    "min_uni_seq": 70,

    ## Genotyping inversions
    "local": "Y",
    "ftp_route": "https://ftp.1000genomes.ebi.ac.uk/vol1/ftp/data_collections/1KG_ONT_VIENNA/hg38",
    "task": "megablast",
    "coverage_of_probes": 90,
    "identity_probe": 85,

    ## Genotype classification
    "Major_allele_proportion": 0.95,
    "Minor_allele_proportion": 0.15,
    "Low_confidence_proportion": 0.05,

    ## Alternative SVs classification
    "Difference_respect_expected": 0.05,
    "Minimum_size_SVs": 100,
    "Ratio_to_split_SV": 0.1,
    
    ## SNP analysis
    "SNP_selection_route": "/main/route/Infor/Reference/All_snp.vcf.gz",
    "Quality_by_base": 1,
    "Min_freq_snps": 0.15,
    "Min_coverage": 1,
    "Distance_of_tree": 0.5,
}
```
If you have the mapping files in a local directory or perform them with workflow:
  - Parameter local:"Y"
  - Parameter ftp_route:"X" 

However, if you wish to work from an mapping files located in FTPs:
  - Parameter local:"N"
  - Parameter ftp_route:"/route/ftp" (The mapping files must have the following naming convention: individual.bam; If this is not possible, you can modify the script located at Scripts/Genotyping_scripts/runall.sh to adapt the naming)

If you want to keep all parameters after modifying the main paths, proceed with the following steps:
```
snakemake --cores all
```

## Notes
Here is an explanation of the function of some adjustable parameters:
  - Number_of_splits: This parameter determines into how many fragments each downloaded file will be divided. If you have limited RAM and prefer to launch many jobs simultaneously, it's preferable to increase this number. Conversely, if you have ample RAM, you can set this parameter to 1 to map the entire file without splitting it (Default: 30).
  - Quality_reads: Minimum quality threshold required for reads to pass filtering. (Default: 7)
  - Read_length: Minimum desired length of reads in base pairs. (Default: 5000)
  - Kind_reads: Equivalent to the -ax parameter of minimap2. (Default: map-ont)
  - Inversion_detection: Equivalent to the -z parameter of minimap2. (Default: 400,100)
  - SV_detection: Equivalent to the -r parameter of minimap2. (Default: 100,1000)
  - Mapq_filter: Filtering of reads based on mapping quality. (Default: 20)
  - Flanking_region: Additional sequence added upstream and downstream using the coordinates of the breakpoint as a reference. Higher complexity regions may require more flanking sequence. For high complexity regions a values of 50000-100000 are recomended. For low complex or simple inversions, values of 10000 or less should be enough. (Default: 100000)
  - Extra_region: Additional sequence used to assess the uniqueness of generated probes. Should be adjusted based on read type (e.g., ONT, PacBio Hi-fi). Longer reads require more extra region. (Default: 500000)
  - Size_probes: Size of the generated probes. Adjust based on base quality; higher for low-quality bases (ONT reads) and lower for high-quality bases (PacBio Hi-fi). Range of 100-500 bp for ONT, 100-300 bp for PacBio Hi-fi. (Default: 300)
  - Overlap_probes: Overlap between each tested probe. (Default: 275)
  - Min_uni_seq: Amount of sequence not classified as mobile elements. Lower values increase the likelihood of non-specific probes affecting genotyping. (Default: 70)
  - Task: Parameter -task for blast. (Default: megablast)
  - Coverage_of_probes: Parameter -qcov_hsp_perc for blast. (Default: 90)
  - Identity_probe: Parameter -perc_identity for blast. (Default: 85)
  - Major_allele_proportion: Proportion of reads with the same orientation to consider as a unique allele. (Default: 0.95)
  - Minor_allele_proportion: Proportion of reads of the minor allele required to be considered a real allele. (Default: 0.15)
  - Low_confidence_proportion: Proportion of reads of the minor allele to consider inconclusive genotypes. Used together with Minor_allele_proportion for inconclusive genotype thresholds. If low_confidence_proportion is the same value as minor_allele_proportion, not low confident genotypes will be generated (Default: 0.05)
  - Difference_respect_expected: Minimum percentage difference to consider an additional structural variant. Higher values are needed for reads with more errors. Recomendations: for ONT, 0.01-0.05 for PacBio Hi-Fi. (Default: 0.05)
  - Minimum_size_SVs: Minimum size of detection of structural variants. (Default: 100)
  - Ratio_to_split_SV: Percentage difference to determine if two similar found structural variants should be classified as one or two. (Default: 0.1)
  - Quality_by_base: Minimum quality at each base when using samtools mpileup. Higher values extract fewer but more likely correct bases. (Default: 1)
  - Min_freq_snps: Minimum frequency of the minor allele found by SNPs. (Default: 0.15)
  - Min_coverage: Minimum depth required at each SNP position found. (Default: 1)
  - Distance_of_tree: Minimum distance in the generated read cluster to consider reads as different. (Default: 0.5)

These parameters can be adjusted in the config file according to your specific data and analysis needs.

## Contributors
YO

## How to cite
Ricardo Moreira-Pinhal, Konstantinos Karakostis, Illya Yakymenko, Oscar Conchillo, Maria Díaz-Ros, Andrés Santos, Miquel Àngel Senar, Jaime Martínez-Urtaza, Marta Puig, Mario Cáceres, Complete characterization of human polymorphic inversions and other complex variants from long read data, a saber cuando y a saber donde se publica esto

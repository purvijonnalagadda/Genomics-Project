# NEW PROJECT NOTES
## 3/12 Goal: Download and Clean Reads
### Step 1: Load software environment
>#navigate to home directory to start the workflow\
cd ~
#load the Anaconda module on the HPC so conda and bioinformatics packages are available\
module load anaconda3
### Step 2: Create and activate SRA toolkit environment
#create a new conda environment called sra_env and install the SRA toolkit from the bioconda channel\
conda create -n sra_env -c bioconda sra-tools\
#activate the environment so SRA tools (prefetch, fasterq-dump) can be used
conda activate sra_env
### Step 3: Create project directory
#create a directory to hold sequencing files and analysis folders\
mkdir project_fastq\
#move into the project directory
cd project_fastq
### Step 4: Download sequencing data from SRA (BioSample accession: SAMN08784165, Run Accession: SRR6996007)
#download the sequencing run from the SRA database and store the .sra file locally\
prefetch SAMN08784165
### Step 5: Convert SRA file to FASTQ format
#move into the directory containing the downloaded .sra file\
cd SRR6996007
#convert the .sra sequencing file into FASTQ format for downstream analysis
fasterq-dump SRR6996007.sra

### Step ___: FastQc on Trimmed Files
Input Read Pairs: 1000988 Both Surviving: 741232 (74.05%) Forward Only Surviving: 172918 (17.27%) Reverse Only Surviving: 32678 (3.26%) Dropped: 54160 (5.41%)

### Step____: Evaluate trimmed output files 

The raw sequencing reads contained 1,000,988 sequences with lengths ranging from 35-301 bp and a GC content of 47%. The FastQC report indicated several quality issues. First, the per base sequence quality showed a clear decline toward the end of the reads, with many bases falling into lower quality ranges after around 200 bp. The per sequence GC content also differed from the theoretical distribution, and sequence duplication levels were high, suggesting there may be repeated reads. After trimming using a sliding window of 4 bases with a quality threshold of 20 and a minimum read length of 50 bp, the dataset was reduced to 741,232 paired reads, so 74.05% of read pairs survived trimming. The trimmed reads ranged from 50-301 bp, and the GC content remained stable at approximately 47%. The FastQC analysis of the trimmed reads showed an improvement in per base sequence quality, with most bases now falling within the high-quality (green) range across the read length. This suggests that we were able to successfully remove low-quality regions with trimming and improve the overall quality of the sequencing data.

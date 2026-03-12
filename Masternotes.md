# NEW PROJECT NOTES
## 3/12 Goal: Download and Clean Reads
### Step 1: Load software environment
#navigate to home directory to start the workflow
cd ~
#load the Anaconda module on the HPC so conda and bioinformatics packages are available
module load anaconda3
### Step 2: Create and activate SRA toolkit environment
#create a new conda environment called sra_env and install the SRA toolkit from the bioconda channel
conda create -n sra_env -c bioconda sra-tools
#activate the environment so SRA tools (prefetch, fasterq-dump) can be used
conda activate sra_env
### Step 3: Create project directory
#create a directory to hold sequencing files and analysis folders
mkdir project_fastq
#move into the project directory
cd project_fastq
### Step 4: Download sequencing data from SRA (BioSample accession: SAMN08784165, Run Accession: SRR6996007)
#download the sequencing run from the SRA database and store the .sra file locally
prefetch SAMN08784165
### Step 5: Convert SRA file to FASTQ format
#move into the directory containing the downloaded .sra file
cd SRR6996007
#convert the .sra sequencing file into FASTQ format for downstream analysis
fasterq-dump SRR6996007.sra

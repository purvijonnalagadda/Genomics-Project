# NEW PROJECT NOTES
## 3/12 Goal: Download and Clean Reads
### Step 1: Load software environment
>#navigate to home directory to start the workflow\
cd ~\
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
fasterq-dump SRR6996007.sra\
#compress FASTQ files to save space\
gzip *.fastq
### Step 6: Organize reads
#create directory for raw reads within project_fastq\
mkdir raw\
#create directory for cleaned reads within project_fastq\
mkdir cleaned
### Step 7: Run FastQC on raw reads
#start interactive compute session\
srun --pty bash\
#load FastQC\
module load fastqc\
#create output directory for FastQC reports\
mkdir projectfastqc_out\
#run FastQC on raw read files\
fastqc -o projectfastqc_out \
raw/SRR6996007.sra_1.fastq.gz \
raw/SRR6996007.sra_2.fastq.gz\
### Step 8: Trim reads with Trimmomatic
#make folder for cleaned reads within project_fastq\
mkdir cleaned\
#load trimmomatic\
module load trimmomatic\
#run paired-end trimming with adapter removal and quality filtering\
trimmomatic PE \
raw/SRR6996007.sra_1.fastq.gz raw/SRR6996007.sra_2.fastq.gz \
cleaned/SRR6996007_R1_paired.fastq.gz cleaned/SRR6996007_R1_unpaired.fastq.gz \
cleaned/SRR6996007_R2_paired.fastq.gz cleaned/SRR6996007_R2_unpaired.fastq.gz \
ILLUMINACLIP:/home/share/apps/trimmomatic/0.39/adapters/TruSeq3-PE.fa:2:30:10 \
SLIDINGWINDOW:4:20 \
MINLEN:50\

#Trimming results:\
Input Read Pairs: 1000988
Both Surviving: 590207 (58.96%)
Forward Only Surviving: 323943 (32.36%)
Reverse Only Surviving: 22658 (2.26%)
Dropped: 64180 (6.41%)\

These results indicate that approximately 59% of read pairs remained intact after trimming, while some reads were removed due to low quality or adapter contamination.
### Step 9: Run FastQC on trimmed reads
fastqc -o projectfastqc_out \
cleaned/SRR6996007_R1_paired.fastq.gz \
cleaned/SRR6996007_R2_paired.fastq.gz
### Step 10: Evaluate trimmed output files
The raw sequencing reads contained 1,000,988 read pairs with lengths ranging from 35–301 bp and a GC content of approximately 47%. The FastQC report indicated several quality issues. The per-base sequence quality declined toward the ends of reads, with many bases falling into lower quality ranges after around 200 bp. Sequence duplication levels were also elevated, suggesting the presence of repeated reads.

After trimming using Trimmomatic with adapter removal, a sliding window of 4 bases with a quality threshold of 20, and a minimum read length of 50 bp, the dataset was reduced to 590,207 paired reads, meaning 58.96% of read pairs survived trimming. The trimmed reads ranged from 50–301 bp and maintained a GC content of approximately 47%.

Notable observations include:
- Per base sequence quality:
Quality scores were high across most of the read length, with most bases falling in the green (high-quality) range.
- Adapter content:
Adapter contamination was reduced after trimming, indicating that adapter sequences were successfully removed.
- Sequence length distribution:
Trimmed reads showed a range of lengths due to removal of low-quality bases and adapters, with all reads longer than the 50 bp minimum threshold.

Overall, the trimming process improved the quality of the sequencing reads by removing low-quality regions and adapter contamination, resulting in a cleaner dataset for downstream analysis.

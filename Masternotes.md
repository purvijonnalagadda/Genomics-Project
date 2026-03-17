# New Project Notes

# Date: 3/12/2026

## Goal
Download sequencing reads from SRA and perform quality control and trimming.

---

## Workflow Overview

1. Load bioinformatics software environment  
2. Download sequencing reads from SRA  
3. Convert `.sra` file to FASTQ format  
4. Run FastQC on raw reads  
5. Trim adapters and low-quality bases with Trimmomatic  
6. Run FastQC again on trimmed reads  
7. Evaluate quality improvements  

---

## Step 1: Load software environment

Load the Anaconda module on the HPC so that conda and bioinformatics packages are available.

```bash
# navigate to home directory to start the workflow
cd ~

# load the Anaconda module
module load anaconda3
```

---

## Step 2: Create and activate SRA toolkit environment

Create a conda environment containing the SRA toolkit.

```bash
# create environment with SRA tools from bioconda
conda create -n sra_env -c bioconda sra-tools

# activate environment
conda activate sra_env
```

---

## Step 3: Create project directory

Create a directory to hold sequencing data and analysis files.

```bash
# create project directory
mkdir project_fastq

# move into project directory
cd project_fastq
```

---

## Step 4: Download sequencing data from SRA

BioSample accession: **SAMN08784165**  
Run accession: **SRR6996007**

```bash
# download sequencing run from SRA
prefetch SAMN08784165
```

---

## Step 5: Convert SRA file to FASTQ format

Convert the `.sra` file to FASTQ format for downstream analysis.

```bash
# move into directory containing downloaded run
cd SRR6996007

# convert SRA file to FASTQ
fasterq-dump SRR6996007.sra

# compress FASTQ files to save space
gzip *.fastq
```

---

## Step 6: Organize reads

Create directories for raw and cleaned reads.

```bash
# directory for raw sequencing reads
mkdir raw

# directory for trimmed reads
mkdir cleaned

# move FASTQ files into the raw directory
mv SRR6996007/*.fastq.gz raw/
```

---

## Step 7: Run FastQC on raw reads

Run quality control analysis on the raw sequencing reads.

```bash
# start interactive compute session
srun --pty bash

# load FastQC
module load fastqc

# create output directory for FastQC reports
mkdir projectfastqc_out

# run FastQC on raw read files
fastqc -o projectfastqc_out \
raw/SRR6996007.sra_1.fastq.gz \
raw/SRR6996007.sra_2.fastq.gz
```

---

## Step 8: Trim reads with Trimmomatic

Perform adapter removal and quality trimming.

```bash
# load trimmomatic
module load trimmomatic

# run paired-end trimming
trimmomatic PE \
raw/SRR6996007.sra_1.fastq.gz raw/SRR6996007.sra_2.fastq.gz \
cleaned/SRR6996007_R1_paired.fastq.gz cleaned/SRR6996007_R1_unpaired.fastq.gz \
cleaned/SRR6996007_R2_paired.fastq.gz cleaned/SRR6996007_R2_unpaired.fastq.gz \
ILLUMINACLIP:/home/share/apps/trimmomatic/0.39/adapters/TruSeq3-PE.fa:2:30:10 \
SLIDINGWINDOW:4:20 \
MINLEN:50
```

---

## Results

### Trimming Statistics

```
Input Read Pairs: 1000988
Both Surviving: 590207 (58.96%)
Forward Only Surviving: 323943 (32.36%)
Reverse Only Surviving: 22658 (2.26%)
Dropped: 64180 (6.41%)
```

Approximately **59% of read pairs remained intact after trimming**, while the remaining reads were removed due to low quality or adapter contamination.

---

## Step 9: Run FastQC on trimmed reads

Evaluate sequence quality after trimming.

```bash
fastqc -o projectfastqc_out \
cleaned/SRR6996007_R1_paired.fastq.gz \
cleaned/SRR6996007_R2_paired.fastq.gz
```

---

## Evaluation of Trimmed Output Files

The raw sequencing reads contained **1,000,988 read pairs**, with lengths ranging from **35–301 bp** and a **GC content of approximately 47%**.

Initial FastQC analysis indicated several quality concerns:

- Per-base sequence quality declined toward the ends of reads.
- Many bases fell into lower quality ranges after approximately **200 bp**.
- Sequence duplication levels were elevated, suggesting repeated reads.

After trimming with **Trimmomatic** using:

- adapter removal (`ILLUMINACLIP`)
- sliding window quality filtering (`SLIDINGWINDOW:4:20`)
- minimum read length (`MINLEN:50`)

the dataset was reduced to **590,207 paired reads**, meaning **58.96% of read pairs survived trimming**.

The trimmed reads ranged from **50–301 bp** and maintained a **GC content of approximately 47%**.

### Notable FastQC observations after trimming

**Per-base sequence quality**

Quality scores were high across most of the read length, with most bases falling in the high-quality (green) range.

**Adapter content**

Adapter contamination was reduced after trimming, indicating that adapter sequences were successfully removed.

**Sequence length distribution**

Trimmed reads showed a range of lengths due to removal of low-quality bases and adapter sequences, with all reads remaining longer than the **50 bp minimum threshold**.

Overall, trimming improved sequencing read quality by removing low-quality regions and adapter contamination, resulting in a cleaner dataset for downstream analysis.

---

## Project Directory Structure

```
project_fastq/
│
├── raw/
│   ├── SRR6996007.sra_1.fastq.gz
│   └── SRR6996007.sra_2.fastq.gz
│
├── cleaned/
│   ├── SRR6996007_R1_paired.fastq.gz
│   └── SRR6996007_R2_paired.fastq.gz
│
└── projectfastqc_out/
    ├── SRR6996007.sra_1_fastqc.html
    ├── SRR6996007.sra_2_fastqc.html
    ├── SRR6996007_R1_paired_fastqc.html
    └── SRR6996007_R2_paired_fastqc.html
```
# Date: 3/17/2026
## Goal: 
Perform metagenome assembly using trimmed paired-end reads with MEGAHIT.
---

## Workflow Overview

1. Create directories for assembly and logs
2. Write a SLURM script to run MEGAHIT
3. Submit the job to the HPC scheduler
4. Monitor job status
5. Retrieve assembled contigs

## Absolute Path to Fastq Files
```bash
# Identify full path to trimmed reads
/home/netid/project_fastq/trimmed
```

## Step 1: Confirm project directories and prepare assembly workspace
Verify that all required directories exist and organize the project structure before running assembly.

```bash
# navigate to project directory
cd project_fastq

# list existing directories
ls
```
Output: 
assembly  logs  projectfastqc_out  raw  SRR6996007  trimmed


Rename the cleaned reads directory to match the expected naming:
```bash
mv cleaned trimmed
```
Create directories for assembly output and logs:
```bash
mkdir assembly
mkdir logs
```

Create a subdirectory for SLURM scripts:
```bash
cd assembly
mkdir slurmscripts
cd ..
```

Verify final directory structure:
```bash
ls
```
Output:
assembly  logs  projectfastqc_out  raw  SRR6996007  trimmed

## Step 2: Create MEGAHIT environment
Install MEGAHIT using mamba and create a dedicated environment.
```bash
module load mamba
mamba create -y -n megahit-env -c conda-forge -c bioconda megahit
mamba activate megahit-env
```

## Step 3: Verify trimmed reads
```bash
cd trimmed
ls

Output:
R1_paired.fastq.gz    R2_paired.fastq.gz
R1_unpaired.fastq.gz  R2_unpaired.fastq.gz
```

## Step 4: Create and submit SLURM script
Create the MEGAHIT job script:
```bash
nano megahit_run.sh

# Submit the job
sbatch megahit_run.sh
```
Slurm Script:
#!/bin/bash
#SBATCH --job-name=megahit_SRR6996007
#SBATCH --nodes=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=03:00:00
#SBATCH --output=/home/pj288/project_fastq/logs/megahit_test1.%j.out
#SBATCH --error=/home/pj288/project_fastq/logs/megahit_test1.%j.err
#SBATCH --mail-user=pj288@georgetown.edu
#SBATCH --mail-type=END,FAIL

# note %j = job ID

module load mamba
source $(mamba info --base)/etc/profile.d/conda.sh

conda activate megahit-env

# Directory where the cleaned reads live
READDIR=/home/pj288/project_fastq/cleaned

# Input read files (paired-end)
READ1=${READDIR}/SRR6996007_R1_paired.fastq.gz
READ2=${READDIR}/SRR6996007_R2_paired.fastq.gz

# Output directory
OUTDIR=/home/pj288/project_fastq/assembly/megahit_SRR6996007_out

megahit \
  -1 ${READ1} \
  -2 ${READ2} \
  -t ${SLURM_CPUS_PER_TASK} \
  -o ${OUTDIR}

echo "Done. Contigs should be in: ${OUTDIR}/final.contigs.fa"

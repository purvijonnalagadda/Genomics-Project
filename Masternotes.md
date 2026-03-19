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

Create directories for raw and trimmed reads.

```bash
# directory for raw sequencing reads
mkdir raw

# directory for trimmed reads
mkdir trimmed

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
trimmed/SRR6996007_R1_paired.fastq.gz trimmed/SRR6996007_R1_unpaired.fastq.gz \
trimmed/SRR6996007_R2_paired.fastq.gz trimmed/SRR6996007_R2_unpaired.fastq.gz \
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
trimmed/SRR6996007_R1_paired.fastq.gz \
trimmed/SRR6996007_R2_paired.fastq.gz
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
├── trimmed/
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

## Goal
Perform metagenome assembly using trimmed paired-end reads with MEGAHIT.

---

## Workflow Overview

1. Confirm project directory structure  
2. Create MEGAHIT environment  
3. Verify trimmed reads  
4. Create and submit SLURM script  
5. Monitor job status and retrieve output  

---

## Step 1: Confirm project directories

Verify that all required directories exist and organize the project structure before running assembly.

```bash
# navigate to project directory
cd project_fastq

# list existing directories
ls
```

Output:

```
assembly  logs  projectfastqc_out  raw  SRR6996007  trimmed
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

```
assembly  logs  projectfastqc_out  raw  SRR6996007  trimmed
```

---

## Step 2: Create MEGAHIT environment

Install MEGAHIT using mamba and create a dedicated environment.

```bash
module load mamba

mamba create -y -n megahit-env -c conda-forge -c bioconda megahit

mamba activate megahit-env
```

---

## Step 3: Verify trimmed reads

Confirm that trimmed paired-end reads are present before assembly.

```bash
cd trimmed
ls
```

Output:

```
R1_paired.fastq.gz    R2_paired.fastq.gz
R1_unpaired.fastq.gz  R2_unpaired.fastq.gz
```

---

## Step 4: Create and submit SLURM script

Create the MEGAHIT job script.

```bash
nano megahit_run.sh
```

Submit the job:

```bash
sbatch megahit_run.sh
```

SLURM script:

```bash
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

module load mamba
mamba activate megahit-env

READDIR=/home/pj288/project_fastq/trimmed
READ1=${READDIR}/R1_paired.fastq.gz
READ2=${READDIR}/R2_paired.fastq.gz

OUTDIR=/home/pj288/project_fastq/assembly/megahit_SRR6996007_out

megahit \
  -1 ${READ1} \
  -2 ${READ2} \
  -t ${SLURM_CPUS_PER_TASK} \
  -o ${OUTDIR}

echo "Done. Contigs should be in: ${OUTDIR}/final.contigs.fa"
```

---

## Step 5: Monitor job and check output

Check if the job is running:

```bash
squeue -u pj288
```

Navigate to the assembly output directory:

```bash
cd /home/pj288/project_fastq/assembly/megahit_SRR6996007_out
ls
```

Output (initial):

```
checkpoints.txt  done  final.contigs.fa  intermediate_contigs  log  options.json
```

---
## Step 6: Install SeqKit for assembly statistics

```bash
module load mamba
mamba install -n megahit-env -c bioconda seqkit
mamba activate megahit-env
```
---

## Results

### Assembly Statistics (MEGAHIT)

Assembly was performed using **MEGAHIT v1.2.9** on trimmed paired-end reads.

```bash
seqkit stats -a final.contigs.fa
```

### Assembly Output Summary

```
file              format  type  num_seqs    sum_len  min_len  avg_len  max_len   Q1   Q2   Q3  sum_gap  N50  N50_num  GC(%)
final.contigs.fa  FASTA   DNA     20,288  9,474,621      204      467  129,544  338  395  476        0  439      958  47.09
```

### Key Metrics

- **Total contigs:** 20,288  
- **Total assembly length:** 9,474,621 bp  
- **Average contig length:** 467 bp  
- **Minimum contig length:** 204 bp  
- **Maximum contig length:** 129,544 bp  
- **N50:** 439 bp  
- **GC content:** 47.09%  

### Interpretation

- The assembly produced **~9.47 Mb of sequence data**, indicating successful reconstruction of genomic fragments from the metagenomic dataset.  
- The relatively **high number of contigs (20k+)** and **moderate N50 (439 bp)** suggest a fragmented assembly, which is typical for:
  - metagenomic samples  
  - complex microbial communities  
  - short-read sequencing data  

- The **maximum contig length (~129 kb)** indicates that some longer genomic regions were successfully assembled.

- The **GC content (~47%)** is consistent with many bacterial genomes, supporting biological plausibility of the assembly.

Overall, the MEGAHIT assembly successfully generated contigs suitable for downstream analyses such as:
- gene prediction  
- taxonomic classification  
- functional annotation  

---

## Output Files

```
assembly/megahit_SRR6996007_out/
│
├── final.contigs.fa        # Final assembled contigs
├── log                     # Assembly log file
├── checkpoints.txt         # Assembly checkpoints
├── options.json            # Parameters used
└── intermediate_contigs/   # Intermediate assembly steps
```

## Updated Project Directory Structure

```
project_fastq/
│
├── raw/
├── trimmed/
├── assembly/
│   └── megahit_SRR6996007_out/
│       ├── intermediate_contigs
│       ├── log
│       ├── tmp
│       └── final.contigs.fa
│
├── logs/
│   ├── megahit_test1.<jobID>.out
│   └── megahit_test1.<jobID>.err
│
└── projectfastqc_out/
```
# Date: 3/19/2026

## Goal
Identify viral contigs from assembled metagenomic data using VirSorter2, including environment setup, database installation, and SLURM-based execution.

---

## Workflow Overview

1. Load mamba and create VirSorter2 environment  
2. Verify installation  
3. Set up VirSorter2 database (fix conda issue)  
4. Prepare directories  
5. Create and run SLURM script  
6. Monitor job  
7. Check logs  
8. Interpret results  

---

## Step 1: Load environment and create VirSorter2 environment

```bash
module load mamba
mamba create -y -n vs2-env -c conda-forge -c bioconda virsorter
mamba activate vs2-env
```

---

## Step 2: Verify installation

```bash
which virsorter
virsorter --help | head
```

Expected:
```bash
~/.conda/envs/vs2-env/bin/virsorter
```

---

## Step 3: Set up VirSorter2 database (working fix)

```bash
rm -rf /home/pj288/db
rm -rf /home/pj288/.snakemake/conda
virsorter setup -d /home/pj288/db -j 4 --conda-frontend conda
```

Successful completion:
```bash
[INFO] Dependencies installed
[INFO] All setup finished
```

---

## Step 4: Prepare directories

```bash
cd /home/pj288/project_fastq
mkdir -p virsorter
mkdir -p logs
```

---

## Step 5: Create SLURM script

```bash
nano virsorter_run.sh
```

Paste:

```bash
#!/bin/bash
#SBATCH --job-name=virsorter_SRR6996007
#SBATCH --nodes=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=20G
#SBATCH --time=03:00:00
#SBATCH --mail-user=pj288@georgetown.edu
#SBATCH --mail-type=END,FAIL
#SBATCH --output=/home/pj288/project_fastq/logs/virsorter.%j.out
#SBATCH --error=/home/pj288/project_fastq/logs/virsorter.%j.err

module load mamba
source $(mamba info --base)/etc/profile.d/conda.sh
mamba activate vs2-env

INDIR=/home/pj288/project_fastq/assembly/megahit_SRR6996007_out
OUTROOT=/home/pj288/project_fastq/virsorter
SAMPLE_ID=SRR6996007

INPUT=${INDIR}/final.contigs.fa
OUTDIR=${OUTROOT}/vs2-${SAMPLE_ID}

mkdir -p "${OUTDIR}"

virsorter run \
  -w "${OUTDIR}" \
  -i "${INPUT}" \
  --keep-original-seq \
  --include-groups dsDNAphage,NCLDV,ssDNA \
  --min-length 5000

echo "Done. Results in: ${OUTDIR}"
```

---

## Step 6: Submit job

```bash
sbatch virsorter_run.sh
```

Example:
```bash
Submitted batch job 28245
```

---

## Step 7: Monitor job

```bash
squeue -u pj288
```

Running:
```bash
JOBID PARTITION NAME USER ST TIME NODES
28245 base1 virsorte pj288 R ...
```

Finished:
```bash
# no output → job complete
```

---

## Step 8: Check logs

```bash
cd /home/pj288/project_fastq/logs
ls
cat virsorter.<jobID>.err
```

---

## Results

### Filtering

```bash
# of seqs < 5000 bp removed: 20270
# of circular seqs: 1
# of linear seqs: 17
```

### Viral Detection

```bash
Full viral contigs: 17
Partial viral contigs: 0
Short viral contigs: 0
```

### Interpretation

- Most contigs were removed due to length (<5 kb)  
- 17 contigs were confidently identified as viral  
- Presence of 1 circular contig suggests a potential complete viral genome  
- Results are consistent with fragmented metagenomic assembly  

---

## Step 9: Inspect output

```bash
cd /home/pj288/project_fastq/virsorter/vs2-SRR6996007
ls
```

Output:

```bash
config.yaml
final-viral-boundary.tsv
final-viral-combined.fa
final-viral-score.tsv
iter-0
log
```

---

## Updated Project Directory Structure

```
project_fastq/
│
├── assembly/
│   └── megahit_SRR6996007_out/
│       └── final.contigs.fa
│
├── virsorter/
│   └── vs2-SRR6996007/
│       ├── final-viral-combined.fa
│       ├── final-viral-score.tsv
│       ├── final-viral-boundary.tsv
│       ├── config.yaml
│       └── log/
│
├── logs/
│   └── virsorter.<jobID>.out/.err
│
└── raw/ trimmed/ projectfastqc_out/
```

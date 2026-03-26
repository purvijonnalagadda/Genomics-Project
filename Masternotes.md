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

```
## Step 9: Locate VirSorter2 results and count viral contigs

After VirSorter2 finished, navigate to the output directory and inspect the result files.

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

Count the total number of viral contigs:

```bash
grep -c ">" final-viral-combined.fa
```

Output:

```bash
17
```

VirSorter2 identified **17 viral contigs**.

---

## Step 11: Install SeqKit and filter contigs by length

Activate the `megahit-env` environment and install SeqKit:

```bash
module load mamba
mamba activate megahit-env
mamba install -n megahit-env -c bioconda seqkit
```

After installation, remain in the VirSorter2 output directory:

```bash
cd /home/pj288/project_fastq/virsorter/vs2-SRR6996007
```

Count how many viral contigs are at least **5 kb** long:

```bash
seqkit seq -m 5000 final-viral-combined.fa | grep -c ">"
```

Output:

```bash
17
```

Create a filtered FASTA file containing only contigs at least **5 kb** long:

```bash
seqkit seq -m 5000 final-viral-combined.fa > final-viral-combined_min5kb.fa
```

Verify the number of contigs in the filtered file:

```bash
grep -c ">" final-viral-combined_min5kb.fa
```

Output:

```bash
17
```

---

## Interpretation

- VirSorter2 identified **17 total viral contigs**
- Filtering with SeqKit showed that **all 17 viral contigs were at least 5,000 bp long**
- Therefore, no viral contigs were removed during the 5 kb filtering step

The filtered file for downstream analysis is:

```bash
final-viral-combined_min5kb.fa
```
---
## Step 12: Cluster viral contigs into vOTUs with vclust

Create and activate a new environment for vclust:

```bash
module load mamba
mamba create -y -n votu-env -c bioconda -c conda-forge vclust
mamba activate votu-env
```

Verify installation:

```bash
vclust --help | head
```

Run vclust on the filtered viral contigs file:

```bash
cd /home/pj288/project_fastq/virsorter/vs2-SRR6996007
```

Prefilter similar genome pairs:

```bash
vclust prefilter \
  -i final-viral-combined_min5kb.fa \
  -o fltr.txt
```

Align sequences and calculate ANI:

```bash
vclust align \
  -i final-viral-combined_min5kb.fa \
  -o ani.tsv \
  --filter fltr.txt
```

Cluster sequences into vOTUs at 95% ANI:

```bash
vclust cluster \
  -i ani.tsv \
  -o clusters.tsv \
  --ids ani.ids.tsv \
  --metric ani \
  --ani 0.95 \
  --out-repr
```

### Results

The clustering step reported:

- **17 total objects**
- **17 total clusters (including singletons)**

This indicates that none of the viral contigs clustered together at the 95% ANI threshold, so each contig represented its own vOTU.

---

## Step 13: Extract vOTU representative sequences

Generate a list of vOTU seed IDs from the clustering output. Because `clusters.tsv` contains a header row, skip the first line when extracting IDs:

```bash
tail -n +2 clusters.tsv | awk '{print $2}' | sort -u > votu_seeds.txt
```

Switch back to the environment containing SeqKit:

```bash
mamba deactivate
mamba activate megahit-env
```

Extract the representative vOTU sequences:

```bash
seqkit grep -f votu_seeds.txt final-viral-combined_min5kb.fa > votus_final.fna
```

Sanity checks:

```bash
wc -l votu_seeds.txt
grep -c ">" votus_final.fna
```

Output:

```bash
17 votu_seeds.txt
17
```

---

## Interpretation

- VirSorter2 identified **17 viral contigs**
- Filtering confirmed that **all 17 contigs were ≥ 5 kb**
- vclust grouped the sequences into **17 vOTUs**
- Therefore, each viral contig represented a unique viral operational taxonomic unit
- `votus_final.fna` contains the final non-redundant representative viral sequences for downstream analysis
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

# Date: 3/24/2026

## Goal:
1. Identify viral contigs (VirSorter2)
2. Cluster into vOTUs
3. Assess genome quality (CheckV)
4. Map reads to class pooled vOTUs (Bowtie2)
5. Upload BAM files for class-wide analysis

---

## 1. Locate vOTU File

```bash
cd /home/osh10/project_fastq/virsorter/vs2-SRR6996007
ls
```

vOTU file used:
```
votus_final.fna
```

Get full path:
```bash
realpath votus_final.fna
```

---

## 2. Set Up CheckV

```bash
mkdir -p /home/osh10/project_fastq/checkv
cd /home/osh10/project_fastq/checkv

module load checkv
checkv download_database ./
```

---

## 3. Run CheckV (Slurm)

Create Slurm script:

```bash
nano run_checkv.slurm
```

Contents of script:

```bash
#!/bin/bash
#SBATCH --job-name=checkv
#SBATCH --output=/home/osh10/project_fastq/logs/checkv-%j.out
#SBATCH --error=/home/osh10/project_fastq/logs/checkv-%j.err
#SBATCH --time=03:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=16G
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=osh10@georgetown.edu

module load checkv

CHECKVDB="/home/osh10/project_fastq/checkv/checkv-db-v1.5"
INPUT="/home/osh10/project_fastq/virsorter/vs2-SRR6996007/votus_final.fna"
OUTDIR="/home/osh10/project_fastq/checkv/vOTUs"

mkdir -p "${OUTDIR}"

echo "Running CheckV on ${INPUT}"
checkv end_to_end "${INPUT}" "${OUTDIR}" -d "${CHECKVDB}" -t ${SLURM_CPUS_PER_TASK}
echo "Done."
```

Submit job:

```bash
sbatch run_checkv.slurm
```

Check job status:

```bash
squeue -u osh10
```

Output directory:

```
/home/osh10/project_fastq/checkv/vOTUs/
```

---

## 4. CheckV Results

```bash
cd /home/osh10/project_fastq/checkv/vOTUs
ls
```

Main results file:
```
quality_summary.tsv
```

```bash
column -t quality_summary.tsv | less -S
tail -n +2 quality_summary.tsv | cut -f8 | sort | uniq -c
```

---

## 5. Download Class Pooled vOTUs

```bash
cd /home/osh10/project_fastq/bowtie2
gcloud storage cp gs://gu-biology-dept-class/ClassProject/votus_10kb_6samples.fna .
```

---

## 6. Build Bowtie2 Index

```bash
srun --pty bash
module load bowtie2

bowtie2-build votus_10kb_6samples.fna votu_index

exit
```

---

## 7. Locate Trimmed Reads

```bash
find /home/osh10/project_fastq/trimmed -type f | grep paired
```

---

## 8. Bowtie2 + Samtools

```bash
#!/bin/bash
#SBATCH --job-name=bowtie2_vOTUs
#SBATCH --output=/home/osh10/project_fastq/logs/bowtie-%j.out
#SBATCH --error=/home/osh10/project_fastq/logs/bowtie-%j.err
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=osh10@georgetown.edu
#SBATCH --time=8:00:00
#SBATCH --mem=16G

module purge
module load bowtie2/2.5.4
module load samtools

SAMPLE="SRR6996007"
INDEX="/home/osh10/project_fastq/bowtie2/votu_index"
OUTPUTDIR="/home/osh10/project_fastq/bowtie2/${SAMPLE}"

mkdir -p "${OUTPUTDIR}"
cd "${OUTPUTDIR}"

bowtie2 -p 8 \
  -x "${INDEX}" \
  -1 "/home/osh10/project_fastq/trimmed/SRR6996007_R1_paired.fastq.gz" \
  -2 "/home/osh10/project_fastq/trimmed/SRR6996007_R2_paired.fastq.gz" \
| samtools view -bS - > "${SAMPLE}.bam"

samtools sort "${SAMPLE}.bam" > "${SAMPLE}_sorted.bam"
samtools index "${SAMPLE}_sorted.bam"
```

---

## 9. Rename Output

```bash
mv sample1_osh10_sorted.bam sample7_osh10_sorted.bam
mv sample1_osh10_sorted.bam.bai sample7_osh10_sorted.bam.bai
```

---

## 10. Upload

```bash
gcloud storage cp sample7_osh10_sorted.bam gs://osh10/
gcloud storage cp sample7_osh10_sorted.bam.bai gs://osh10/
```

---

## Conceptual Notes

- vOTUs reduce redundancy and represent viral populations  
- pooled vOTUs increase diversity and improve genome reconstruction  
- mapping identifies viral presence and abundance  

---

## Updated Directory Structure

```
/home/osh10/project_fastq/
├── raw/                          # raw FASTQ files
├── trimmed/                      # trimmed reads (Trimmomatic output)
│   ├── SRR6996007_R1_paired.fastq.gz
│   ├── SRR6996007_R2_paired.fastq.gz
│   ├── SRR6996007_R1_unpaired.fastq.gz
│   └── SRR6996007_R2_unpaired.fastq.gz
│
├── virsorter/
│   └── vs2-SRR6996007/
│       ├── final-viral-combined.fa
│       ├── final-viral-combined_min5kb.fa
│       ├── final-viral-score.tsv
│       ├── final-viral-boundary.tsv
│       ├── votus_final.fna              # vOTUs (used for CheckV)
│       ├── clusters.tsv
│       ├── ani.tsv
│       └── iter-0/
│
├── checkv/
│   ├── checkv-db-v1.5/                 # CheckV database
│   └── vOTUs/                          # CheckV output
│       ├── quality_summary.tsv
│       ├── completeness.tsv
│       ├── contamination.tsv
│       ├── complete_genomes.tsv
│       ├── viruses.fna
│       └── proviruses.fna
│
├── bowtie2/
│   ├── votus_10kb_6samples.fna         # class pooled vOTUs
│   ├── votu_index.*                    # bowtie2 index files
│   └── SRR6996007/
│       ├── SRR6996007.bam              # unsorted BAM
│       ├── sample7_osh10_sorted.bam    # final BAM
│       └── sample7_osh10_sorted.bam.bai
│
└── logs/                              # slurm output/error logs
```
# Date: 3/26/2026
<img width="1512" height="1008" alt="image" src="https://github.com/user-attachments/assets/c5fb9e07-d0e2-4b9e-bc98-51c3a9ed0c1c" />
<img width="1512" height="1050" alt="image" src="https://github.com/user-attachments/assets/a7d1f445-5afb-49ba-ac59-5112c23b944d" />
<img width="1512" height="1050" alt="image" src="https://github.com/user-attachments/assets/499e654c-5d11-4f94-b055-281d59e4094b" />



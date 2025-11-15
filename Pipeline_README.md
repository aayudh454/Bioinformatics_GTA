# CTC360 ctDNA Pre/Post Treatment Pipeline

This README describes the full computational workflow for analyzing **pre- and post-treatment ctDNA samples** from patient **CTC360** in project **PRJNA714799**, focusing on **KRAS mutations on chromosome 12**.

## Variables
PATIENT="CTC360"
PRE="SRR13973704"
POST="SRR13973878"

The workflow is optimized for **Google Colab** but is portable to any Linux environment.

---

## 1. Environment Setup

### 1.1 Mount Google Drive (Colab only)

Mounting Google Drive allows storing outputs, figures, and results persistently.  
Running directly in Colab means files inside `/content` will be deleted when the session resets, so Drive helps preserve results.

---

### 1.2 Create Project Directory Structure

Organizing files improves reproducibility and cleanliness of the workflow.

Folders created:

- `fastq/` – raw FASTQ data  
- `trimmed/` – adapter- and quality-trimmed FASTQs  
- `qc/` – FastQC and fastp reports  
- `bam/` – aligned BAM files  
- `variants/` – VCF files from Mutect2  
- `figures/` – final visualization outputs  

This clean structure mirrors common bioinformatics project layouts.

---

### 1.3 Install Required Tools

Tools installed:

- **fastqc** – raw QC  
- **fastp** – trimming and QC  
- **bwa** – alignment  
- **samtools** – BAM handling  
- **bcftools** – VCF handling  
- **GATK 4.4.0.0** – variant calling (Mutect2) and duplicate marking  

These represent a standard ctDNA analysis toolkit used in industry and academia.

---

## 2. Reference Genome Preparation

### 2.1 Download chr12 (hg38)

We use **only chromosome 12** because ctDNA analysis focuses on KRAS mutations, saving:

- CPU time  
- RAM  
- Disk usage  

This makes it ideal for Colab.

### 2.2 Build Reference Indexes

We generate:

- BWA index → required for alignment  
- `.fai` index → required for random access  
- `.dict` file → required by GATK  

Tools must share identical reference files and indexes.

---

## 3. FASTQ Retrieval

### 3.1 Download FASTQs (EBI FTP)

Direct downloading from **EBI ENA** is significantly faster and simpler than using SRA Toolkit in Colab.

FASTQs are renamed:

- `CTC360_pre_R1.fastq.gz`  
- `CTC360_pre_R2.fastq.gz`  
- `CTC360_post_R1.fastq.gz`  
- `CTC360_post_R2.fastq.gz`  

Clear naming clarifies which files belong to which timepoint.

---

## 4. Quality Control and Trimming

### 4.1 FastQC

Initial QC identifies:

- Sequencing quality  
- Adapter contamination  
- GC bias  
- Overrepresented sequences  

Running FastQC early prevents wasting time on broken fastq files.

---

### 4.2 fastp

`fastp` handles:

- Adapter trimming  
- Low-quality base removal  
- Quality filtering  
- Generating HTML + JSON QC reports  

Clean data is essential for accurate alignment and reliable variant calling, especially for low-frequency ctDNA mutations.

---

## 5. Alignment to chr12

### 5.1 BWA-MEM Alignment + Sorting

Mapping trimmed reads to chr12 and sorting immediately (`bwa mem | samtools sort`) avoids generating large intermediate SAM files.

BAMs produced:

- `CTC360_pre.bam`  
- `CTC360_post.bam`  

### 5.2 Index BAM Files

Indexing enables:

- IGV visualization  
- GATK access  
- Efficient region lookups  

Indexes are required for most downstream steps.

---

## 6. Duplicate Marking

PCR duplicates inflate allele counts—especially problematic in ctDNA where variant allele fractions (VAFs) are often <5%.

We remove duplicates using:

`MarkDuplicates --REMOVE_DUPLICATES true`

Outputs:

- `CTC360_pre.dedup.bam`
- `CTC360_post.dedup.bam`

Including metrics files summarizing duplication rates.

---

## 7. Variant Calling with Mutect2

Mutect2 is designed for:

- somatic mutations  
- low-VAF variants  
- ctDNA  
- tumor-only calling  

We run Mutect2 on pre and post samples independently.

Outputs:

- `CTC360_pre.unfiltered.vcf.gz`  
- `CTC360_post.unfiltered.vcf.gz`

---

## 8. Variant Filtering

`FilterMutectCalls` applies Mutect2’s built-in filters to produce high-confidence variants:

- `CTC360_pre.vcf.gz`  
- `CTC360_post.vcf.gz`

We use PASS variants for downstream analysis.

---

## 9. Python-Based Variant Analysis

### 9.1 Load VCFs into pandas

Parsing VCFs into DataFrames allows flexible manipulation and filtering.

### 9.2 Extract VAF

We extract VAF from:

- `AF` value (if provided)  
- else compute from `AD` (allelic depths)  

VAF is the most important metric for ctDNA monitoring.

### 9.3 Focus on KRAS Region

Restrict to region:

```
chr12:25205246–25250929 (hg38)
```

This corresponds to KRAS, a clinically relevant hotspot gene.

### 9.4 Merge Pre vs Post

Create:

```
VAR_ID = CHROM:POS:REF>ALT
```

Merge on this key to track the same variant across time.

Interpretation becomes easy:

- Variant disappears → therapy effective  
- Variant decreases → partial response  
- Variant increases → possible resistance clone  

---

## 10. Visualization of VAF Changes

Generate a barplot:

- X-axis: Variant ID  
- Y-axis: VAF  
- Colors: Pre vs Post  

This provides a clear picture of ctDNA KRAS dynamics across treatment.

---

## 11. Summary of Workflow Design

The pipeline prioritizes:

### **Efficiency**
- Using chr12 only  
- Using fast tools (fastp, BWA-MEM, samtools)

### **Accuracy**
- Trimming  
- Duplicate removal  
- Mutect2 with filtering  

### **Interpretability**
- Pre/post VAF comparison  
- KRAS-focused analysis  

### **Reproducibility**
- Clear folder structure  
- Well-defined steps  
- Portable commands  

---

## Notable variants

| Chr | Pos_start | Ref | Alt | Mutation type      | Gene | AA change    | Baseline cfDNA Total reads | Baseline cfDNA Ref reads | Baseline cfDNA Var reads | Baseline cfDNA VAF (%) | Tissue Total reads | Tissue Ref reads | Tissue Var reads | Tissue VAF (%) |
|-----|-----------|-----|-----|---------------------|------|--------------|-----------------------------|---------------------------|---------------------------|--------------------------|---------------------|-------------------|-------------------|------------------|
| 12  | 25398284  | C   | A   | missense_variant    | KRAS | p.Gly12Val   | 2536                        | 2528                      | 5                         | 0.20                     | 634                 | 557               | 77                | 12.15            |



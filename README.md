# 🧬 variant-calling-pipeline

> Reproducible variant calling pipeline for Illumina paired-end sequencing data — from raw FASTQ to annotated VCF, with two parallel calling approaches (GATK HaplotypeCaller and FreeBayes).

![Platform](https://img.shields.io/badge/platform-Google%20Colab-F9AB00?logo=googlecolab)
![Conda](https://img.shields.io/badge/conda-mamba-green)
![Status](https://img.shields.io/badge/status-in%20development-yellow)

---

## Overview

This pipeline processes Illumina paired-end sequencing data (BRCA1 gene, GRCh38) through quality control, trimming, alignment, and variant calling using two complementary approaches:

| Approach | Caller | Best for |
|----------|--------|----------|
| **A** | GATK HaplotypeCaller | Clinical / germline (GATK Best Practices) |
| **B** | FreeBayes | Research / flexible allele-frequency analysis |

```
Raw FASTQ (paired-end)
    │
    ├── [Step 1] Download data            → Entrez (chr17 GRCh38) + GitHub (BRCA1)
    ├── [Step 2] Quality Control          → FastQC + MultiQC
    ├── [Step 3] Trimming                 → fastp (Q≥20)
    ├── [Step 4] Alignment                → BWA-MEM → sorted, indexed, mapped BAM
    ├── [Step 5A] Variant Calling — GATK  → HaplotypeCaller → GVCF → VCF → filter
    ├── [Step 5B] Variant Calling — Free  → FreeBayes → bcftools filter (QUAL≥20)
    ├── [Step 6] Normalisation (FreeBayes)→ rename chromosomes + bcftools norm
    ├── [Step 7] Annotation               → bcftools annotate + ClinVar
    └── [Step 8] Summary Statistics       → bcftools stats (SNP/INDEL, Ts/Tv)
```

---

## Repository Structure

```
variant-calling-pipeline/
├── variant_calling_pipeline_v2.ipynb   # Main notebook (Google Colab)
├── README.md
├── .gitignore
└── LICENSE
```

> All input data (FASTQ, reference genome) are downloaded automatically inside the notebook.

---

## Requirements

- **Google Colab** (recommended — no local installation needed)
- Or: Linux / WSL2 with [Mamba](https://mamba.readthedocs.io/) installed

### Tools installed automatically by the notebook

| Tool | Conda environment |
|------|-------------------|
| entrez-direct | `entrez` |
| FastQC, MultiQC, fastp | `quality` |
| BWA | `bwa` |
| Samtools | `samtools` |
| GATK4 | `gatk` |
| FreeBayes, bcftools | `freebayes` |
| bcftools | `bcftools` |

---

## Usage

### Google Colab (recommended)

1. Open [Google Colab](https://colab.research.google.com/)
2. Upload `variant_calling_pipeline_v2.ipynb`
3. Run cells sequentially from top to bottom
4. After **Step 0**, wait for the Colab kernel to restart before continuing

### Local (Linux / WSL2)

Replace `%%bash` cells with terminal commands and `mamba run -n <env>` with `conda activate <env>`.

---

## Input Data

| File | Source |
|------|--------|
| `chr17_GRCh38.fasta` | NCBI Entrez (NC_000017.11) |
| `BRCA1_*.fq.gz` | [oncogensus/Curso-RSG-Brazil](https://github.com/oncogensus/Curso-RSG-Brazil) |
| `clinvar_mini.vcf.gz` | same |

---

## Output Structure

```
├── fastqc/               # Per-sample FastQC HTML reports
├── multiqc/              # Aggregated MultiQC report
├── trimmed/              # Trimmed FASTQ files (fastp)
├── aligned/              # Sorted, indexed and mapped-only BAM files
├── variants/
│   ├── gatk/             # GATK results (GVCFs, cohort VCF, filtered VCF)
│   └── freebayes/        # FreeBayes results + chr-renamed + normalised VCF
└── annotated/            # VCF files annotated with ClinVar (both callers)
```

---

## Why FreeBayes needs a normalisation step (Step 6)

The `bcftools annotate` command matches variants by **exact chromosome name + position + allele**. Without normalisation, FreeBayes produces empty annotation IDs due to two differences:

| Issue | FreeBayes output | ClinVar mini expects | Fix |
|-------|-----------------|----------------------|-----|
| Chromosome name | `17` (from Entrez FASTA header) | `chr17` | `bcftools annotate --rename-chrs` |
| INDEL representation | May be right-aligned | Left-aligned | `bcftools norm -m-any` |

This is why Step 6 (`rename-chrs` + `bcftools norm`) is required before annotation for FreeBayes, but not for GATK (which inherits chromosome names from the reference dict created by `CreateSequenceDictionary`).

---

## GATK vs FreeBayes: Key Differences

| Feature | GATK HaplotypeCaller | FreeBayes |
|---------|---------------------|-----------|
| Calling model | Local re-assembly + HMM | Bayesian allele frequency model |
| Multi-sample workflow | GVCF → CombineGVCFs → GenotypeGVCFs | Direct multi-BAM input |
| Speed | Slower | Faster |
| GVCF intermediate | Required | Not needed |
| Chromosome naming | Inherits from reference dict | Inherits from FASTA header |
| Normalisation needed for annotation | No | Yes (Step 6) |
| Best for | Germline / clinical | Research / flexible |

---

## Filtering Parameters

| Parameter | Value | Tool |
|-----------|-------|------|
| Min base quality (trimming) | Q ≥ 20 | fastp |
| GATK QD filter | < 2.0 | VariantFiltration |
| GATK FS filter | > 60.0 | VariantFiltration |
| GATK MQ filter | < 40.0 | VariantFiltration |
| FreeBayes QUAL filter | ≥ 20 | bcftools filter |
| Variant types kept | SNPs + INDELs | bcftools view |
| Min mapping quality | 20 | FreeBayes |
| Min base quality | 20 | FreeBayes |

---

## Tools and References

| Tool | Reference |
|------|-----------|
| FastQC | [Andrews, 2010](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) |
| MultiQC | [Ewels et al., 2016](https://doi.org/10.1093/bioinformatics/btw354) |
| fastp | [Chen et al., 2018](https://doi.org/10.1093/bioinformatics/bty560) |
| BWA | [Li & Durbin, 2009](https://doi.org/10.1093/bioinformatics/btp324) |
| Samtools / bcftools | [Danecek et al., 2021](https://doi.org/10.1093/gigascience/giab008) |
| GATK4 | [Van der Auwera & O'Connor, 2020](https://doi.org/10.1002/cpz1.20) |
| FreeBayes | [Garrison & Marth, 2012](https://arxiv.org/abs/1207.3907) |
| ClinVar | [Landrum et al., 2018](https://doi.org/10.1093/nar/gkx1153) |

---

## Notes

- Run **Step 0** first and wait for the Colab kernel to restart automatically before proceeding.
- Each tool runs in its own isolated mamba environment to avoid dependency conflicts.
- The BRCA1 dataset is small and runs within Colab's free-tier limits (~30 min total).
- For larger datasets (WES/WGS), consider Colab Pro or running locally.
- This pipeline is based in part on the [RSG Brazil 2026 course material](https://github.com/oncogensus/Curso-RSG-Brazil).

---

## Author

**Gabrieli Bovi**
Bioinformatics | Genomics | Variant Analysis
🔗 [github.com/gabrielibovi-bioinfo](https://github.com/gabrielibovi-bioinfo)

---

## License

MIT License — see the [LICENSE](LICENSE) file for details.

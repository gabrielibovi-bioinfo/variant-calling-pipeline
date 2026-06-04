# 🧬 variant-calling-pipeline
Reproducible variant calling pipeline for Illumina paired-end sequencing data — from raw FASTQ to annotated VCF, with two parallel calling approaches (GATK HaplotypeCaller and FreeBayes).

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
    │
    ├── [Step 2] Quality Control          → FastQC + MultiQC
    │
    ├── [Step 3] Trimming                 → fastp (Q≥20, auto adapter detection)
    │
    ├── [Step 4] Alignment                → BWA-MEM → sorted, indexed BAM
    │
    ├── [Step 5A] Variant Calling — GATK  → HaplotypeCaller → GVCF → VCF
    │                                       VariantFiltration (QD, FS, MQ)
    │
    ├── [Step 5B] Variant Calling — Free  → FreeBayes → bcftools filter (QUAL≥20)
    │
    ├── [Step 6] Annotation               → bcftools annotate + ClinVar mini
    │
    └── [Step 7] Summary Statistics       → bcftools stats (SNP/INDEL counts, Ts/Tv)
```


---

## Repository Structure

```
variant-calling-pipeline/
├── variant_calling_pipeline.ipynb   # Main notebook (Google Colab)
├── README.md
├── .gitignore
└── LICENSE
```

> Input data (FASTQ files, reference genome) are downloaded automatically inside the notebook. No local files are needed.

---

## Requirements

- **Google Colab** (recommended — free, no local installation needed)
- Or: Linux with [Mamba](https://mamba.readthedocs.io/) installed

### Tools installed automatically by the notebook

| Tool | Version | Conda environment |
|------|---------|-------------------|
| condacolab | latest | base |
| entrez-direct | latest | `entrez_ncbi` |
| FastQC | ≥0.12 | `quality` |
| MultiQC | ≥1.14 | `quality` |
| fastp | ≥0.23 | `quality` |
| BWA | ≥0.7 | `bwa` |
| Samtools | ≥1.17 | `samtools` |
| GATK4 | ≥4.4 | `gatk` |
| FreeBayes | ≥1.3 | `freebayes` |
| bcftools | ≥1.17 | `bcftools` |

---

## Usage

### Google Colab (recommended)

1. Open [Google Colab](https://colab.research.google.com/)
2. Upload `variant_calling_pipeline.ipynb` or open directly from GitHub
3. Run cells sequentially from top to bottom
4. All tools and data are downloaded automatically

> ⚠️ Run **Step 0** (condacolab setup) first and wait for the kernel to restart before continuing.

### Local (Linux / WSL2)

If running locally, replace `!mamba run -n <env>` calls with `conda activate <env>` and run each step in the terminal. The logic is identical.

---

## Input Data

| File | Source | Description |
|------|--------|-------------|
| `chr17_GRCh38.fasta` | NCBI Entrez (NC_000017.11) | Reference genome — chromosome 17 |
| `BRCA1_WT_R1/R2.fq.gz` | [oncogensus/Curso-RSG-Brazil](https://github.com/oncogensus/Curso-RSG-Brazil) | Wild-type control |
| `BRCA1_185delAG_R1/R2.fq.gz` | same | Pathogenic deletion variant |
| `BRCA1_c.5266dupC_R1/R2.fq.gz` | same | Pathogenic duplication variant |
| `clinvar_mini.vcf.gz` | same | ClinVar mini database for annotation |

---

## Output Structure

```
├── fastqc/               # Per-sample FastQC HTML reports
├── multiqc/              # Aggregated MultiQC HTML report
├── trimmed/              # Trimmed FASTQ files (fastp)
├── aligned/              # Sorted and indexed BAM files
├── variants/
│   ├── gatk/             # GATK HaplotypeCaller results
│   │   ├── *.g.vcf.gz    # Per-sample GVCFs
│   │   ├── cohort.vcf.gz # Joint-genotyped VCF
│   │   └── cohort_filtered.vcf.gz
│   └── freebayes/        # FreeBayes results
│       ├── cohort_freebayes.vcf.gz
│       └── cohort_freebayes_filtered.vcf.gz
└── annotated/            # VCF files annotated with ClinVar
    ├── cohort_gatk_annotated.vcf
    └── cohort_freebayes_annotated.vcf
```

---

## Filtering Parameters

| Parameter | Value | Tool |
|-----------|-------|------|
| Min base quality (trimming) | Q ≥ 20 | fastp |
| Min mapping quality | 20 | FreeBayes |
| GATK QD filter | < 2.0 | VariantFiltration |
| GATK FS filter | > 60.0 | VariantFiltration |
| GATK MQ filter | < 40.0 | VariantFiltration |
| FreeBayes QUAL filter | ≥ 20 | bcftools filter |
| Variant types kept | SNPs + INDELs | bcftools view |

---

## GATK vs FreeBayes: Key Differences

| Feature | GATK HaplotypeCaller | FreeBayes |
|---------|---------------------|-----------|
| Calling model | Local re-assembly + HMM | Bayesian allele frequency model |
| Multi-sample workflow | GVCF → CombineGVCFs → GenotypeGVCFs | Direct multi-BAM input (one command) |
| Speed | Slower | Faster |
| GVCF intermediate | Required | Not needed |
| Somatic/low-frequency variants | Not recommended | Flexible |
| Standard in clinical pipelines | ✅ Yes | Less common |
| Reference | [GATK Best Practices](https://gatk.broadinstitute.org) | [FreeBayes GitHub](https://github.com/freebayes/freebayes) |

---

## Tools and References

| Tool | Reference |
|------|-----------|
| FastQC | [Andrews, 2010](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) |
| MultiQC | [Ewels et al., 2016](https://doi.org/10.1093/bioinformatics/btw354) |
| fastp | [Chen et al., 2018](https://doi.org/10.1093/bioinformatics/bty560) |
| BWA | [Li & Durbin, 2009](https://doi.org/10.1093/bioinformatics/btp324) |
| Samtools | [Danecek et al., 2021](https://doi.org/10.1093/gigascience/giab008) |
| GATK4 | [Van der Auwera & O'Connor, 2020](https://doi.org/10.1002/cpz1.20) |
| FreeBayes | [Garrison & Marth, 2012](https://arxiv.org/abs/1207.3907) |
| bcftools | [Danecek et al., 2021](https://doi.org/10.1093/gigascience/giab008) |
| ClinVar | [Landrum et al., 2018](https://doi.org/10.1093/nar/gkx1153) |

---

## Notes

- The notebook was designed and tested on **Google Colab** (free tier).
- Run **Step 0** first and wait for the Colab kernel to restart automatically before proceeding.
- Each tool runs in its own isolated mamba environment — this avoids dependency conflicts.
- `%%bash` cells execute multi-line shell blocks and require the entire cell to run at once.
- The BRCA1 dataset is small and runs comfortably within Colab's free-tier limits (~30 min total).
- For larger datasets (WES/WGS), consider using Colab Pro or running locally.

---

## Author

**Gabriel Ibovi**
Bioinformatics | Genomics | Variant Analysis
🔗 [github.com/gabrielibovi-bioinfo](https://github.com/gabrielibovi-bioinfo)

---

## License

This pipeline is based in part on the [RSG Brazil 2026 course material](https://github.com/oncogensus/Curso-RSG-Brazil) (Oncogensus project) and is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
EOF

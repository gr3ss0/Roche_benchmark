# Benchmarking Roche Axelios GIAB Data with Hap.py

## Project Overview
Introduce the new Roche Axelios technology and demonstrate its variant-calling performance using Genome in a Bottle (GIAB) reference data.

## Data Sources
In March 2026, Roche published 30x coverage GIAB VCFs, following their initial presentation in a September 2025 webinar. 
- **VCFs Data:** [Webinar GIAB XOOS VCFs 30x Duplex](https://web.sbxdata.kamino.platform.navify.com/files/030626-Webinar-GIAB-XOOS-VCFs-30x-Duplex/) *(registration required)*
- **Webinar:** [SBX Data Analysis Webinar](https://diagnostics.roche.com/global/en/events/sbx-d-data-analysis-webinar.html#video)

## Tools and Configuration
### Hap.py (Illumina)
[Hap.py GitHub Repository](https://github.com/Illumina/hap.py)
- **Truth Data:** NIST v4.2.1 from [NCBI](https://ftp-trace.ncbi.nlm.nih.gov/ReferenceSamples/giab/release/)
- **Reference Genome:** hg38 from [UCSC](https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/)
- **Additional Flags:**
  - `--engine=vcfeval`

---

## Theoretical Background
### SBX Technology 
**S**equencing **b**y **e**xpansion (SBX, 2007) uses DNA as a template to create a surrogate molecule called an **Xpandomer**.
- **Reported Speed:** 7 genomes per hour at >30x depth.
- **Flexible Length:** Ranges from 100 bp to over 500 bp, as reported in the webinar.

#### Xpandomer Anatomy:
- **Translocation Control Element (TCE):** Holds or stops a sequence in the barrel so the current can be measured. A high-voltage pulse releases the Xpandomer, precisely advancing it to the next TCE.
- **Reporter:** Designed to have four distinct ion current level responses, but with similar voltage-pulse translocation control.
- **Acid-Cleavable Bond:** cleaved before sequencing
- **dNTP**
- **Enhancer:** Polyamines, which are necessary for incorporation by the Xp Synthase.

![XNTP Anatomy](images/anatomy_XNTP.png)

### Duplex and Simplex Modes
Double-stranded DNA is connected by a hairpin, unraveled, and sequenced simultaneously. This intramolecular consensus calling results in three possible quality scores:
- **75% yield `H`:** Concordant basepair (double coverage). *Note: Only these are used in downstream analysis.*
- **24% yield `7`:** Simplex mode (single coverage).
- **1% yield `&`:** Discordant basepair (mismatch).

![Duplex Mode](images/duplex.png)

### SBX-Fast
Ultra-rapid application, amplificaiton-free solo run for 20 minutes with 30x
![alt text](images/SBX-Fast.png)
#### Quirks of Roche Benchmark Pre-filtering
When counting ALT/REF in homopolymer or tandem repeat (HP/TR) regions, reads are filtered out if:
- They do not span the entire region in DUPLEX mode.
- There is a discordant base.
- **Pan-genome aware consensus calling:** If the pangenome supports one mate from a discordant basepair, that mate is chosen and assigned a higher quality score.

---

## Tool Compatibility
Due to single-end reads and variable insert sizes, the following tools are **not recommended**:
- Picard MarkDuplicates (reads of different lengths ending at the same position are incorrectly marked as duplicates)
- samtools pileup
- FastQC
- Standard sequencing metrics tools
- UMI-based consensus callers

The following tools **are compatible** with some optimization:
- IGV (>2.19.6)
- HaplotypeCaller
- Mutect2
- DeepVariant
- GRIDSS or Manta
- ExpansionHunter

*Note: Ultimately, the XOOS provides solutions for standard analysis.*

---

## SBX Analysis Strategy
- **Multiplexing:** 4 library pools run in parallel.

The sequencer executes several steps in parallel to the flow run, providing:
- Standard raw reads
- Consensus reads
- Mapped/Aligned reads 
  - Sorted BAM files are available in under 1 minute after completion (for an SBX solo run).
  - Mapping to the pangenome is handled by XOOS (Giraffe-based).
- **INTRA- vs. INTER-molecular Consensus Quality Scores:**
  - Intra-molecular scores have three levels: `H`, `7`, and `&`.
  - Inter-molecular consensus is estimated by the Read Collapser.

### XOOS 
#### Modules

| Module | Description |
|--------|-------------|
| **Demux** | Demultiplexes samples and trims adapters from raw reads; for duplex chemistries also performs consensus base calling. |
| **Read Collapser** | Clusters reads by genomic position and/or UMI, then either marks duplicates or generates consensus reads. Supports germline WGS and target enrichment (TE). |
| **Alignment Metrics** | Computes alignment and quality metrics from aligned SBX reads. |
| **Small Variant Caller**| Filters and re-genotypes small variants (SNVs and short indels) using a machine-learning model. |
| **Copy Number Caller** | Calls copy-number events and estimates purity/ploidy. |
| **STR Caller** | Genotypes and detects repeat expansions in short tandem repeats (STRs). |
| **Pan-Genome Consensus Caller** | Resolves duplex discordant bases using a pan-genome reference. |
| **Tumor Fraction Estimator** | Estimates tumor fraction and detects sample contamination. |

XOOS utilizes an optimized version of **DeepVariant**:
- Sequencing read data is converted into an image-like representation.
- **Optimization for SBX-D and SBX-Fast:**
  - Utilizes a De Bruijn graph to identify potential haplotypes. The inclusion of any 1- or 2-base pair insertion requires at least 8% read evidence.
  - The parameter `ws_min_base_quality` used for realignment is increased from 20 to 25.
  - Based on Release v.10.

---

## Benchmarking Results
### Inhouse 
Downsampled 30x

| Sample | Variant Type | Recall | Precision | F1 Score |
| :--- | :--- | :---: | :---: | :---: |
| **HG001** | SNP   | 99.74% | 99.87% | 99.80% |
| **HG001** | INDEL | 99.46% | 99.69% | 99.57% |
| **HG002** | SNP   | 99.63% | 99.87% | 99.75% |
| **HG002** | INDEL | 99.39% | 99.68% | 99.54% |
| **HG003** | SNP   | 99.59% | 99.83% | 99.71% |
| **HG003** | INDEL | 99.34% | 99.60% | 99.47% |
| **HG004** | SNP   | 99.60% | 99.88% | 99.74% |
| **HG004** | INDEL | 99.45% | 99.70% | 99.57% |
| **HG005** | SNP   | 99.61% | 99.88% | 99.74% |
| **HG005** | INDEL | 99.57% | 99.78% | 99.67% |
| **HG006** | SNP   | 99.60% | 99.87% | 99.73% |
| **HG006** | INDEL | 99.47% | 99.74% | 99.60% |
| **HG007** | SNP   | 99.58% | 99.83% | 99.71% |
| **HG007** | INDEL | 99.45% | 99.66% | 99.55% |

### Claimed by Roche
![Roche Benchmarking Results](images/roche-results.png)

---

## Further Resources
- [bioRxiv Preprint on SBX Technology (March 2025)](https://www.biorxiv.org/content/10.1101/2025.02.19.639056v1.full.pdf)
- [Poster: Duplex Sequencing by Expansion](https://sequencing.roche.com/content/dam/diagnostics_microsites/sequencing/master-blueprint/en/resources/pdfs/posters/redefining-nanopore-sequencing-w-duplex-sequencing-by-expansion-SBX-D.pdf)
- [Roche DeepVariant Whitepaper (2026)](https://diagnostics.roche.com/content/dam/acadia/whitepaper/399/588/SBX002_Google%20DV%20white%20paper_v1-0-0326.pdf)
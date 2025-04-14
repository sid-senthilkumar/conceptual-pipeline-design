# Conceptual Design: Functional Genomics Analysis Pipeline

**Author:** Lynn Cole
**Date:** October 26, 2023
**Context:** Proposed analysis workflow for high-throughput functional genomics screens (e.g., CRISPR or TnSeq) at Pioneer Labs.

## 1. Overview

**Goal:** To systematically process raw Next-Generation Sequencing (NGS) data from functional genomics screens, identify genetic elements (e.g., guide RNAs, transposon insertions) conferring a desired phenotype, and generate a prioritized list of candidate genes for further investigation and validation.

**Pipeline Philosophy:** This pipeline is designed with modularity, reproducibility, and robust Quality Control (QC) as core principles. Each step is conceptually distinct, allowing for flexibility in tool choice and independent execution/validation. Proper metadata tracking and parameter logging are assumed throughout.

**Input:** Raw FASTQ sequencing files, Sample Metadata Sheet.
**Output:** Prioritized, annotated list of candidate genes associated with the phenotype of interest.

---

## 2. Pipeline Stages

### Stage 1: Raw Data Ingest & Initial QC

*   **Input:**
    *   Raw FASTQ files (paired-end or single-end) from sequencing facility.
    *   Metadata file mapping sample IDs to experimental conditions, replicates, timepoints, library used, etc.
*   **Process:**
    *   **Integrity Check:** Verify file existence and MD5 checksums to ensure data transfer integrity.
    *   **Quality Assessment:** Utilize standard tools (conceptually, e.g., FastQC) to assess per-base quality scores, GC content, adapter contamination, duplication rates, and sequence length distribution.
    *   **(Optional) Read Trimming/Filtering:** Remove adapter sequences and low-quality bases/reads if QC indicates necessity (using tools like Trimmomatic or fastp conceptually).
    *   *Rationale:* Establishes baseline data quality. Prevents "garbage in, garbage out." Identifies potential sequencing issues early.
*   **Output:**
    *   QC reports (e.g., HTML files) for each sample.
    *   (Optional) Trimmed/Cleaned FASTQ files.
    *   Log file summarizing QC metrics and any filtering performed.

### Stage 2: Read Alignment / Mapping

*   **Input:**
    *   Cleaned (or raw) FASTQ files.
    *   Appropriate Reference Sequence:
        *   For CRISPR screens: Library file mapping guide RNAs (gRNAs) to target sequences/genes.
        *   For TnSeq: Reference genome of the host organism (*E. coli*, *B. subtilis*).
        *   *Note:* Choice depends critically on screen type. Reference must be accurate and complete.
*   **Process:**
    *   Align reads to the reference using a suitable aligner (e.g., Bowtie2, BWA, or potentially specialized gRNA mapping tools). Parameters chosen based on read length, reference type, and need for sensitivity vs. speed.
    *   *Rationale:* Determines the origin (target gRNA, genomic insertion site) of each sequencing read.
*   **Output:**
    *   Alignment files (e.g., sorted BAM format with index).
    *   Alignment statistics (overall mapping rate, reads mapping uniquely, etc.) per sample.

### Stage 3: Quantification / Feature Counting

*   **Input:**
    *   Alignment files (BAM).
    *   Feature Definition File:
        *   For CRISPR screens: List of expected gRNA sequences.
        *   For TnSeq: Gene annotations (GTF/GFF) or defined genomic regions.
*   **Process:**
    *   Count reads unambiguously assigned to each defined feature (gRNA sequence, gene locus, etc.). Tools like `featureCounts` or custom scripts might be used conceptually. Handle multi-mapping reads appropriately based on chosen strategy (e.g., discard, allocate fractionally).
    *   *Rationale:* Generates the fundamental data matrix representing the abundance of each screened element in each sample.
*   **Output:**
    *   Raw Count Matrix: A table (e.g., CSV/TSV file) where rows are features (gRNAs, genes) and columns are samples, cells contain raw read counts.

### Stage 4: Normalization & Data QC

*   **Input:**
    *   Raw Count Matrix.
    *   Sample Metadata.
*   **Process:**
    *   **Normalization:** Apply normalization methods to account for variations in sequencing depth and library composition bias (e.g., counts per million (CPM), DESeq2/edgeR size factors conceptually).
    *   **Data Exploration & QC:**
        *   Visualize library size distributions.
        *   Perform dimensionality reduction (PCA, MDS) on normalized counts to visually inspect sample relationships (replicates clustering, separation by condition).
        *   Calculate sample correlation matrices.
    *   **(Optional) Filtering:** Remove low-count features or outlier samples identified during QC.
    *   *Rationale:* Ensures fair comparison between samples. Identifies potential batch effects or experimental issues before statistical modeling. Like checking the elevation map of our city district for unexpected dips or spikes.
*   **Output:**
    *   Normalized Count Matrix.
    *   QC plots (PCA, correlation heatmap, library size distribution).
    *   Log of normalization factors and filtering decisions.

### Stage 5: Differential Abundance / Hit Calling

*   **Input:**
    *   Normalized Count Matrix (or Raw Counts + Normalization factors depending on model).
    *   Experimental Design information (from Metadata).
*   **Process:**
    *   Employ appropriate statistical models to test for significant differences in feature abundance between experimental groups (e.g., using negative binomial models like DESeq2/edgeR, or specialized screen analysis tools like MAGeCK/BAGEL conceptually). Account for biological variability through replicate data.
    *   Correct for multiple hypothesis testing (e.g., using Benjamini-Hochberg FDR).
    *   *Rationale:* Statistically identifies features whose abundance changes significantly due to the experimental perturbation or selection pressure. This is where potential 'hits' emerge from the noise.
*   **Output:**
    *   Results Table (per feature): Contains feature ID, base mean abundance, log2 Fold Change, standard error, test statistic, p-value, adjusted p-value (FDR).

### Stage 6: Gene-Level Aggregation & Annotation

*   **Input:**
    *   Feature-level Results Table.
    *   Mapping file connecting features (gRNAs, insertion sites) to target genes.
*   **Process:**
    *   **Aggregation:** Combine feature-level statistics (e.g., p-values, fold changes) into a single gene-level score. Methods range from selecting the most significant feature per gene, averaging effects, to more sophisticated rank-based aggregation (e.g., RRA algorithm used in MAGeCK).
    *   **Annotation:** Enrich the gene list with functional information: Gene Ontology (GO) terms, KEGG pathways, known protein domains, homology to other organisms (using resources like DAVID, PANNZER2, UniProt, EggNOG conceptually).
    *   *Rationale:* Shifts focus from individual screening elements to the biological target (gene). Adds biological context crucial for interpretation.
*   **Output:**
    *   Gene-level Results Table: Contains gene ID, aggregated statistics (score, p-value), and functional annotations.

### Stage 7: Prioritization & Interpretation

*   **Input:**
    *   Annotated Gene-level Results Table.
*   **Process:**
    *   **Rank & Filter:** Sort genes based on combined evidence: statistical significance (gene-level FDR), magnitude and direction of effect (gene-level fold change), consistency across features targeting the same gene (if applicable), and biological plausibility (relevant annotations). Define clear thresholds.
    *   **Visualization:** Generate plots like volcano plots (fold change vs. significance) to visualize results. Consider pathway enrichment analysis on hit lists.
    *   **Review:** Manually review top candidates in the context of the specific experiment and Pioneer's goals.
    *   *Rationale:* Distills the results into a actionable list of high-confidence candidates for experimental follow-up. This step bridges computation with biological insight.
*   **Output:**
    *   Final Prioritized Gene List: Top N candidates with supporting statistics and annotations.
    *   Summary visualizations (volcano plot, pathway enrichment results).

---

## 3. Key Considerations

*   **Modularity:** Design each stage as a self-contained unit (conceptually a script or function) with defined inputs/outputs. This facilitates testing, reuse, and replacement of specific tools.
*   **Reproducibility:** Track software versions, parameters used for each step, and reference database versions. Containerization (e.g., Docker) is recommended for deployment.
*   **Scalability:** Ensure tools and data structures can handle large datasets typical of high-throughput screens. Consider parallelization options.
*   **Parameterization:** Allow key parameters (e.g., alignment settings, statistical thresholds, normalization methods) to be configurable.
*   **Documentation:** Thoroughly document the pipeline logic, tool choices (even conceptual ones), and parameter settings.

---

## 4. Potential Next Steps / Implementation Notes

*   Translate this conceptual design into a workflow management system (e.g., Snakemake, Nextflow) for automated execution.
*   Select specific bioinformatics tools for each stage based on data characteristics and available resources.
*   Develop robust error handling and logging within each module.
*   Integrate QC checks that can automatically flag issues or halt the pipeline if necessary.

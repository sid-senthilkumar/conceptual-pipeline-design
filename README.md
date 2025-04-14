# conceptual-pipeline-design
A conceptual functional genomics pipeline for pioneer labs

```mermaid
graph TD
    A[Start: Raw FASTQ Data + Metadata] --> B(Stage 1: Ingest & QC);
    B --> C{QC Passed?};
    C -- Yes --> D[Stage 2: Alignment/Mapping];
    C -- No --> E[Review/Flag Data];
    D --> F[Stage 3: Quantification/Feature Counting];
    F --> G[Stage 4: Normalization & Data QC];
    G --> H{Sample QC Passed?};
    H -- Yes --> I[Stage 5: Differential Abundance Analysis];
    H -- No --> J[Review/Exclude Samples];
    I --> K[Stage 6: Gene-Level Aggregation & Annotation];
    K --> L[Stage 7: Prioritization & Interpretation];
    L --> M[End: Prioritized Gene List];

    %% Styling Nodes (Optional - depends on Mermaid renderer)
    % classDef default fill:#f9f,stroke:#333,stroke-width:2px;
    % classDef decision fill:#ccf,stroke:#333,stroke-width:2px;
    % class A,M default;
    % class C,H decision; 

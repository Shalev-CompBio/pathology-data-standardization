# Pathology Sample-Level Clinical Extraction Pipeline

**Version**: 1.0.0  
**Environment**: Python 3.11+ (Pandas)  
**Input Format**: Unstructured Pathology Excel Reports (`.xlsx`)  
**Primary Output**: Research-ready CSV + intermediate artifacts (`.csv`)  
**License**: Proprietary / Internal Use Only  

---

## 1. Executive Summary

The **Pathology Sample-Level Clinical Extraction Pipeline** is an ETL + NLP workflow that transforms raw pathology test rows into a compact, sample-level research dataset. It addresses the common clinical data problem where key attributes (malignancy, tumor site, morphology, grade) are embedded in free-text or split across multiple rows.

The pipeline is delivered as a single Jupyter Notebook for portability and reproducibility. It uses conservative, traceable transformations with deterministic output files (timestamped) and basic validation to ensure fitness for downstream clinical queries.

---

## 2. Directory Structure

```text
Psipas/
|-- input/
|   |-- pathology_tests.xlsx
|   |-- terminology_mapping.csv
|-- output/
|   |-- pathology_sample_level_clinical_extraction_pipeline/
|       |-- pathology_tests_<TIMESTAMP>.csv
|       |-- pathology_tests_summary_<TIMESTAMP>.csv
|       |-- pathology_samples_<TIMESTAMP>.csv
|       |-- pathology_research_<TIMESTAMP>.csv
|       |-- terminology_mapping_audit_<TIMESTAMP>.csv
|       |-- terminology_unmapped_<TIMESTAMP>.csv
|-- scripts/
|   |-- pathology_sample_level_clinical_extraction_pipeline.ipynb
|   |-- README.md
```

---

## 3. Input Data Schema

**Source**: `input/pathology_tests.xlsx` (first sheet is used).

| Column | Description |
| :--- | :--- |
| `Patient_ID` | Anonymized patient identifier |
| `Code` | Source code for the record (often LOINC or internal code) |
| `Order_Num` | Order identifier |
| `Order_Code` | Order code |
| `Entry_Time` | Timestamp of record entry |
| `Comment_Type` | Type of comment (clinical diagnosis vs other text) |
| `Data` | Free-text content associated with `Comment_Type` |
| `SampleID` | Sample identifier |
| `SampleNumber` | Sub-sample identifier |
| `Pathology_Procedure` | Procedure name |
| `Test_Code` | Test code |
| `Test_Name` | Test name |
| `Result` | Test result text |

**Notes**:
- All fields are loaded as strings (`dtype=str`) and empty cells remain empty strings (not NaN).
- The notebook assumes the first Excel sheet contains the data.
- `terminology_mapping.csv` is required for terminology standardization (see Section 5).

---

## 4. Pipeline Stages (What Actually Happens)

### Stage 1: Ingestion & Standardization
**Goal**: Create an immutable CSV copy of the raw Excel sheet and verify data integrity.

Key behaviors:
- Reads the first sheet in `input/pathology_tests.xlsx` with `dtype=str` and `keep_default_na=False`.
- Writes `pathology_tests_<TIMESTAMP>.csv` using `utf-8-sig` encoding.
- Verifies **shape**, **column order**, and **cell-level equality** by reloading the CSV.
- Computes a content hash for additional confidence.

### Stage 2: Dataset Profiling (Summary Report)
**Goal**: Provide a quick data quality profile before restructuring.

Output: `pathology_tests_summary_<TIMESTAMP>.csv`, which includes:
- **Dataset-level rows**: `source_file`, `sheet_used`, `export_timestamp`, `rows`, `columns`, `total_cells`, `empty_cells_count`, `empty_cells_percent`, `duplicate_rows_count`, `estimated_memory_bytes`.
- **Column-level rows**: `column_name`, `position_index`, `inferred_type_hint`, `non_empty_count`, `empty_count`, `empty_percent`, `unique_non_empty_count`, `unique_percent_non_empty`, `most_frequent_value`, `most_frequent_count`, `sample_values`.

### Stage 2B: Sample-Level Restructuring (Pivot)
**Goal**: Convert test-level rows into a single row per sample.

Mechanics:
- Groups by `(SampleID, SampleNumber)`.
- Preserves metadata: `Patient_ID`, `Code`, `Order_Num`, `Order_Code`, `Entry_Time`, `SampleID`, `SampleNumber`, `Pathology_Procedure`.
- Aggregates distinct `Comment_Type` values into `comment_types_raw` (pipe-delimited).
- Aggregates `Data` **only for clinical-diagnosis rows** into `Clinical_Diagnosis`.
- Aggregates remaining `Data` values into `Additional_Diagnostic_Text`.
- Creates a `Result_<Test_Name>` column for every distinct `Test_Name`.
- Creates a `TestCode_<Test_Name>` column (first observed code).
- If a sample has multiple results for the same test, values are concatenated with ` | ` and de-duplicated.

Output: `pathology_samples_<TIMESTAMP>.csv`.

### Stage 3: Research Dataset Mapping (Feature Extraction)
**Goal**: Produce the minimal structured dataset required for clinical analysis.

#### Malignancy (`is_malignant`)
A priority waterfall is used (negations override keyword hits):
1. **Structured**: `Result_MALIGNANT` values (including English/Hebrew variants).
2. **Histology text**: keyword search in `Result_Surgical pathology,microscopic examination`.
3. **Clinical text**: keyword search in `Clinical_Diagnosis`.

Output encoding is normalized to: **`TRUE` / `FALSE` / `UNKNOWN`**.

#### Tumor Site, Morphology, Grade
- **Site**: SNOMED body structure pattern first, then histology header regex, then a small bilingual dictionary fallback.
- **Morphology**: SNOMED morphologic abnormality pattern first, then histology regex, then bilingual dictionary fallback.
- **Grade**: Histology regex (FIGO, Gleason, grade/group patterns).

#### Terminology Standardization (SNOMED CT)
- SNOMED-like strings are parsed into `(term, category)` pairs.
- Terms are mapped using `input/terminology_mapping.csv`.
- Standardized fields are populated **only when a mapping exists**.
- Unmapped terms are exported for review.
- Mapping matches are **case-insensitive exact matches** on `source_term` + `semantic_category`.
- Standardized outputs are currently populated only from `body structure` and `morphologic abnormality` categories.

Output: `pathology_research_<TIMESTAMP>.csv` with the following schema:

| Column | Description |
| :--- | :--- |
| `patient_id` | Patient identifier |
| `sample_id` | Sample identifier |
| `sample_number` | Sub-sample identifier |
| `entry_time` | Entry timestamp |
| `order_id` | Order number |
| `order_code` | Order code |
| `clinical_diagnosis_text_raw` | Clinical diagnosis text (from `Clinical_Diagnosis`) |
| `macroscopic_text_raw` | Macroscopic text (`Result_MACROSCOPIC`) |
| `histology_text_raw` | Histology text (`Result_Surgical pathology,microscopic examination`) |
| `is_malignant` | `TRUE` / `FALSE` / `UNKNOWN` |
| `malignancy_source` | `structured_field` / `histology_text` / `clinical_text` / empty |
| `tumor_site_text_raw` | Extracted tumor site (raw text) |
| `tumor_site_standardized` | SNOMED CT code (from mapping file) |
| `tumor_morphology_text_raw` | Extracted morphology (raw text) |
| `tumor_morphology_standardized` | SNOMED CT code (from mapping file) |
| `tumor_grade_text_raw` | Extracted grade (raw text) |
| `snomed_codes_raw` | Raw SNOMED text from `Result_SNOMED` |
| `additional_diagnostic_text_raw` | Non-diagnosis text from `Additional_Diagnostic_Text` |
| `pathology_procedure` | Source procedure |
| `source_code` | Source code from input |

### Stage 4: Clinical Query Validation
**Goal**: Demonstrate that 17 required clinical queries can be answered using structured fields.

The notebook prints a validation report to stdout, including:
- Patient/sample counts
- Malignancy counts
- Site and morphology distributions
- Grade extraction rates
- Provenance metrics (structured vs free-text)

**Note**: The notebook currently prints the validation report but does **not** write it to disk, even though a `validation_report_<TIMESTAMP>.txt` path is defined in code.

---

## 5. Terminology Selection & Field Mapping

### 5.1 Chosen Terminology (and Why)
**Selected terminology**: **SNOMED CT** (primary), with an oncology-oriented perspective.

**Rationale**:
- **Coverage**: SNOMED CT provides rich concepts for tumors, morphology, and anatomical sites.
- **Adoption**: Widely used in clinical systems and compatible with OHDSI vocabularies (Athena).
- **Suitability**: Supports both site (body structure) and morphology (morphologic abnormality) patterns, which align with the raw SNOMED-style strings observed in the input.

**Current implementation status**:
- The pipeline parses SNOMED-like terms and **maps them through a controlled CSV** (`terminology_mapping.csv`).
- Standardized fields are filled when a mapping exists; otherwise they remain blank.
- Mapping coverage is auditable via `terminology_mapping_audit_<TIMESTAMP>.csv` and `terminology_unmapped_<TIMESTAMP>.csv`.

**Required columns in `terminology_mapping.csv`**:
- `source_term`
- `semantic_category`
- `standard_concept_id`
- `snomed_ct_code`
- `concept_name`
- `vocabulary_id`

**Matching rules**:
- Matching is done on **`source_term` + `semantic_category`** after trimming and lowercasing.
- If a row exists in the mapping file but codes are blank, the standardized fields remain empty.
- Rows with categories other than `body structure` / `morphologic abnormality` are logged but not used to fill standardized outputs.

**Excel caution**:
- If you edit the mapping file in Excel, set code columns to **Text** to avoid scientific notation (e.g., `1.29E+08`).

### 5.2 Field Mapping Documentation (Minimal Dataset)

| Output Field | Source(s) | Extraction Logic | Coding / Standardization |
| :--- | :--- | :--- | :--- |
| `is_malignant` | `Result_MALIGNANT`, histology text, clinical diagnosis | Priority waterfall with negation handling | Encoded as `TRUE` / `FALSE` / `UNKNOWN` |
| `tumor_site_text_raw` | `Result_SNOMED`, histology text, clinical diagnosis | SNOMED pattern → regex header → bilingual dictionary | Raw text (standardized placeholder exists) |
| `tumor_morphology_text_raw` | `Result_SNOMED`, histology text, clinical diagnosis | SNOMED pattern → histology regex → bilingual dictionary | Raw text (standardized placeholder exists) |
| `tumor_grade_text_raw` | Histology text | Regex patterns (FIGO, Gleason, grade group) | Raw text |
| `clinical_diagnosis_text_raw` | `Data` filtered by clinical `Comment_Type` | Filter + concatenate | Raw text |
| `additional_diagnostic_text_raw` | Remaining `Data` rows | Concatenate | Raw text |

---

## 6. Approach & Methodology

### 6.1 Resources Used
- **OHDSI Athena** and SNOMED CT documentation to align terminology and patterns.
- **Pathology terminology references** (general oncology/pathology literature).
- **LLM tools (optional, offline)** for translation or keyword expansion (not used for automated coding in the pipeline itself).

### 6.2 Handling Ambiguity
- **Negations override keywords** (e.g., “אין עדות לממאירות” → `FALSE`).
- If no reliable signal is found, malignancy is set to **`UNKNOWN`**.
- Site/morphology extraction falls back to a **small bilingual dictionary** when regex/SNOMED patterns fail.

### 6.3 Tools and Techniques
- Deterministic ETL (pandas), strict `dtype=str`.
- Regex + keyword matching (English + Hebrew).
- Bilingual dictionaries for site/morphology fallback.
- Explicit terminology mapping via `terminology_mapping.csv` (SNOMED CT codes).
- Validation queries to check fitness for research use.

### 6.4 Challenges Encountered
- **Multilingual text** (Hebrew + English).
- **Inconsistent terminology** across fields.
- **Mixed structured/unstructured content** within the same sample.

### 6.5 Rationale for Minimal Fields
- The dataset is intentionally minimal and focused on fields required to answer clinical queries about:
  **tumor behavior, site, morphology, and grade**.
- Additional fields are kept raw for traceability and audit.

### 6.6 Baseline Data Profile (Example Run)
The following figures were observed on the **provided sample dataset** during an initial run (January 29, 2026).  
These numbers are **informational only** and may change after logic updates or when using different datasets.

- **Raw input size (Stage 1)**: 81 rows × 13 columns.
- **Empty-cell rate (Stage 1)**: ~4.18% empty cells.
- **Research dataset size (Stage 3)**: 15 rows × 20 columns (one row per sample).
- **Malignancy classification coverage**: 100% classified in that run (8 TRUE, 7 FALSE).
- **Extraction coverage (Stage 3)**:
  - Tumor site: 11/15
  - Morphology: 7/15
  - Grade: 2/15 (low due to specific FIGO/Gleason regex requirements)

---

## 7. Bonus: LLM-Assisted Automation Proposal (Optional)

If scaling to thousands of reports, an LLM-assisted pipeline could be used as a **secondary extraction layer**:

**Prompt strategy (high level)**:
- Provide the raw histology + clinical text.
- Ask for **structured JSON** with fixed keys (site, morphology, behavior, grade, negation flags).
- Enforce schema with strict validators.

**Validation steps**:
- Automatic rule-based checks (e.g., negation conflicts, missing required fields).
- Cross-check with SNOMED / ICD‑O dictionaries.
- Confidence scoring and thresholding.

**Human review**:
- Required for low-confidence outputs, conflicting evidence, or rare morphologies.

---

## 8. Usage Instructions

### Prerequisites
- **Python 3.11+**
- **Pandas** (`pip install pandas`)
- Read/write permissions to the project folder

### Run the Pipeline
1. Place the raw Excel file at `input/pathology_tests.xlsx`.
2. Prepare `input/terminology_mapping.csv` with SNOMED CT mappings.
3. Open `scripts/pathology_sample_level_clinical_extraction_pipeline.ipynb` in VS Code or JupyterLab.
4. Execute **Run All**.
5. Collect outputs from `output/pathology_sample_level_clinical_extraction_pipeline/`.

**Tip**: If you edit `terminology_mapping.csv` in Excel, set code columns to **Text** to prevent scientific-notation corruption.

---

## 9. Troubleshooting & Customization

- **"Input file not found"**: Ensure `input/pathology_tests.xlsx` exists (not in `scripts/`).
- **Stage 1 verification failed**: Check disk space and encoding; Stage 1 expects byte-for-byte equality after CSV write/read.
- **Noisy clinical diagnosis field**: Adjust the `Comment_Type` filter in `pivot_sample_to_row` (Stage 2B).
- **Low malignancy extraction rate**: Edit the `MALIGNANT_KEYWORDS`, `BENIGN_KEYWORDS`, and `NEGATION_PHRASES` lists in the **Stage 3 Field Mapping Functions** cell.
- **Missing site/morphology**: Extend the regex patterns or the bilingual dictionaries in `extract_tumor_site` / `extract_tumor_morphology`.
- **Standardized fields empty**: Ensure `terminology_mapping.csv` contains matching `source_term` + `semantic_category` pairs for the extracted SNOMED terms.
- **Terminology file missing/empty**: Create `input/terminology_mapping.csv` and populate it; otherwise standardized fields will remain blank and audit will list all terms as unmapped.

---

## 10. Known Limitations

- Only the **first Excel sheet** is processed.
- Keyword/regex extraction is heuristic and may miss rare or non-standard phrasing.
- The bilingual dictionaries are intentionally small and require manual expansion for better coverage.
- Standardization coverage depends on the completeness of `terminology_mapping.csv`.
- Terms categorized as `finding` are audited but not used to populate standardized site/morphology fields by default.

---

**Developed by**: Psipas Data Engineering Team  
**Last Updated**: January 29, 2026

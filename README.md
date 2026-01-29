# Pathology Sample-Level Clinical Extraction Pipeline

**Assignment Submission**

## Assignment Context

This project was developed as part of an assignment focused on transforming unstructured pathology reports into a minimal, standardized dataset suitable for clinical research queries.
The emphasis was on conservative extraction, terminology standardization, and methodological transparency rather than full clinical completeness or production optimization.

---

## 1. Executive Summary

This pipeline transforms raw, unstructured pathology test data into a minimal, sample-level structured dataset suitable for clinical research.
Key clinical attributes such as malignancy, tumor site, morphology, and grade are often embedded in free text or spread across multiple rows; the pipeline consolidates and extracts these attributes while preserving raw text for traceability.

The output enables standardized clinical queries while maintaining conservative assumptions and auditability.

---

## 2. Input Data Overview

The input consists of an Excel file (`pathology_tests.xlsx`) containing mixed structured and unstructured pathology records, including free-text clinical diagnoses and histology descriptions in Hebrew and English.
Multiple rows may correspond to the same patient or sample, requiring restructuring to a single-row-per-sample representation.

---

## 3. Data Processing Overview

The pipeline performs the following high-level steps:

1. Load the raw Excel data as-is, preserving all values as strings.
2. Restructure the data from test-level rows to a single row per sample.
3. Extract a minimal set of clinically relevant fields.
4. Apply conservative terminology standardization where mappings exist.
5. Validate that the resulting dataset can support key clinical queries.

---

## 4. Sample-Level Restructuring

Test-level rows are grouped by `(SampleID, SampleNumber)` to produce a single consolidated row per sample.

During this process:

* Core metadata (patient ID, order information, procedure) is preserved.
* Clinical diagnosis text is aggregated separately from other diagnostic text.
* Test results are pivoted into structured columns.
* Multiple values for the same test are concatenated and de-duplicated.

This step produces one row per sample while retaining all relevant diagnostic context.

---

## 5. Terminology Selection & Field Mapping

### 5.1 Chosen Terminology

**Selected terminology**: **SNOMED CT**, accessed conceptually through the **OHDSI Athena** framework.

**Rationale**:

* **Coverage**: SNOMED CT provides comprehensive concepts for anatomical sites and tumor morphology.
* **Adoption**: Widely used in clinical systems and supported within the OHDSI ecosystem.
* **Suitability**: Well aligned with pathology and oncology data, including body structure and morphologic abnormality concepts observed in the source data.

SNOMED CT was therefore selected as the primary terminology for coding pathology findings.

---

### 5.2 Minimal Field Mapping

Only fields that could be consistently identified across samples were extracted.

* **Malignancy (`is_malignant`)**
  Derived using a priority-based approach combining structured fields and free-text signals, with explicit negation handling.
  Encoded as `TRUE`, `FALSE`, or `UNKNOWN`.

* **Tumor site**
  Extracted from SNOMED-like strings, histology headers, or a small bilingual dictionary.
  Stored as raw text, with optional SNOMED CT standardization when a mapping exists.

* **Tumor morphology**
  Extracted using SNOMED patterns, histology regex rules, or bilingual dictionary fallback.
  Stored as raw text, with optional SNOMED CT standardization.

* **Tumor grade**
  Extracted from histology text using FIGO, Gleason, and grade-group patterns.

* **Clinical diagnosis text**
  Preserved verbatim to maintain interpretability and auditability.

All terminology standardization relies on an explicit mapping file (`terminology_mapping.csv`).
If no mapping exists, standardized fields are left empty and the raw extracted text is retained.

---

## 6. Clinical Query Validation

The resulting dataset enables clinical research queries such as:

* Counting patients or samples with malignant tumors using the `is_malignant` field.
* Identifying bladder cancer cases by filtering tumor site fields (standardized or raw).
* Identifying kidney-related suspicious samples through site extraction.
* Counting benign tumors using malignancy classification.

Validation queries are executed within the notebook to confirm that these questions can be answered using the structured fields.

---

## 7. Approach & Methodology

The guiding principle of the approach was to extract only fields that could be consistently identified across all samples, while preserving raw text for auditability.

### Resources Used

* OHDSI Athena and SNOMED CT documentation.
* General pathology and oncology terminology references.
* Optional LLM tools for translation or keyword exploration (not used for automated coding).

### Handling Ambiguity

* Explicit negation handling overrides keyword-based signals.
* Ambiguous or unclear cases are labeled as `UNKNOWN`.
* No inference is made when evidence is insufficient.

### Challenges

* Multilingual free text (Hebrew and English).
* Inconsistent terminology across records.
* Mixed structured and unstructured content within the same sample.

This conservative approach prioritizes reliability and interpretability over aggressive normalization.

---

## 8. Bonus: LLM-Assisted Automation Proposal

To scale this process to thousands of reports, an LLM-assisted layer could be introduced as a secondary extraction step.

A possible strategy would include:

* Prompting the model with raw clinical and histology text.
* Requesting a strictly structured JSON output with fixed fields (site, morphology, behavior, grade).
* Enforcing schema validation and confidence thresholds.
* Flagging low-confidence or conflicting cases for human review.

Rule-based validation and terminology cross-checking would remain essential to ensure reliability.

---

**Submission Date**: January 29, 2026

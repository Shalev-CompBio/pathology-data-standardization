# Pathology Sample-Level Clinical Extraction Pipeline

![Focus](https://img.shields.io/badge/Focus-Clinical_Informatics_%26_NLP-7B1FA2)
![Stack](https://img.shields.io/badge/Stack-Python_%7C_SNOMED_CT-FF9800)
![Institution](https://img.shields.io/badge/Institution-Hebrew_University_of_Jerusalem-00695C)
---
# Pathology Sample-Level Clinical Extraction Pipeline

**Assignment Submission**

## Context and Assignment Scope

This project was developed as part of an assignment focused on transforming unstructured pathology reports into a minimal, standardized dataset suitable for clinical research queries.
The emphasis was on conservative extraction, terminology standardization, and methodological transparency rather than full clinical completeness.

---

## 1. Executive Summary

This pipeline transforms raw, unstructured pathology test data into a minimal, sample-level structured dataset suitable for clinical research.
Key clinical attributes such as malignancy, tumor site, morphology, and grade are often embedded in free text or distributed across multiple rows. The pipeline consolidates and extracts these attributes while preserving raw text for traceability and auditability.

The resulting dataset enables standardized clinical queries while maintaining conservative assumptions.

---

## 2. Input Data Overview

The input consists of an Excel file (`pathology_tests.xlsx`) containing mixed structured and unstructured pathology records.
The data includes free-text clinical diagnoses and histology descriptions in Hebrew and English, with multiple rows often corresponding to the same patient or sample.

---

## 3. Data Processing Overview

The pipeline performs the following high-level steps:

1. Load the raw Excel data as-is, preserving all values as strings.
2. Restructure the data from test-level rows to a single row per sample.
3. Extract a minimal set of clinically relevant fields.
4. Apply conservative terminology standardization where mappings exist.
5. Validate that the resulting dataset supports key clinical queries.

---

## 4. Sample-Level Restructuring

Test-level rows are grouped by `(SampleID, SampleNumber)` to produce a single consolidated row per sample.

During this step:

* Core metadata such as patient identifiers, order information, and pathology procedure are preserved.
* Clinical diagnosis text is aggregated separately from other diagnostic text.
* Test results are pivoted into structured columns.
* Multiple values for the same test are concatenated and de-duplicated.

The resulting dataset contains exactly one row per pathology sample while retaining all relevant diagnostic context.

---

## 5. Terminology Selection and Field Mapping

### 5.1 Chosen Terminology

**Selected terminology:** SNOMED CT, conceptually aligned with the OHDSI Athena vocabulary framework.

**Rationale:**

* Coverage: SNOMED CT provides comprehensive concepts for anatomical sites and tumor morphology.
* Adoption: It is widely used in clinical systems and supported within the OHDSI ecosystem.
* Suitability: The terminology aligns well with body structure and morphologic abnormality concepts commonly observed in pathology data.

SNOMED CT was therefore selected as the primary terminology for coding pathology findings.

---

### 5.2 Minimal Field Mapping

Only fields that could be consistently identified across samples were extracted.

* **Malignancy (`is_malignant`)**
  Derived using a priority-based approach combining structured fields and free-text signals, with explicit negation handling.
  Encoded as `TRUE`, `FALSE`, or `UNKNOWN`.

* **Tumor site**
  Extracted from histology headers or a small bilingual dictionary.
  Stored as raw text, with optional SNOMED CT standardization when a mapping exists.

* **Tumor morphology**
  Extracted using regex patterns or dictionary-based fallback logic.
  Stored as raw text, with optional standardization.

* **Tumor grade**
  Extracted from histology text using FIGO, Gleason, and grade-group patterns.

* **Clinical diagnosis text**
  Preserved verbatim to maintain interpretability and auditability.

The coded fields in the minimal dataset include tumor site, tumor morphology, and tumor behavior (malignancy).

All terminology standardization relies on an explicit mapping file (`terminology_mapping.csv`).
If no mapping exists, standardized fields are left empty and raw extracted text is retained.

---

## 6. Clinical Query Validation

The resulting dataset supports clinical research queries such as:

* Counting patients or samples with malignant tumors using the `is_malignant` field.
* Identifying bladder cancer cases by filtering tumor site fields (standardized or raw).
* Identifying kidney-related suspicious samples through site extraction.
* Counting benign tumors using malignancy classification.

These queries are supported directly by structured fields in the final dataset and do not require re-parsing of raw text.

---

## 7. Approach and Methodology

The guiding principle of the approach was to extract only fields that could be consistently identified across all samples, while preserving raw text for auditability.

**Resources used:**
OHDSI Athena and SNOMED CT documentation, along with general pathology and oncology references.

**Handling ambiguity:**
Explicit negation handling overrides keyword-based signals.
Ambiguous or unclear cases are labeled as `UNKNOWN`.
No inference is made when evidence is insufficient.

**Challenges encountered:**
Multilingual free text (Hebrew and English), inconsistent terminology, and mixed structured and unstructured content within the same sample.

The minimal dataset was designed specifically to support clinical questions related to tumor location, behavior, and morphology, as demonstrated in the query validation section.

---

## 8. Bonus: LLM-Assisted Automation Proposal

To scale this process to thousands of reports, an LLM-assisted layer could be introduced as a secondary extraction step.

A possible strategy would include:

* Prompting the model with raw clinical and histology text.
* Requesting strictly structured JSON outputs with fixed fields (site, morphology, behavior, grade).
* Applying schema validation, confidence thresholds, and rule-based consistency checks.
* Requiring human review for low-confidence or conflicting cases.

---

**Submission Date:** January 29, 2026

---

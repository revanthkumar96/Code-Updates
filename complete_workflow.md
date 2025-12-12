# Smart Seva: Verification Workflow Documentation

This document details the end-to-end flow of the document verification system.

## Overview
The system processes an input document (Image/PDF) through a linear graph of 4 nodes, transforming the state at each step to build a comprehensive verification report.

**Graph Flow**: `Start` -> `Validate` -> [OCR](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/image_processor.py#13-160) -> `Verify` -> `Fraud Detection` -> `End`

---

## 1. Start: Input State
**Input**: [VerificationState](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/states.py#3-20) (Initial)
The workflow begins with the following data provided by the API:
*   `file_path` / `image_bytes`: The raw document file.
*   `input_type`: MIME type (e.g., `image/jpeg`).
*   `form_fields`: User-provided claim data to verify against the document (e.g., `{"name": "Ravi", "income": 50000}`).

---

## 2. Layer 1: Input Validation
**Node**: [validate_input](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/input_validate_node.py#42-129)
*   **Action**: Checks if the file format is supported and file size is within limits.
*   **Output Update**:
    *   If invalid: Adds error to `errors` list. Status -> `rejected`.
    *   If valid: Passes to next node.

---

## 3. Layer 2: OCR Extraction
**Node**: [extract_ocr](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/image_ocr_node.py#7-25)
*   **Action**: Uses **PaddleOCR** to extract raw text from the image.
*   **Output Update**:
    *   `ocr_text`: "GOVERNMENT OF INDIA... Name: Ravi..." (Raw String)
    *   If no text found: Adds error, Status -> `rejected`.

---

## 4. Layer 3: Document Verification (LLM)
**Node**: [verify_document](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/verification_node.py#7-61)
*   **Action**: Sends `ocr_text` and `form_fields` to the LLM (Groq/Ollama).
*   **Sub-Tasks**:
    1.  **classify**: Identifies Document Type (e.g., "Income Certificate").
    2.  **verify_fields**: Compares `form_fields` vs `ocr_text` (fuzzy matching).
    3.  **check_originality**: Heuristic check for "original" markings (stamps, signatures).
*   **Output Update**:
    *   `document_type`: "Income Certificate"
    *   `ocr_extracted_data`: `{"name": "Ravi", "annual_income": 90000, "address": "..."}`
    *   `form_verification`: `{"name": {"match": true, ...}}`
    *   `originality_check`: `{"is_original": true, "score": 0.95}`
    *   [verification_status](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/verification_processor.py#48-94): `verified` | `manual_review` | `rejected` (Preliminary status)

---

## 5. Layer 4: Fraud Detection (The "Forensics" Layer)
**Node**: [image_forensics](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py#19-67)
*   **Action**: Runs specialized sub-modules to detect anomalies.
*   **Sub-Modules**:
    1.  **Pattern Analysis**: Checks logic rules (e.g., Is "Date of Issue" in the future?).
    2.  **Statistical Analysis**: Z-Score check on numeric fields (e.g., Is Income > 3 Sigma from mean?).
    3.  **Geospatial Analysis** (LLM): Checks Jurisdiction (e.g., Does "Mandal X" belong to "District Y"? Is the distance plausible?).
    4.  **Text Consistency** (LLM): Checks NLP features (Grammar, Tone, Language Mixing, Parsing Errors).
*   **Output Update**:
    *   `fraud_detection`:
        ```json
        {
          "pattern_analysis": { "flags": ["Future Date"] },
          "statistical_analysis": { "score": 100, "status": "outlier" },
          "geospatial_analysis": { "status": "mismatch", "risk_score": 85 },
          "text_consistency_analysis": { "score": 40, "issues": ["Bad Grammar"] },
          "overall_score": 85 (Max Risk Score)
        }
        ```
    *   **Final Status Logic**:
        *   If `overall_score` > Threshold (e.g., 80) -> Status becomes `rejected` or `manual_review`.

---

## 6. Stop: Final Output
**Output**: [VerificationResponse](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/models.py#8-17) (JSON)
The final state is formatted into the API response:

```json
{
  "document_type": "Income Certificate",
  "verification_status": "manual_review",
  "confidence_score": 0.95,
  "extracted_data": { ... },
  "form_verification": { ... },
  "fraud_detection": {
     "geospatial_analysis": { ... },
     "text_consistency_analysis": { ... },
     ...
  },
  "originality_check": { ... },
  "errors": []
}
```

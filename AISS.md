# Codebase Explanation: `src/api` and `src/services`

This document provides a detailed breakdown of the functionality of files within the `src/api` and `src/services` directories.

## 1. API Layer (`src/api/`)

This folder handles incoming HTTP requests, input validation, and API routing.

### [main.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/main.py)
- **Purpose**: The entry point for the FastAPI application.
- **Key Functions**:
  - Initializes the `FastAPI` app with metadata (title, version).
  - Configures **CORS** middleware to allow cross-origin requests.
  - Includes the application router from [routes.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/routes.py).
  - Defines a root (`/`) and health check (`/health`) endpoint.
  - Configuration to run the server using `uvicorn` if executed directly.

### [routes.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/routes.py)
- **Purpose**: Defines the API endpoints.
- **Key Functions**:
  - **`POST /api/verify`**: The core endpoint.
    - Accepts a file upload (PDF/Image) and a JSON string of form fields.
    - Validates file types (JPEG, PNG, PDF, etc.).
    - Saves the file to a temporary location.
    - **Workflows**: Initializes a [VerificationState](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/states.py#3-19) and invokes the **LangGraph** workflow ([verification_graph](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/workflow.py#5-23)) to process the document.
    - Returns a structured [VerificationResponse](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/models.py#8-16) containing validation results, potential errors, and originality scores.

### [models.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/models.py)
- **Purpose**: Defines data structures (Pydantic models) for API validation.
- **Key Models**:
  - [VerificationRequest](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/models.py#4-7): Defines expected input (form fields).
  - [VerificationResponse](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/models.py#8-16): Defines the structured output schema, including:
    - [verification_status](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/verification_processor.py#35-81) ("verified", "rejected", "manual_review").
    - `form_verification` (field-by-field match results).
    - `originality_check` (scores and confidence).
    - `ocr_text` and `extracted_data`.

---

## 2. Services Layer (`src/services/`)

This folder contains the core business logic, AI workflows, and processing engines.

### Core Workflow Files

#### [workflow.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/workflow.py)
- **Purpose**: Orchestrates the document verification pipeline using **LangGraph**.
- **Logic**: Defines a `StateGraph` with a linear flow:
  1.  [validate_input](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/input_validate_node.py#42-129) (Check file validity).
  2.  [extract_ocr](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/image_ocr_node.py#7-25) (Convert image to text).
  3.  [verify_document](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/verification_node.py#7-61) (AI analysis).
  4.  `END`.

#### [states.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/states.py)
- **Purpose**: Defines the [VerificationState](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/states.py#3-19) TypedDict.
- **Key Data**: This is the shared state object passed between graph nodes, holding data like `file_path`, `image_bytes`, `ocr_text`, [verification_status](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/verification_processor.py#35-81), and `errors`.

### Nodes (`src/services/nodes/`)
Individual steps in the LangGraph workflow.

#### [input_validate_node.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/input_validate_node.py)
- **Purpose**: Prepares the input file for processing.
- **Logic**:
  - Validates file existence and paths.
  - Detects MIME types.
  - **Key Feature**: Converts the first page of a **PDF to an image** so the OCR engine can process it.

#### [image_ocr_node.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/image_ocr_node.py)
- **Purpose**: Extracts text from the document image.
- **Logic**: Calls the `ocr_service` (PaddleOCR) to get raw text. Handles empty text errors.

#### [verification_node.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/verification_node.py)
- **Purpose**: The "brain" of the operation.
- **Logic**: Calls the `verification_processor` (DSPy) to interpret the text. It aggregates results such as document type, form field matching, and originality scores into the final state.

#### [fraud_detection_node.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py)
- **Purpose**: Placeholder for future image forensics steps (currently logs "performing forensics").

### Processors (`src/services/processors/`)
Reusable logic and AI models used by the nodes.

#### [image_processor.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/image_processor.py)
- **Purpose**: Handles Optical Character Recognition (OCR).
- **Tech**: Uses **PaddleOCR**.
- **Logic**:
  - Configured for lightweight CPU usage (threads limited, angle classifier disabled).
  - Auto-resizes large images to prevent OOM errors.
  - Returns combined text from all detected blocks.

#### [verification_processor.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/verification_processor.py)
- **Purpose**: Uses LLMs to "understand" the document.
- **Tech**: Uses **DSPy** connected to **Ollama**.
- **Logic**:
  - Defines a [DocumentVerification](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/verification_processor.py#12-29) signature for the LLM.
  - **[verify()](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/verification_processor.py#82-139)**: Sends OCR text and form data to the LLM. The LLM returns a structured JSON assessing:
    - **Document Type** (e.g., "Aadhaar Card").
    - **Originality** (Is it a photocopy/fake?).
    - **Field Matching** (Does the name in the form match the name in the doc?).
  - Calculates the final [verification_status](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/verification_processor.py#35-81) based on a weighted score of matched fields and originality.

#### [fraud_detection_processor.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/fraud_detection_processor.py)
- **Purpose**: Currently empty/placeholder file.

### Fraud Detection Module (`src/services/modules/fraud_detection/`)
Advanced forensic tools for deep fake/tampering detection.

#### [error_level_analysis.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/error_level_analysis.py) ([ELAProcessor](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/error_level_analysis.py#27-417))
- **Purpose**: Detects digital manipulation in images (photoshopping).
- **Technique**: Uses **Error Level Analysis (ELA)**. It re-compresses images at different JPEG qualities and compares the differences to find "suspect" regions that don't match the compression artifacts of the rest of the image. Generates a "heatmap" of tempered areas.

#### [metadata_analysis.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/metadata_analysis.py) ([MetadataProcessor](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/metadata_analysis.py#6-115))
- **Purpose**: Analyzes file metadata for warning signs.
- **Technique**: Uses **ExifTool**. Checks for:
  - Suspicious software names (e.g., "Photoshop", "Canva").
  - Missing metadata (common in stripped social media images).
  - Chronological inconsistencies in creation/modification dates.

#### [pattern_analysis.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/pattern_analysis.py) ([PatternAnalyzer](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/pattern_analysis.py#33-392))
- **Purpose**: Logical fraud detection on the *extracted text*.
- **Checks**:
  - **Temporal**: Certificates issued on holidays/weekends, future dates, marriage before 18, etc.
  - **Statistical**: Unrealistic income (>1 crore) or age data.
  - **Entity**: Detects patterns like "Test User", keysmash text ("asdf"), or all-caps filler text.

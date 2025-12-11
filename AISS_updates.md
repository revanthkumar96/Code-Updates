# Project Updates: Groq Integration & Statistical Fraud Detection

This document details the changes made to the `smart_seva` backend to enable **Groq LLM support** and implement **Statistical Outlier Detection** for fraud prevention.

## 1. Feature: Codebase Analysis
**Goal**: Understand the existing architecture of `src/api` and `src/services`.
- **Output**: Analyzed the purpose of all API and Service files.
- **Reference**: [codebase_explanation.md](file:///C:/Users/sudik/.gemini/antigravity/brain/83e7be0d-09da-4eda-9e6d-af20efa46338/codebase_explanation.md)

---

## 2. Feature: LLM Provider Switching (Groq Support)
**Goal**: Enable the system to use **Groq** as an alternative to Ollama (which was missing on the host machine).

### Configuration Updates
- **File**: [src/config/settings.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/config/settings.py)
- **Change**: Added support for provider switching configurations (`llm_provider`, `groq_api_key`, `groq_model`).
- **Effect**: The app can now switch between `groq` and `ollama` via the `.env` file.

```python
# src/config/settings.py
class Settings(BaseSettings):
    # ... existing settings ...
    
    # LLM Settings
    llm_provider: str = "groq"  # "groq" or "ollama"
    groq_api_key: str | None = None
    groq_model: str = "openai/gpt-oss-20b"
```

### Processor Logic Updates
- **File**: [src/services/processors/verification_processor.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/verification_processor.py)
- **Change**: Added conditional logic to initialize the `dspy.LM` client. If `settings.llm_provider` is "groq", it connects to Groq's API; otherwise, it defaults to the local Ollama instance.

---

## 3. Feature: Statistical Outlier Detection
**Goal**: Detect potential fraud by identifying unrealistic values (e.g., extremely high income, unusual family size) using statistical profiles.

### New Module: Statistical Analyzer
- **New File**: [src/services/modules/fraud_detection/statistical_analysis.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/statistical_analysis.py)
- **Functionality**:
  - Defines `CERTIFICATE_PROFILES` for different document types (e.g., standard income/family size ranges).
  - Implements [_calculate_z_score](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/statistical_analysis.py#39-44) to measure how far a value is from the norm.
  - Flags values with a z-score > 2 (warning) or > 3 (outlier/anomaly).

### State Management
- **File**: [src/services/states.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/states.py)
- **Change**: Added a `fraud_detection` dictionary to the [VerificationState](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/states.py#3-20) to store comprehensive reports.

### Node Implementation
- **File**: [src/services/nodes/fraud_detection_node.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py)
- **Change**: 
  - Integrated both [PatternAnalyzer](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/pattern_analysis.py#33-392) (existing) and [StatisticalAnalyzer](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/statistical_analysis.py#35-136) (new).
  - The [image_forensics](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py#15-45) node now runs both analyzers and compiles an `overall_score`.
  - Automatically flags documents as `rejected` or `manual_review` if high fraud scores are detected.

### Workflow Integration
- **File**: [src/services/workflow.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/workflow.py)
- **Change**: Added the [image_forensics](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py#15-45) node to the processing graph. The workflow pipeline is now:
  [validate_input](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/input_validate_node.py#42-129) -> [extract_ocr](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/image_ocr_node.py#7-25) -> [verify_document](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/verification_node.py#7-61) -> [image_forensics](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py#15-45) -> `END`.
- **File**: [src/services/nodes/__init__.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/__init__.py)
- **Change**: Exported [image_forensics](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py#15-45) so it could be imported by the workflow.

---

## Summary of Modified Files

| File Path | Status | Description |
|-----------|--------|-------------|
| [src/config/settings.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/config/settings.py) | Modified | Added Groq configuration variables. |
| [src/services/processors/verification_processor.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/processors/verification_processor.py) | Modified | Implemented LLM provider switching logic. |
| [src/services/modules/fraud_detection/statistical_analysis.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/statistical_analysis.py) | **NEW** | Implemented z-score based outlier detection. |
| [src/services/states.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/states.py) | Modified | Added `fraud_detection` field to state. |
| [src/services/nodes/fraud_detection_node.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py) | Modified | Wired up analyzers to the forensic node. |
| [src/services/workflow.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/workflow.py) | Modified | Added forensic node to the execution graph. |
| [src/services/nodes/__init__.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/__init__.py) | Modified | Exported the new node module. |

# Statistical Analysis Module: Usage & Integration

**File**: [src/services/modules/fraud_detection/statistical_analysis.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/statistical_analysis.py)

## 1. What it does
The [StatisticalAnalyzer](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/modules/fraud_detection/statistical_analysis.py#35-136) class implements **Statistical Outlier Detection**.
- **Logic**: It compares extracted values (like Income, Age) against defined "profiles" (mean and standard deviation).
- **Math**: It calculates a **Z-score** [(value - mean) / std_dev](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/main.py#25-32).
- **Result**:
  - **Z > 3**: Flagged as an **Outlier** (High chance of fraud).
  - **Z > 2**: Flagged as a **Warning**.

## 2. Where it is used
The module is integrated into the **Fraud Detection Node**, which acts as a centralized verification step in the pipeline.

### Integration Point: [fraud_detection_node.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py)
**File**: [src/services/nodes/fraud_detection_node.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py)

It is imported and initialized at the top of the file:
```python
from src.services.modules.fraud_detection.statistical_analysis import StatisticalAnalyzer

# Initialize
statistical_analyzer = StatisticalAnalyzer()
```

And executed inside the [image_forensics](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py#15-45) function:
```python
def image_forensics(state: VerificationState) -> VerificationState:
    # ...
    # 2. Statistical Analysis
    statistical_results = statistical_analyzer.analyze(extracted_data, document_type)
    
    # Combine results
    fraud_report = {
        "pattern_analysis": pattern_results,
        "statistical_analysis": statistical_results, # <--- Added here
        # ...
    }
    # ...
```

## 3. Workflow Execution
**File**: [src/services/workflow.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/workflow.py)

The [image_forensics](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py#15-45) node (which handles the statistical analysis) is part of the main [verification_graph](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/workflow.py#5-25).

**Flow**:
1. [extract_ocr](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/image_ocr_node.py#7-25) (Gets text)
2. [verify_document](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/verification_node.py#7-61) (LLM checks)
3. **[image_forensics](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/services/nodes/fraud_detection_node.py#15-45)** (Runs Statistical Analysis & Pattern Checks)
4. `END`

## 4. API Output
**File**: [src/api/routes.py](file:///c:/Users/sudik/OneDrive/Desktop/Mindcres/ai-space/smart_seva/server/src/api/routes.py)

The results are finally exposed in the API response under the `fraud_detection` field:

```json
{
  "document_type": "Income Certificate",
  "verification_status": "manual_review",
  "fraud_detection": {
    "statistical_analysis": {
      "score": 100,
      "details": {
        "field_analysis": {
          "annual_income": {
            "value": 10000000,
            "z_score": 123.5,
            "status": "outlier"
          }
        }
      }
    }
  }
}
```

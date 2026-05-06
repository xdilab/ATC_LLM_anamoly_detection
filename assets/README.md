# ATC LLM Anomaly Detection: Digital Twin-Grounded ATC Safety Monitoring

<p align="center">
  <img src="assets/atc_anomaly_detection_pipeline.png" alt="ATC LLM Anomaly Detection Pipeline" width="95%">
</p>

<p align="center">
  <b>Digital Twin-grounded anomaly detection pipeline for KGSO ATC scenarios, contextual anomaly injection, Phi-4 detector training, evaluation, and ATC-safe response support.</b>
</p>

---

## Overview

`ATC_LLM_anamoly_detection` focuses on anomaly detection and response-support modeling for Air Traffic Control (ATC) simulation. The project extends the broader ATC LLM / XAION work by using KGSO Digital Twin snapshot data to create anomaly-injected datasets, train a fine-tuned Phi-4 detector, evaluate detection performance, and support downstream ATC-safe response generation.

The system is designed around contextual aviation anomalies rather than isolated text classification. Each anomaly is grounded in Digital Twin state, historical aircraft behavior, operational context, and expected ATC logic. This allows the model to learn how abnormal aircraft states, degraded surveillance information, runway/surface conflicts, and emergency-related events differ from normal operational patterns.

> **Research-use note:** This repository is a research prototype for simulation, anomaly detection, and ATC decision-support experimentation. It is not intended for operational air traffic control use.

---

## Anomaly Detection Pipeline

The pipeline is organized into four major stages:

### 1. Data Preparation

The pipeline begins with KGSO Digital Twin snapshot data. Each row represents a synchronized operational state that may include aircraft state, timing, position, speed, heading/track, altitude, surveillance fields, and contextual ATC information.

The preparation stage transforms raw CSV rows into contextual model inputs using historical context. Instead of only showing a single row, the transformed examples can include surrounding rows or prior aircraft behavior so the detector can identify whether a change is abnormal relative to recent state evolution.

```text
KGSO Digital Twin Snapshot
        ↓
Transformation of CSV rows using historical context
        ↓
Anomaly-injected datasets
```

### 2. Anomaly Detection Training

The anomaly-injected dataset is split into training and evaluation partitions. The training portion is used to fine-tune the anomaly detector, while the held-out portion is used to evaluate whether the model can identify anomalies it has not directly trained on.

```text
80% Dataset → Model Training
20% Dataset → Evaluation
```

The current prototype is centered around a fine-tuned Phi-4 model used as the anomaly detector.

### 3. Evaluation

The detector is evaluated using classification-oriented metrics:

- **Precision** — how many predicted anomalies were actually anomalous.
- **Recall** — how many true anomalies the detector successfully found.
- **F1 Score** — the balance between precision and recall.

These metrics help determine whether the detector is too aggressive, too conservative, or balanced enough for simulation-based response support.

### 4. Response Support

After detection, the pipeline connects anomaly classification to response support. The detector output, predicted label, and Digital Twin context are passed into a fine-tuned Phi-4 response model to suggest an ATC-safe response.

```text
Target / Predicted Output + DT Context
        ↓
Phi-4 Fine-Tuned Detector
        ↓
Phi-4 Fine-Tuned Response Model
        ↓
Suggested ATC-Safe Response Support
```

The response-support stage is intended to help convert anomaly detection into interpretable ATC decision support, such as alerting, monitoring, requesting clarification, issuing conservative instructions, or recommending a safe procedural response.

---

## Project Goals

- Transform KGSO Digital Twin CSV snapshots into model-ready anomaly detection datasets.
- Inject contextual anomalies using historical aircraft behavior and operational constraints.
- Train a fine-tuned Phi-4 detector to classify normal versus anomalous ATC/aircraft states.
- Evaluate detection using precision, recall, and F1 score.
- Connect detector outputs to a Phi-4 response model for suggested ATC-safe response support.
- Support aviation anomaly families such as airborne deviations, runway/surface conflicts, ADS-B integrity issues, identity conflicts, surveillance degradation, and emergency/transponder anomalies.
- Provide a repeatable pipeline for anomaly dataset generation, training, evaluation, and response-support experimentation.

---

## Main Pipeline Components

| Component | Purpose |
|---|---|
| KGSO Digital Twin Snapshot | Source contextual state data for aircraft, time, position, speed, altitude, and operational context |
| Historical Context Transformer | Converts raw rows into model-ready contextual examples using prior state/history |
| Anomaly Injector | Creates labeled anomaly examples by modifying selected fields or inserting abnormal state transitions |
| Anomalous Datasets | Train/validation/test datasets for detector training and evaluation |
| Phi-4 Detector | Fine-tuned model for identifying anomaly presence, family, and type |
| Evaluation Module | Computes precision, recall, F1 score, and related classification metrics |
| Response-Support Model | Uses predicted anomaly output and DT context to suggest an ATC-safe response |
| Suggested ATC-Safe Response | Human-readable controller-support output for simulation and review |

---

## Example Anomaly Families

The anomaly detector can support multiple aviation anomaly categories.

| Anomaly Family | Example Type | Example Signal |
|---|---|---|
| Airborne Anomalies | Altitude deviation | Aircraft altitude changes sharply relative to recent vertical profile |
| ADS-B Integrity / Surveillance | Stale or jumpy position | Aircraft coordinates repeat while surveillance age increases |
| ADS-B Integrity / Identity | Duplicate or spoofed ID | Same ICAO hex appears on conflicting spatial tracks |
| Surveillance Integrity | Low SIL / NAC / RSSI | Surveillance-quality indicators drop below expected reliability levels |
| Runway / Surface Anomalies | Runway occupancy conflict | Aircraft enters or occupies a runway when another operation conflicts |
| Speed / Movement Anomalies | Abnormal ground speed | Speed is inconsistent with taxi, takeoff, approach, or landing phase |
| Emergency / Transponder | Emergency squawk or state | Aircraft reports emergency-related code or abnormal transponder state |

---

## Example Dataset Record

The exact input format may vary depending on the dataset representation, but a model-ready example can include the current row, historical context, anomaly label, anomaly family, anomaly type, and expected response support.

```json
{
  "dt_time": "032230Z",
  "callsign": "FDX1249",
  "current_state": {
    "altitude_ft": 7600,
    "ground_speed_kt": 273.3,
    "track_deg": 205.81,
    "squawk": "2076"
  },
  "historical_context": [
    {
      "dt_time": "032200Z",
      "altitude_ft": 5700,
      "ground_speed_kt": 276.5,
      "track_deg": 208.05
    }
  ],
  "label": "anomaly",
  "anomaly_family": "Airborne Anomalies",
  "anomaly_type": "Altitude deviation",
  "detection_logic": "Sudden altitude change inconsistent with recent vertical profile.",
  "target_response_support": "Flag altitude deviation and request pilot verification of assigned altitude."
}
```

---

## Example Model Output

```text
Input:
DT Time 032230Z. FDX1249 altitude changed from 5700 ft to 7600 ft within the recent context while ground speed and track remained consistent with arrival flow.

Detector Output:
anomaly_family: Airborne Anomalies
anomaly_type: Altitude deviation
severity: monitor / verify
reason: altitude change breaks the aircraft's recent vertical profile.

Suggested ATC-Safe Response:
FDX1249, verify assigned altitude.
```

---

## Evaluation Metrics

| Metric | Meaning | Why It Matters |
|---|---|---|
| Accuracy | Overall percentage of correct predictions | Useful as a general baseline, but can be misleading if classes are imbalanced |
| Precision | Correct anomaly predictions divided by all anomaly predictions | Measures false-alarm behavior |
| Recall | Correct anomaly predictions divided by all true anomalies | Measures missed-anomaly behavior |
| F1 Score | Harmonic mean of precision and recall | Balances false alarms and missed detections |
| Family Accuracy | Correct anomaly-family prediction | Measures whether the model identifies the broad anomaly category |
| Type Accuracy | Correct anomaly-type prediction | Measures whether the model identifies the specific injected anomaly |
| Exact JSON Accuracy | Exact structured-output match, if JSON output is used | Measures formatting and schema compliance |

---

## Suggested Repository Structure

```text
ATC_LLM_anamoly_detection/
│
├── README.md
│
├── assets/
│   └── atc_anomaly_detection_pipeline.png
│
├── data/
│   ├── raw/
│   │   └── KGSO Digital Twin snapshot CSV files
│   ├── processed/
│   │   └── transformed contextual rows
│   └── anomaly_injected/
│       └── anomaly-injected train/validation/test datasets
│
├── mapping/
│   └── anomaly mapping and injection verification files
│
├── scripts/
│   ├── reconstruct_anomaly_hf_datasets.py
│   ├── train_phi4_anomaly_detector.py
│   ├── evaluate_phi4_anomaly_detector.py
│   └── generate_atc_safe_response_support.py
│
├── outputs/
│   ├── predictions/
│   ├── metrics/
│   └── response_support/
│
└── docs/
    └── dataset format notes, anomaly family definitions, and examples
```

The structure above is suggested for organization. File and script names can be adjusted to match the local implementation.

---

## Installation

Create and activate a Python environment:

```bash
python -m venv .venv
source .venv/bin/activate
```

Install common dependencies:

```bash
pip install torch transformers pandas numpy scikit-learn
pip install datasets accelerate peft
```

Optional packages for reporting and notebook analysis:

```bash
pip install matplotlib openpyxl jupyter
```

---

## Running the Pipeline

### 1. Prepare / Reconstruct Anomaly Datasets

```bash
python scripts/reconstruct_anomaly_hf_datasets.py
```

Expected outputs may include train/validation/test files in CSV and JSONL format, along with label maps and mapping verification reports.

### 2. Train the Phi-4 Anomaly Detector

```bash
python scripts/train_phi4_anomaly_detector.py
```

### 3. Evaluate the Detector

```bash
python scripts/evaluate_phi4_anomaly_detector.py
```

### 4. Generate ATC-Safe Response Support

```bash
python scripts/generate_atc_safe_response_support.py
```

---

## Dataset Split Strategy

A typical setup uses:

| Split | Purpose |
|---|---|
| Train / 80% | Fine-tune the anomaly detector |
| Evaluation / 20% | Test model behavior on held-out examples |

Depending on experiment design, the dataset may also be separated into train, validation, and test splits by date, month, aircraft, anomaly family, or scenario type.

---

## Relationship to the ATC LLM / XAION Work

This repository complements the broader ATC LLM and XAION pipeline:

- The ATC LLM work focuses on phraseology-aware and simulation-grounded ATC response generation.
- The agentic ATC prototype connects ASR, transcript correction, Phi-4 response generation, monitoring, and TTS.
- This anomaly-detection repository focuses on detecting abnormal Digital Twin states and connecting those detections to ATC-safe response support.

Together, these projects form a broader research direction for Digital Twin-grounded, safety-aware, and context-aware AI support for ATC simulation.

---

## Suggested Image Setup

Place the attached anomaly-detection pipeline image in the repository as:

```text
assets/atc_anomaly_detection_pipeline.png
```

Then the README image line will render correctly:

```html
<p align="center">
  <img src="assets/atc_anomaly_detection_pipeline.png" alt="ATC LLM Anomaly Detection Pipeline" width="95%">
</p>
```

---

## Disclaimer

This project is for research, simulation, and prototype development only. It does not provide certified aviation guidance and should not be used for real-world ATC operations, flight decision-making, or safety-critical deployment.

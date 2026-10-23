# SecureRide: Battery-Only E-Scooter Speeding Detection

> **Paper Reference:** J. Kim et al., *"SecureRide: Detecting E-Scooter Speeding Attacks via Battery Telemetry"*, ACM Transactions on Embedded Computing Systems (TECS), 2025.
>
> *Scope: Option 1 — battery-only **speeding** detection (single behavior). Sidewalk-riding detection and on-device deployment were intentionally out of scope.*

SecureRide detects illegal electric scooter speeding (> 20 km/h) solely using onboard battery electrical telemetry (**Voltage, Current, State-of-Charge** at ~1 Hz), providing tamper-resistant enforcement without reliance on vulnerable GPS or wheel-speed sensors.

---

## Key artifacts
- Final models: `models/theta_quick.{h5,tflite}`, `theta_accurate.{h5,tflite}`, `manifest.json`
- Model selection: `reports/model_selection.md`
- Training QA: `reports/training_qa.md`
- Dual-mode detection results: `results/dual_mode/dual_mode_report.md`

---

## Architecture Overview

```
                      +---------------------------------------+
                      | Raw Battery Telemetry (4 Ride Logs)   |
                      | [Voltage, Current, State-of-Charge]   |
                      +---------------------------------------+
                                          |
                                          v
                      +---------------------------------------+
                      | Stage 1: Data Pipeline (1 Hz Resample,|
                      | Cleaning, Windowing, Augmentation)    |
                      +---------------------------------------+
                                          |
                                          v
                      +---------------------------------------+
                      | Stage 2: Model Training & Selection   |
                      | 1D ResNet (CNN) & GRU Grid Sweep      |
                      | -> θ_quick (L=5, HL=1)                |
                      | -> θ_accurate (L=20, HL=3)            |
                      +---------------------------------------+
                                          |
                                          v
                      +---------------------------------------+
                      | Stage 3: Runtime Dual-Mode Detection  |
                      | (Paper Algorithm 1 State Machine)     |
                      | Quick Proposes (4.0s latency)         |
                      | Accurate Confirms (97.4% steady-acc)  |
                      +---------------------------------------+
```

---

## Key Results (Online Streaming Dual-Mode)

Evaluated on the held-out continuous test ride (`ride_29_12_2021_13_21_21.npz`, 2,923 s):

| Metric | Quick Baseline ($L=5$, HL=1) | Accurate Baseline ($L=20$, HL=3) | **Dual-Mode (Algorithm 1)** |
|---|:---:|:---:|:---:|
| **Raw Accuracy** | 94.42% | 91.01% | **94.07%** |
| **Steady-State Accuracy ($L_{trans}=25$)** | 95.64% | **97.41%** | **98.06%** |
| **Precision** | **99.61%** | 99.41% | 99.35% |
| **Recall** | **93.68%** | 89.82% | 93.51% |
| **F1 Score** | **96.55%** | 94.37% | 96.34% |
| **False Positives** | 9 | 13 | 15 |
| **Median Detection Latency** | 4.0 s | 18.0 s | **4.0 s (4.5× faster)** |

---

## Project Structure

```
E:\SecureRide\
├── README.md                    # Project overview and instructions
├── requirements.txt             # Python dependencies
├── .gitignore                   # Version control ignore rules
├── config.py                    # Data preprocessing configuration
├── train_config.py              # Model training and architecture configuration
├── run_pipeline.py              # Entry point: Full data preprocessing pipeline
├── run_training_pipeline.py     # Entry point: Model training, evaluation, and export
│
├── src/
│   ├── 01_inspect.py … 08_package.py      # Sequential data pipeline scripts
│   ├── model.py, train.py, train_utils.py # Model architecture & training engine
│   ├── evaluate_qa.py, recompute_metrics.py, select_and_export.py
│   └── dual_mode/                         # Runtime streaming detector (Algorithm 1)
│       ├── dual_mode_config.py            # Dual-mode runtime configuration
│       ├── detector.py                    # DualModeDetector state machine
│       ├── evaluate.py                    # Metrics, latency analysis & Fig. 9 plot
│       └── run_dual_mode.py               # Master streaming execution runner
│
├── data/
│   ├── raw/                     # Original 4 continuous ride CSVs
│   ├── processed/               # Cleaned 1 Hz continuous ride NPZs
│   └── final/                   # Packaged train/val/test splits + scaler.npz
│
├── models/
│   ├── theta_quick.{h5,tflite}    # Selected Quick model (L=5, HL=1)
│   ├── theta_accurate.{h5,tflite} # Selected Accurate model (L=20, HL=3)
│   ├── manifest.json              # Model parameters and performance manifest
│   └── sweep/                     # Archived sweep checkpoints
│
├── results/
│   ├── sweep.csv, metrics_corrected.csv # Complete grid sweep evaluation records
│   ├── confusion/                 # Confusion matrices for all models
│   └── dual_mode/                 # Streaming metrics.json, trace.npz, fig9_scenario.png
│
├── reports/                     # Quality assurance reports and analytical plots
└── docs/                        # Specifications, plans, and architectural designs
```

---

## Quickstart & Execution

### 1. Environment Setup
```powershell
# Create and activate virtual environment (Python 3.10+)
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

### 2. Run Data Pipeline (Stages 01–08)
```powershell
python run_pipeline.py
```
*Generates resampled, windowed, augmented datasets in `data/final/` and `reports/final_qa.md`.*

### 3. Run Model Training & Selection Pipeline
```powershell
python run_training_pipeline.py
```
*Executes CNN/GRU architecture sweep, recomputes metrics, evaluates QA sensitivity, and exports `theta_quick` and `theta_accurate` into `models/`.*

### 4. Run Streaming Dual-Mode Detection (Algorithm 1)
```powershell
python -m src.dual_mode.run_dual_mode
```
*Runs streaming detection on the held-out test ride, verifies QA edge cases, and outputs `results/dual_mode/fig9_scenario.png` and `results/dual_mode/dual_mode_report.md`.*

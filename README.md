# Kantar Data Science — Fraud Detection Assignment

This repository contains my solution to the two tasks in the assignment:

- **Task 1 — Anomaly Detection:** identify the time period with an abnormally
  high concentration of fraudulent registrations, and discuss how to detect
  such events without labels.
- **Task 2 — Fraud Classification:** build and evaluate a binary classifier
  that predicts whether a user is fraudulent (`is_fraud`).

---

## 1. Repository structure

```
Fraud Detection/
├── data/
│   └── datasets.csv                 # input data (CSV converted from the supplied Excel)
├── src/
│   ├── Task-1.ipynb                 # Task 1 — anomaly detection notebook
│   └── Task-2.ipynb                 # Task 2 — fraud classification notebook
├── notebooks/
│   └── Fraud_analysis.ipynb         # end-to-end notebook with the report inline
├── outputs/                         # generated plots / tables
│   ├── task1_daily_profile.png
│   ├── task1_hourly_anomaly.png
│   ├── task2_curves.png
│   └── task2_model_comparison.csv
├── REPORT.md                        # standalone analysis report
├── requirements.txt                 # pip dependencies
└── README.md
```

> **Data note:** the assignment supplied data as an Excel file. It has been
> converted to `data/datasets.csv` which is what all notebooks read.

---

## 2. Prerequisites

- **Python 3.14+** (developed on Python 3.14.3)
- `pip` (comes with Python)

---

## 3. Setup — create venv & install dependencies

```bash
# 1. Clone the repo
git clone <your-repo-url>
cd "Fraud Detection"

# 2. Create a virtual environment
python -m venv .venv

# 3. Activate it
#    Windows (PowerShell):
.venv\Scripts\Activate.ps1
#    Windows (CMD):
.venv\Scripts\activate.bat
#    macOS / Linux:
source .venv/bin/activate

# 4. Install all dependencies
pip install -r requirements.txt
```

---

## 4. How to run

All commands should be run from the project root (`Fraud Detection/`) with the
virtual environment **activated**.

### Option A — Run individual task notebooks via Jupyter

```bash
# Start Jupyter and open the notebook manually
jupyter notebook src/Task-1.ipynb
jupyter notebook src/Task-2.ipynb
```

Then click **Run All Cells** inside each notebook.

### Option B — Execute notebooks headlessly (no browser needed)

```bash
# Task 1
jupyter nbconvert --to notebook --execute src/Task-1.ipynb --output src/Task-1.ipynb

# Task 2
jupyter nbconvert --to notebook --execute src/Task-2.ipynb --output src/Task-2.ipynb
```

Both notebooks write results to the `outputs/` folder:

| Notebook | Outputs |
|----------|---------|
| Task-1   | `outputs/task1_daily_profile.png`, `outputs/task1_hourly_anomaly.png` |
| Task-2   | `outputs/task2_curves.png`, `outputs/task2_model_comparison.csv` |

### Option C — Full analysis notebook (code + report together)

```bash
jupyter notebook notebooks/Fraud_analysis.ipynb
```

Then click **Run All Cells**. The notebook reproduces both tasks with
explanatory markdown.

---

## 5. Viewing artefacts

After running, open the PNG/CSV files in the `outputs/` folder, or view the
notebook output cells directly. A written summary of findings is in
[`REPORT.md`](REPORT.md).

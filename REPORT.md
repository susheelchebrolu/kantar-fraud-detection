# Analysis Report — Fraud Detection Assignment

**Author:** Susheel
**Dataset:** 50,000 user registrations, 2025-01-01 → 2025-01-30. Overall fraud rate **5.0%** (2,500 fraudulent users).

---

## Task 1 — Anomaly Detection

### Methodology

I treated the registrations as a time series and looked for periods where the
behaviour deviated from the steady-state baseline. I aggregated at two
resolutions:

1. **Daily** — to spot which day(s) stood out.
2. **Hourly** — to pin down the precise window once a suspicious day was found.

For detection I used a **robust z-score** (median / MAD instead of mean / std,
so the score itself is not distorted by the outlier it is trying to find). An
hour is flagged when it is a strong outlier on **both** registration *volume*
and *fraud rate*.

### Key finding

There is one unmistakable anomaly:

| Metric | Baseline (typical hour) | Anomaly window |
|---|---|---|
| Registrations / hour | ~65–75 | **~230–250** |
| Fraud rate | ~4% | **~70%** |

**Detected anomaly window: `2025-01-11 01:00` → `2025-01-11 04:00` (3 hours).**

- 728 registrations occurred in this window at a **70.6%** fraud rate.
- This single 3-hour window contains **~20.6% of *all* fraud** in the entire month.
- The signature — a ~3.5× volume spike combined with a ~14× fraud-rate spike,
  confined to the small hours of the night — is highly consistent with a
  **scripted bot / bulk-registration attack** rather than organic user growth.

See `outputs/task1_daily_profile.png` (the Jan-11 spike is obvious at daily
resolution) and `outputs/task1_hourly_anomaly.png` (red points = flagged hours).

### Detecting anomalies *without labels*

In production we usually do **not** have `is_fraud` at detection time, so the
fraud-rate signal above is unavailable. Approaches that rely only on the
unlabelled registration stream:

1. **Volume / rate monitoring (statistical control charts).** Track
   registrations per unit time and alert when the count exceeds a robust
   threshold (rolling median + k·MAD, EWMA, or a Poisson/negative-binomial
   control limit). The attack above is trivially caught by volume alone — it is
   a 20σ event.
2. **Time-series decomposition + residual outliers.** Decompose the series
   (e.g. STL) into trend + daily/weekly seasonality + residual, then flag hours
   whose residual is extreme. This avoids false alarms during normal daily peaks.
3. **Unsupervised outlier models on the feature space.** Fit
   `IsolationForest`, `LocalOutlierFactor`, or an autoencoder on the
   per-registration features. Bot cohorts cluster tightly (near-identical
   feature signatures), so they surface as a dense pocket of anomalies.
4. **Population-drift / divergence tests.** Continuously compare the
   distribution of recent registrations against a trusted historical baseline
   (PSI, KL-divergence, KS-test per feature). A sudden shift — e.g. a surge of
   `outlook.com` + `android` + `PhD` sign-ups — signals a coordinated attack.
5. **Entity-level velocity rules.** Even without a fraud label, sudden spikes
   keyed on shared attributes (same email domain, same OS/device fingerprint,
   bursts of registrations in seconds) are strong unsupervised heuristics.

A practical system combines (1)/(2) for *cheap real-time alerting* with (3)/(4)
for *characterising* the anomalous cohort once an alert fires.

---

## Task 2 — Fraud Classification

### 2.1 Preprocessing & feature engineering

- **Target:** `is_fraud` (boolean → int). Class imbalance ≈ 19:1.
- **Missing values:** only `education_level` (~9.8%). Imputed as an explicit
  `"MISSING"` category (the missingness itself may be informative) for
  categoricals; median imputation for numerics.
- **Engineered features:**
  - `reg_hour`, `reg_dayofweek`, `reg_is_night` — generic temporal signals.
  - `email_is_freemail` — robust binary flag for free email providers.
  - `reg_duration_s` — registration time in seconds.
- **Leakage avoidance (important):** I deliberately did **not** feed the raw
  `registration_timestamp` / absolute date into the model. Because Task 1 showed
  the fraud is concentrated in one historical window, a model could simply
  memorise "Jan 11" and score a misleadingly high accuracy that would **not
  generalise** to future attacks. Only reusable, generic signals are used.
- **Encoding/scaling:** one-hot for categoricals, standard-scaling for numerics,
  all wrapped in a single scikit-learn `Pipeline` + `ColumnTransformer` so there
  is no train/test leakage.
- **Imbalance handling:** `class_weight="balanced"` (LR / RF) and
  `scale_pos_weight` (XGBoost). Stratified train/test split (75/25).

### 2.2 Models trained

Logistic Regression (interpretable baseline), Random Forest, and XGBoost

### 2.3 Performance

Evaluated on the held-out 25% test set (12,500 users). Because the classes are
imbalanced, **PR-AUC** is the headline metric (accuracy is misleading here).

| Model | ROC-AUC | PR-AUC |
|---|---|---|
| Logistic Regression | **0.968** | **0.772** |
| Random Forest | 0.960 | 0.762 |
| XGBoost | 0.961 | 0.755 |

All three models are strong (ROC-AUC ≈ 0.96–0.97). The models trade off
precision vs recall differently at the default 0.5 threshold — e.g. Random
Forest is high-precision (≈0.73) while Logistic Regression / XGBoost favour
recall (≈0.77–0.89). The right operating point depends on the business cost of a
missed fraud vs a false accusation, and is set by **moving the decision
threshold** rather than retraining.

### 2.4 What drives fraud (feature importance)

Top signals (Random Forest, `outputs/task2_feature_importance.png`):

| Rank | Feature | Importance |
|---|---|---|
| 1 | `education_level = PhD` | 0.138 |
| 2 | `email_domain = gmail.com` | 0.091 |
| 3 | `education_level = Masters` | 0.072 |
| 4 | `education_level = High School` | 0.068 |
| 5 | `os = android` | 0.064 |

These line up with the raw exploratory rates: fraud rate is **33.7%** for PhD
and **24.3%** for Masters profiles (vs <2% for High School / Bachelors),
**29.5%** for `outlook.com` and **17.8%** for `hotmail.com` (vs 2% gmail),
**21%** for Android, and **19–21%** for CEO/Director job titles. In other words
the synthetic fraudulent profiles over-claim seniority/education and cluster on
specific email/OS combinations — exactly the kind of signature a bulk-generated
cohort produces. Behavioural fields like `age`, `registration_duration_ms`,
`is_smoker`, and `is_drinker` carry little signal.

---

## Further Steps

With more time, data, or resources I would:

- **Deduplicate the anomaly's influence.** Train/evaluate with and without the
  Jan-11 attack cohort, and use time-based (not random) splits, to confirm the
  model generalises to *novel* attacks rather than the one in the sample.
- **Threshold & cost optimisation.** Calibrate probabilities (Platt / isotonic)
  and choose the operating threshold from an explicit cost matrix (cost of
  fraud vs cost of reviewing/blocking a genuine user).
- **Richer features.** Email-domain reputation, device/IP fingerprinting,
  velocity features (registrations per domain/IP per minute), and string
  similarity across user_ids to catch scripted batches.
- **Unsupervised + supervised ensemble.** Feed an `IsolationForest` anomaly
  score in as an extra feature so the classifier benefits from label-free
  structure too — useful for catching *new* attack patterns.
- **Explainability & monitoring.** SHAP values for per-decision explanations
  (important for any "fraud" label that affects a real user), plus drift
  monitoring (PSI) and scheduled retraining so the model keeps up with evolving
  attacks.
- **Hyperparameter search & cross-validation.** Stratified k-fold with
  `RandomizedSearchCV`/Optuna for the tree models, reporting mean ± std of
  PR-AUC rather than a single split.

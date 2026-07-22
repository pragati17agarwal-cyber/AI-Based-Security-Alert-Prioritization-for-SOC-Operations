# AI-Based Security Alert Prioritization for SOC Operations

A Random Forest classifier that prioritizes security alerts into **Critical / Medium / Low** tiers, using Microsoft's real-world [GUIDE cybersecurity incident dataset](https://www.kaggle.com/datasets/microsoft/microsoft-security-incident-prediction).

## Problem

Security Operations Centers (SOCs) receive far more alerts than analysts can manually triage. This project builds a model that automatically prioritizes incoming alerts based on their characteristics, so analysts can focus on the highest-risk incidents first.

## Dataset

- **Source:** Microsoft GUIDE dataset (Kaggle) — real-world telemetry across evidence, alerts, and incidents, annotated by customer security analysts.
- **Features used:** `Category`, `SuspicionLevel`, `LastVerdict`, `DetectorId`, `OSFamily`
- **Target:** Incident grade (`TruePositive` / `BenignPositive` / `FalsePositive`) mapped to priority tiers (`Critical` / `Medium` / `Low`)

## Approach

1. Loaded and cleaned relevant columns from the GUIDE_Train.csv file (13M+ row real-world dataset).
2. Handled missing values (categorical → "Unknown", numeric IDs → -1 placeholder).
3. Mapped raw incident grades to a 3-tier priority scale.
4. Label-encoded categorical features.
5. Trained a class-weight-balanced Random Forest to handle tier imbalance.
6. Evaluated with accuracy, per-class precision/recall/F1, and a confusion matrix — not just overall accuracy, since class imbalance can make accuracy misleading for security triage.
7. Examined feature importance to understand which alert characteristics most influence priority.

## Results

- **Overall accuracy:** 73.98%
- **Per-class performance:**

  | Priority | Precision | Recall | F1-score |
  |----------|-----------|--------|----------|
  | Critical | 0.75 | 0.75 | 0.75 |
  | Low | 0.63 | 0.60 | 0.62 |
  | Medium | 0.78 | 0.80 | 0.79 |

- **Total alerts analyzed:** 9,465,497 (Medium: 4,110,817 · Critical: 3,322,713 · Low: 2,031,967)
- **Most influential feature:** `DetectorId` (79.9% importance), followed by `Category` (13.4%), `LastVerdict` (4.7%), `SuspicionLevel` (1.8%), `OSFamily` (0.1%)

### Limitations & honest observations

- **Critical recall of 0.75 means ~25% of true Critical incidents are missed by the model** — in a real SOC, this is the single most important number to improve, since a missed Critical is far costlier than a false alarm on a lower-priority alert.
- **`DetectorId` dominates the model's decisions** (~80% of feature importance), far more than the alert's actual category, verdict, or suspicion level. This suggests the model is largely learning which detection rules have historically produced which grade, rather than reasoning from the alert's descriptive characteristics. This isn't necessarily wrong — detector reliability is a legitimate real-world signal — but it means the model may generalize poorly to a brand-new detector it hasn't seen before, and it's a clear direction for future improvement (e.g. evaluating a version of the model without `DetectorId` to see how much the other features can carry on their own).
- **"Low" is the weakest-performing class** (recall 0.60) — the confusion matrix shows meaningful overlap between true Low alerts and both Critical and Medium predictions, suggesting the current features don't cleanly separate low-risk alerts from the others.
- Model trained on a 300K-row stratified sample of the full ~9.5M-row cleaned dataset for computational efficiency on free-tier compute; evaluated on the full held-out test set.

### Engineering challenges encountered

- Loading the full `GUIDE_Train.csv` + `GUIDE_Test.csv` alongside each other exhausted Colab's free-tier RAM and crashed the runtime; resolved by loading only the required columns via `usecols` and dropping the unused test file entirely, since a manual stratified train/test split was performed instead.
- Initial `RandomForestClassifier.fit()` calls on the full ~9.5M-row training set took an impractically long time on free-tier CPU compute (single-core by default). Addressed by enabling `n_jobs=-1` for multi-core training, capping `max_depth`, and ultimately training on a 300K-row stratified sample rather than the full set — a deliberate tradeoff between iteration speed and using every available row, appropriate given free-tier compute constraints.
- These constraints meant the model was not trained on the entirety of the ~9.5M-row cleaned dataset; a production or funded-compute setting would likely train on the full set (or a much larger sample) and could plausibly improve on the numbers above.

> [!CAUTION]
> **Known drawback:** the original `RandomForestClassifier.fit()` call on the full training set repeatedly took 5+ minutes and, at one point, crashed the Colab runtime by exhausting free-tier RAM. This required iterating on the training code multiple times — first switching to `usecols` for lighter data loading, then adding `n_jobs=-1` and `max_depth=20` for multi-core, depth-capped training, and finally training on a 300K-row stratified sample instead of the full dataset. This is a real limitation of the current setup (free-tier compute, not the model design itself) and the honest reason the reported metrics come from a sample rather than the full 9.5M-row training set.

## Tools

Python, pandas, scikit-learn (Random Forest, LabelEncoder), matplotlib

## Notes

- Iterated from an earlier synthetic Pandas/NumPy-generated dataset (~94% accuracy) to this real-world GUIDE dataset for a production-realistic evaluation.
- Only a subset of GUIDE's ~40+ available columns were used, chosen for relevance to alert triage; the full dataset includes far more contextual and behavioral signals for future extension.

## How to run

1. Get a free Kaggle API token (kaggle.com → Settings → API → Create New Token).
2. Open `SOC_Project_Final.ipynb` in Google Colab.
3. Paste your token into the first cell, run all cells top to bottom.
4. Delete your token from the notebook before saving/sharing.

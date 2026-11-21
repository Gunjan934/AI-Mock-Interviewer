# Fresh ML answer evaluator — TF-IDF + LinearSVC

This document describes the **active** machine-learning answer evaluator. The
older three-model (Logistic Regression / Linear SVM / Random Forest) trainer and
the standalone Keras ANN experiment were archived in
`archive/previous_evaluators/` and are not imported by the application.

## 1. Files

| Path | Role |
| --- | --- |
| `app/ml/config.py` | Label space, score mapping, new artifact paths. |
| `app/ml/dataset_loader.py` | Dataset validation and question-group-disjoint splits. |
| `app/ml/train.py` | TF-IDF + LinearSVC training, validation tuning, artifact generation. |
| `app/ml/model_registry.py`, `app/ml/predictor.py` | Runtime loading and inference. |
| `app/ml/preprocessing.py` | Compatibility shim; implementation lives in `app/nlp/preprocessing.py`. |
| `scripts/train_ml_evaluator.py` | Training CLI. |
| `scripts/evaluate_ml_evaluator.py` | Re-evaluation CLI for the saved artifact. |
| `ml_artifacts/ml_evaluator/` | New model, metadata, metrics, classification report, confusion matrix. |

## 2. Dataset and leakage control

Dataset: `ml_data/mohler_asag_raw_open_ended.jsonl` (Mohler ASAG, CC BY 4.0).
Candidate answer column: `student_answer`; target: `score_avg` (0-5), mapped to
`Poor`, `Below Average`, `Average`, `Good`, `Excellent`.

Preparation: normalise text with the shared NLP helpers, drop missing/invalid
rows, remove duplicate question+answer+score records, derive a `question_group`
from the Mohler ID (e.g. `E01.Q01`), then split with `StratifiedGroupKFold` so
that answers to the same question never appear in two splits. TF-IDF is fitted
inside the pipeline on training text only; the test split is not used for
fitting, preprocessing choices, or tuning.

## 3. Model

```python
Pipeline([
    ("tfidf", TfidfVectorizer(max_features=5000, ngram_range=(1, 2),
                              min_df=2, max_df=0.9, sublinear_tf=True)),
    ("clf", LinearSVC(class_weight="balanced", random_state=42)),
])
```

A small grid of `C` values and unigram vs. unigram+bigram settings is scored on
the validation split only. The winner is refitted on the training split and
evaluated once on the held-out test split.

## 4. Commands

```bash
.venv\Scripts\python.exe scripts\train_ml_evaluator.py
.venv\Scripts\python.exe scripts\evaluate_ml_evaluator.py
.venv\Scripts\python.exe -m pytest -q
```

## 5. Flask integration

`ML_ANSWER_EVALUATION_ENABLED=1` enables the LinearSVC prediction. Config keys
`ML_MODEL_PATH` and `ML_MODEL_METADATA_PATH` point at `ml_artifacts/ml_evaluator/`.
The prediction is attached to `details["ml_evaluation"]` with `evaluator_kind:
"ml_v1"`; it does **not** change the primary `baseline_v1`/`ai_v1` scores or the
score format expected by the frontend. Empty answers are skipped. The baseline
evaluator remains the fallback, and NVIDIA question generation/follow-ups, RAG
and historical records are untouched.

The classifier produces a single ordinal label. It does **not** evaluate
individual rubric dimensions and does not provide calibrated probabilities
(`confidence` is `null` by design).

## 6. Known limitations

See `ml_artifacts/ml_evaluator/metrics.json` for the current numbers. The
dataset is heavily imbalanced (Excellent ≈ 71%); the model over-predicts
Excellent and performs poorly on the middle/minority classes. Macro-F1, per-class
recall and the confusion matrix are the metrics to watch — accuracy alone is
misleading here.

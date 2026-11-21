> **SUPERSEDED — historical reference only.** The active evaluator is the fresh TF-IDF + LinearSVC implementation documented in `docs/ML_EVALUATOR_LINEAR_SVC.md`. The three-model comparison, `answer_evaluator` artifacts and all ANN work described here were archived in `archive/previous_evaluators/`.

# Phase 4 — ML-Based Answer Evaluation (TF-IDF + Traditional Classifiers)

## 1. Objective
Introduce a separate, modular, offline-trained traditional ML answer-evaluation pipeline for the Intervia Flask application. The ML evaluator predicts answer-quality labels from question+answer text using TF-IDF features and classical classifiers, and is integrated **independently** from:
- `baseline_v1` deterministic evaluation (preserved)
- `ai_v1` NVIDIA-backed structured evaluation (preserved)

ML predictions are **not** verified human assessments and do **not** replace rubric-based scores. ML is disabled by default and only runs when explicitly enabled with a valid trained artifact.

## 2. Architecture (added without disturbing Phases 1–3)

### Added modules
- `app/ml/config.py` — label mapping, paths/config helpers
- `app/ml/preprocessing.py` — deterministic text normalization and question-aware representation
- `app/ml/dataset_loader.py` — dataset loading, validation, leakage-aware splitting, stats/fingerprint
- `app/ml/train.py` — pipelines, CV/tuning, comparison, artifact persistence
- `app/ml/evaluate.py` — offline evaluation of saved artifacts and comparison formatting
- `app/ml/model_registry.py` — safe artifact/metadata loading and validation
- `app/ml/predictor.py` — runtime prediction service (no retraining, no NVIDIA calls)
- `scripts/train_answer_evaluator.py` — offline training/comparison CLI
- `scripts/evaluate_answer_evaluator.py` — offline evaluation CLI
- `ml_data/README.md`, `ml_artifacts/answer_evaluator/` (artifacts), `docs/PHASE4_ML_EVALUATION.md`

### Integration points
- `app/config.py`: adds `ML_ANSWER_EVALUATION_ENABLED` (default `false`), `ML_MODEL_PATH`, `ML_MODEL_METADATA_PATH`
- `app/services/evaluation_service.py`: attaches independent ML prediction to `details['ml_evaluation']` after primary evaluation; **never changes** primary `evaluator_version`, scores, or overall
- `app/routes/interview.py`: preserves AI normalization/storage flow and ensures ML result (when enabled) is stored separately in evaluation details; ML failures never block answer storage
- `app/templates/interview/report.html`: displays ML prediction in a separate section (predicted label/model/version/dataset/confidence when supported) with clear disclaimer

### Preserved behavior
- `evaluate_answer()` and `baseline_v1` unchanged
- `evaluate_submission()` still prefers `ai_v1` when configured/enabled and falls back to `baseline_v1` on AI failure
- `to_storage_fields()` and AI evaluation service unchanged
- No NVIDIA provider/model/question-generation/interview-agent/auth/DB relationship changes
- No migrations; historical records remain valid

## 3. Dataset

- **Name:** Mohler ASAG (`nkazi/MohlerASAG`, config `raw`, split `open_ended`)
- **Source:** https://huggingface.co/datasets/nkazi/MohlerASAG (derived from Mohler et al., 2011 short-answer grading data)
- **Original paper:** Mohler, Bunescu, Mihalcea (2011), *Learning to Grade Short Answer Questions using Semantic Similarity Measures and Dependency Graph Alignments*
- **License:** CC BY 4.0 (per Hugging Face dataset card; verify terms before redistribution/use)
- **Local expected file:** `ml_data/mohler_asag_raw_open_ended.jsonl` (one JSON record per line)
- **Columns used:** `id`, `question`, `instructor_answer` (retained in data, not used in runtime text representation), `student_answer`, `score_grader_1`, `score_grader_2`, `score_avg`
- **Number of records (source split):** 2,273
- **Target column:** `score_avg` (numeric, 0–5). Original numeric scores are preserved.

### Target label definition (derived, deterministic)
Original `score_avg` is preserved; an ordinal 5-class quality label is derived for classification:

| Label | score_avg range |
|---|---|
| Poor | [0.0, 1.0) |
| Below Average | [1.0, 2.0) |
| Average | [2.0, 3.0) |
| Good | [3.0, 4.0) |
| Excellent | [4.0, 5.0] |

Rationale: dataset provides human-graded numeric scores; classification labels make Logistic Regression/SVM/Random Forest comparison straightforward while preserving ordinal meaning for interpretation.

### Dataset validation/preprocessing summary (pipeline)
- Required columns checked (`question`, `student_answer`, `score_avg`)
- Missing/empty questions, missing/empty answers, missing/invalid/non-numeric/out-of-range scores dropped with counts reported
- Exact duplicate `(question, student_answer, score_avg)` records removed with counts reported
- HTML/unicode/whitespace normalized; technical terms/numbers/symbols preserved
- Dataset fingerprint (SHA-256), label distribution, question/group counts reported

### Splitting strategy (leakage-aware)
- Train/validation/test split with fixed random state (42)
- **Grouped by question group** (`exercise.question` from Mohler `id`, fallback hash of question text) when feasible to reduce leakage of answers from the same question across splits
- Validation used for model-selection workflow support; **CV/tuning performed on training split only**; **held-out test used only for final reporting**
- Overlap checks recorded in split metadata

### Limitations
- Domain is computer-science/data-structures short answers (Mohler), not a perfect match for all interview domains/roles in this app
- Labels are derived from human grader scores but reflect that dataset’s rubric/context
- TF-IDF classifiers learn lexical/semantic-overlap patterns; they do **not** perform true multi-dimensional rubric reasoning (correctness/relevance/completeness/clarity/practical understanding) as `ai_v1` does
- Confidence/probability is reported only when estimator supports `predict_proba` and is not separately calibrated unless stated

## 4. Preprocessing and text representation

- Deterministic normalization: HTML entity/tag cleanup, Unicode NFKC, whitespace normalization, spacing tidy-up
- Preserves programming keywords, abbreviations, numbers, punctuation/symbols (e.g. C++, #, .NET, O(n log n))
- **Question-aware representation (consistent train/inference):**
  - `Question: <question>\nAnswer: <student_answer>`
- Instructor/reference answers are not included in runtime representation for consistency with app inference inputs

## 5. TF-IDF configuration

Implemented with `sklearn.feature_extraction.text.TfidfVectorizer` inside `Pipeline` (fit on training folds/data only; no test leakage).

Tuned parameters explored (grid search, per model):
- `ngram_range`: `(1,1)` and `(1,2)` (unigrams and unigrams+bigrams)
- `min_df`: `[1, 2]`
- `max_df`: `[0.9, 1.0]`
- `sublinear_tf`: `[True, False]` (and constrained in some RF grids for cost)
- `max_features`: model-specific bounded ranges for tractability

The fitted vectorizer is persisted as part of the selected sklearn pipeline.

## 6. Models

### Logistic Regression
- `sklearn.linear_model.LogisticRegression`
- Multiclass-compatible solver used: `lbfgs` (robust multinomial convergence on this sparse TF-IDF task)
- Tuned: `C`, TF-IDF params (uni/uni+bi); `class_weight='balanced'`, `max_iter=5000`
- Note: `saga` was initially explored but removed from the final full-dataset grid after repeated convergence warnings/excessive runtime; configuration fixed to a reliably converging solver (no convergence warnings in completed run).

### Linear SVM (primary SVM for sparse TF-IDF)
- `sklearn.svm.LinearSVC`
- Tuned: `C`, `loss='squared_hinge'`, class weighting, TF-IDF params
- No RBF SVC by default due to cost/fit characteristics on sparse high-dimensional data (documented tradeoff)

### Random Forest
- `sklearn.ensemble.RandomForestClassifier`
- Tuned: `n_estimators`, `max_depth`, `min_samples_split`, `min_samples_leaf`, TF-IDF params (bounded feature sizes)
- Pipeline densifies TF-IDF output for tree estimator compatibility where needed (model-specific handling documented in code)

## 7. Training, CV, and tuning

- Offline explicit training only: `python scripts/train_answer_evaluator.py`
- `Pipeline` + `GridSearchCV` + `StratifiedKFold` (fold count limited by smallest class count when necessary)
- Primary selection metric: **Macro F1** (`f1_macro`) on CV results (appropriate for class imbalance)
- Held-out test set used for final metrics/reporting only
- Reproducible random state: 42
- Saved artifacts/metadata include model, params, metrics, dataset fingerprint/stats, split info, timestamps

## 8. Evaluation metrics and model comparison

Computed on the same held-out test set (438 samples):
- Accuracy, Precision (macro), Recall (macro), Macro F1, Weighted F1
- Confusion matrix, classification report
- CV best score (macro F1), best hyperparameters, training/prediction duration

### Actual experimental results
<!-- TRAINING_RESULTS_START -->
**Status:** COMPLETED (full-dataset training and evaluation finished successfully).

**Dataset after validation:** 2,187 records (from 2,273 loaded; 86 exact duplicates removed; no missing/empty/invalid labels).  
**Split:** leakage-aware `stratified_rows_fallback` (question-group overlap present due grouping granularity; random_state=42, test_size=0.2, val_size=0.1) — train **1530**, val **219**, test **438**.  
**Selection metric:** Macro F1 (`f1_macro`) from stratified CV on training split (5 folds); held-out test used only for final reporting.

| Model | Accuracy | Precision (macro) | Recall (macro) | Macro F1 | Weighted F1 | CV F1 (macro) | Train (s) | Test pred (s) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Logistic Regression | 0.6507 | 0.4616 | 0.5752 | 0.4997 | 0.6717 | 0.4579 | 88.21 | 0.0453 |
| **Linear SVM (selected)** | **0.6872** | **0.5160** | **0.5242** | **0.5181** | **0.6834** | **0.5154** | 59.57 | 0.0471 |
| Random Forest | 0.6758 | 0.4362 | 0.3909 | 0.4086 | 0.6678 | 0.4397 | 2351.89 | 0.1896 |

**Best model:** Linear SVM (`LinearSVC`, pipeline with fitted TF-IDF). Selected by highest CV Macro F1 (0.5154).  
**Best hyperparameters:** `clf__C=2.0`, `clf__loss='squared_hinge'`, `tfidf__ngram_range=(1,2)`, `tfidf__min_df=1`, `tfidf__max_df=1.0`, `tfidf__sublinear_tf=True`, `tfidf__max_features=5000`.  
**Predict probabilities supported:** No (LinearSVC) — confidence/probability not returned by runtime predictor.

**Held-out confusion matrix (rows=true, cols=pred; labels [Poor, Below Average, Average, Good, Excellent]):**
```json
[[4,0,0,0,0],[0,3,2,2,4],[0,4,7,12,19],[0,3,9,23,33],[0,6,20,23,264]]
```
**Note:** Class imbalance is significant (Excellent dominates: 1560/2187 after validation), so Macro F1 is the primary selection metric; minority-class performance is limited and reflected in per-class metrics in `training_report.json`.
<!-- TRAINING_RESULTS_END -->

## 9. Best model selection and persistence

- Selection based on predefined CV Macro F1 (not training accuracy)
- Final reporting on held-out test set
- Artifacts:
  - `ml_artifacts/answer_evaluator/best_model.joblib`
  - `ml_artifacts/answer_evaluator/model_metadata.json`
  - `ml_artifacts/answer_evaluator/training_report.json`
- Runtime loader validates artifact existence/readability, metadata schema/labels, and estimator compatibility; missing/invalid artifacts reported as unavailable (no dummy predictions)

## 10. Prediction workflow

1. Check feature flag `ML_ANSWER_EVALUATION_ENABLED`
2. Load/validate cached model artifact + metadata
3. Normalize/build text: `Question: ...\nAnswer: ...`
4. Predict label via persisted pipeline (includes fitted TF-IDF)
5. Return label, model/version/metadata, and probabilities/confidence only when `predict_proba` is genuinely supported
6. Store as `details.ml_evaluation` separately from primary evaluation

No retraining on requests/startup; no NVIDIA/external calls; empty answers are not ML-scored.

## 11. Integration with Phase 3 / existing app

- ML is additive and separate (`details.ml_evaluation`, evaluator kind `ml_v1`)
- Primary evaluator remains `baseline_v1` or `ai_v1`; ML never overwrites primary scores/overall/evaluator_version
- ML failure/unavailability is logged/recorded safely and does not affect stored answers or session completion
- Disabled by default; existing behavior unchanged when disabled

## 12. Configuration

Environment/app config:
- `ML_ANSWER_EVALUATION_ENABLED=false`
- `ML_MODEL_PATH=ml_artifacts/answer_evaluator/best_model.joblib`
- `ML_MODEL_METADATA_PATH=ml_artifacts/answer_evaluator/model_metadata.json`

Dependencies added: `scikit-learn`, `pandas`, `numpy`, `scipy`, `joblib` (see `requirements.txt`).

## 13. Reproduction commands

From project root (Windows, venv):

```powershell
# 1) Ensure genuine labelled dataset is available
#    Default: ml_data/mohler_asag_raw_open_ended.jsonl (see ml_data/README.md)

# 2) Train all models, compare, and save best artifact
.\.venv\Scripts\python.exe scripts\train_answer_evaluator.py

# 3) Evaluate saved artifact on held-out test split
.\.venv\Scripts\python.exe scripts\evaluate_answer_evaluator.py

# 4) Run tests (non-live)
.\.venv\Scripts\python.exe -m pytest tests -q -k "not live"
```

## 14. Limitations and safety notes
- ML quality depends on dataset domain/labels; not a substitute for human grading or AI rubric evidence
- No five-dimension rubric scores invented from this classifier
- No fabricated metrics/labels; if dataset/artifacts unavailable, feature reports unavailable/blocked
- Do not use untrusted uploaded model artifacts; runtime loads configured local artifacts only
- Candidate answers must not be auto-added to training data without consent/privacy/labelling controls

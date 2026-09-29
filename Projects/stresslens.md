# StressLens: How a Survey, a Selfie, and a Little Machine Learning Estimate Student Stress

> A Streamlit app that fuses a 21-question well-being survey with facial emotion recognition to predict **Low / Medium / High** stress — and explains its reasoning.

---

## 1. The idea in one paragraph

University stress rarely comes from a single cause. Bad sleep, heavy study load, financial worry, weak social support, bullying, noisy housing — they stack up. **StressLens** treats stress as a multimodal signal: what students *say* (a structured survey) plus what their *face* shows (an emotion snapshot). It predicts a stress band with a confidence score, shows which survey answers drove the call, and offers coping resources when the result is High. It is a screening and self-reflection tool, not a medical diagnosis.

## 2. The user journey

1. Open the app (`streamlit run app.py`).
2. Left panel — **Survey**: 21 sliders, each 1–5, covering sleep, headache frequency, academic performance, study load, extracurriculars, social support, anxiety, depression, isolation, future insecurity, financial stress, teacher quality, peer pressure, bullying, safety, living conditions, basic needs, noise, self-esteem, mental-health history, and academic workload.
3. Sidebar — **Photo (optional)** + a **survey-vs-emotion weight** slider (default 0.7 = 70% survey, 30% face).
4. Right panel — **Results**: a big coloured stress label (green / amber / red), a confidence percentage, the detected emotion and its stress score, and a SHAP bar chart titled *"Why the model predicted this."*

No photo? No problem. The app substitutes a calm neutral face signal (`neutral`, `0.20`) and lets the survey decide — the exact behaviour you see when the emotion service is unreachable.

## 3. System architecture

```text
Streamlit UI (app.py)
 ├── Module 1 — Survey form (modules/module1_survey.py)
 │     └── Module 3 — Predict (modules/module3_predict.py)
 │           └── models/{scaler, selector_kbest, selector_rfecv, stacking_model}.pkl
 ├── Module 2 — Emotion (modules/module2_emotion.py)
 │     ├── OpenCV Haar-cascade face check
 │     └── Hugging Face Inference API (trpakov/vit-face-expression)
 ├── Module 4 — Fusion (modules/module4_fusion.py)
 │     └── survey_proba × w + emotion_proba × (1-w) → label + confidence
 └── Module 5 — SHAP explainability (modules/module5_shap.py)
       └── survey-only bar chart of top-8 drivers

Shared contracts (contracts.py) · Dataset (data/StressLevelDataset.csv)
```

The file `contracts.py` is the single source of truth: the 21-feature order (`SURVEY_FEATURE_ORDER`), the label set (`Low/Medium/High`), and the emotion-to-stress map (`EMOTION_TO_STRESS_WEIGHT`).

## 4. The survey brain: a tuned scikit-learn stack

The questionnaire is not summed naively. Each submission becomes a `raw_vector` of shape `(21,)` and flows through the same pipeline used in training (`modules/module3_train.py`):

1. **`MinMaxScaler`** — puts every 1–5 answer on a 0–1 scale.
2. **`SelectKBest(f_classif, k=15)`** — keeps the 15 most discriminative features.
3. **`RFECV(LinearSVC, cv=5)`** — recursive elimination with cross-validation prunes down to the minimal set that holds accuracy (minimum 5 features).
4. **`StackingClassifier`** — three diverse base learners vote, and a meta-learner adjudicates:
   - `SVC(probability=True)` — strong on margins,
   - `RandomForestClassifier(100 trees)` — robust to noisy answers,
   - `XGBoost` — catches nonlinear interactions,
   - Final estimator: `LogisticRegression`.

Training uses 10-fold stratified cross-validation with accuracy and per-class reports before a final fit on the full dataset. Inference (`module3_predict.py`) just replays scaler → KBest → RFECV → `predict_proba()` and returns `[P(Low), P(Medium), P(High)]`.

One pragmatic detail: the public CSV has 20 feature columns under slightly different names (e.g. `depression → depression_level`, `blood_pressure → financial_stress` as a proxy). A rename map plus a derived 21st feature (`academic_workload ≈ study_load`) bridges the Kaggle schema to the app schema.

**Technologies:** `scikit-learn`, `xgboost`, `pandas`, `numpy`, `joblib`.

## 5. The face brain: one API call, honestly labelled

Earlier versions of this project juggled local DeepFace/TensorFlow *and* a cloud API with provider flags and fallbacks. It was cut down to a single straight-line method:

1. **Face presence** — `opencv-python-headless` Haar cascade (`haarcascade_frontalface_default`). Blank wall or missing file → `no_face`, not a fake emotion.
2. **Emotion inference** — one `POST` of the JPEG bytes to the Hugging Face Router:
   `https://router.huggingface.co/hf-inference/models/trpakov/vit-face-expression`, a ViT (vision transformer) fine-tuned for facial expression. Auth is a `HF_TOKEN` loaded from `.env.local` via `python-dotenv` (or Streamlit Secrets in the cloud). Two attempts with a short wait on `503 model-loading` keep cold starts tolerable.
3. **Normalise + weigh** — labels like `happiness → happy` and `anger → angry` are canonicalised, the top score must clear `0.35` (else `low_confidence`), and the winner is converted to stress via a fixed psychological prior:

| Emotion | Stress weight |
|---|---|
| fear | 0.90 |
| angry | 0.85 |
| sad | 0.75 |
| disgust | 0.70 |
| surprise | 0.30 |
| neutral | 0.20 |
| happy | 0.05 |

Every failure mode — `image_missing`, `missing_token`, `auth_401/403`, `http_*`, `request_exception` — fails closed to `(neutral, 0.20)` with a status string the UI surfaces, so a camera problem can never invent stress.

**Technologies:** `opencv-python-headless`, `requests`, `python-dotenv`, Hugging Face Inference API, `Pillow`.

## 6. Fusion: where the two signals meet

`modules/module4_fusion.py` is deliberately transparent — a weighted average, not another black box:

```python
emotion_proba = [1 - es, es * 0.4, es * 0.6]  # Low, Medium, High
emotion_proba /= sum(emotion_proba)
fused = survey_proba * survey_weight + emotion_proba * emotion_weight
label = ["Low", "Medium", "High"][argmax(fused)]
confidence = max(fused)
```

A fearful face (`es = 0.90`) leans hard toward High; a happy face (`es = 0.05`) leans toward Low. The default 70/30 split reflects a design judgment: self-reported context is more reliable than a single snapshot, but a strong facial signal can still tip a borderline case. Users can drag the balance from 50/50 to 90/10.

## 7. Explainability: SHAP on the survey model

After every prediction the app renders a SHAP bar chart (survey model only — the face is excluded by design). Using a 200-row background sample from the training CSV, `shap.Explainer(model.predict_proba)` attributes the predicted class probability to each surviving feature, and the top 8 by absolute value are plotted: red pushes stress up, blue pushes it down. If SHAP fails in a minimal runtime, a graceful placeholder still shows the label and confidence instead of crashing the page.

**Technologies:** `shap`, `matplotlib`.

## 8. The full stack at a glance

| Layer | Tools |
|---|---|
| Frontend | `streamlit` (two-column layout, sidebar uploader + weight slider) |
| Survey ML | `scikit-learn` (Scaler, SelectKBest, RFECV, SVC, RF, LogReg), `xgboost`, `joblib` |
| Vision | `opencv-python-headless`, Hugging Face `vit-face-expression`, `requests`, `python-dotenv` |
| Explainability | `shap`, `matplotlib` |
| Data & science | `pandas`, `numpy`, `Pillow` |
| Secrets & deploy | `.env.local` / `.env` (gitignored), `.streamlit/secrets.toml`, any Python 3.12 venv |

`requirements.txt` is intentionally small now — DeepFace and `tf-keras` (~1 GB) were removed when the face path went cloud-only.

## 9. Limitations worth stating

- **Screening, not diagnosis.** Cutoffs are learned from one student dataset with proxy column mappings; they do not transfer to clinical settings.
- **One photo ≠ mood.** Lighting, pose, culture, and neurodiversity all shift expression; that is why the face only gets 30% by default and why low-confidence faces fall back to neutral.
- **Survey honesty matters.** Random or socially desirable answers produce a confident-looking but meaningless reading — SHAP at least makes that visible.
- **Privacy.** Photos are written to a temp file, sent once to the inference API, and deleted (`os.unlink`). No image is stored by the app.

## 10. What I would build next

- Optional multi-frame averaging (3–5 captures instead of one still) to smooth expression noise.
- Calibration curves and per-class thresholds rather than raw `argmax`.
- Persistent (opt-in) history with trend charts and export for counsellors.
- Replacing the Haar check with a modern face detector (e.g. MediaPipe/RetinaFace) while keeping the single-API-call philosophy.
- Model cards and fairness slices across gender, age band, and campus.

---

*Built with Python 3.12, trained with cross-validation, served with Streamlit, explained with SHAP — and deliberately simple where it counts: one survey pipeline, one face API call, one weighted average you can read in ten lines.*

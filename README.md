# End‑to‑End Data Science Project (wine quality)— Extended README

> Repository: `ALFIE-SADMAN/ds_end_to_end_project`

---

## 1) What this project is

A modular, configuration‑driven, end‑to‑end machine learning pipeline that takes you from raw data → validated & transformed features → trained model → experiment tracking → deployable app. The repo already sketches a classic production‑style layout with separate config, params, schema, pipelines, and a lightweight web app to demo inference.

**Core stages**

1. **Data Ingestion** → download/locate raw data and persist to `artifacts/`
2. **Data Validation** → check schema & basic sanity constraints from `schema.yaml`
3. **Data Transformation** → cleaning, feature engineering, train/val/test splits
4. **Model Training** → train/evaluate using the hyper‑parameters in `params.yaml`
5. **Model Evaluation & Experiment Tracking** → metrics logged to **MLflow** (optionally mirrored to **DAGsHub**)
6. **Packaging/Serving** → simple demo app in `app.py` (Flask/Jinja) + a `Dockerfile` for containerized runs

---

## 2) Repository layout

```
.
├── artifacts/                 # Saved datasets, models, scalers, reports (created by the pipeline)
├── config/                    # Configuration manager helpers (loaded in pipeline)
├── logs/                      # Rotating run logs
├── research/                  # Notebooks/prototypes (exploration & scratch work)
├── src/
│   └── datascience/           # Package with components & pipelines
│       ├── __init__.py
│       ├── components/        # Ingestion/validation/transform/train/eval building blocks
│       ├── pipeline/          # Orchestrating scripts that call components in order
│       ├── utils/             # I/O utils, logger, common helpers
│       └── ...
├── templates/                 # HTML templates for the demo web app (used by app.py)
├── ds_expirament/             # (typo?) scratch/experiment outputs
├── app.py                     # Demo web app to run inference
├── main.py                    # CLI entrypoint that runs the full training pipeline
├── params.yaml                # Model & training hyper‑parameters (learning rate, split sizes, etc.)
├── schema.yaml                # Expected input schema (dtypes, ranges, null rules)
├── requirements.txt           # Python dependencies
├── setup.py                   # Install the package (editable mode recommended)
├── Dockerfile                 # Container image to run training/inference
├── template.py                # Project bootstrap/templating helper
└── README.md                  # (this) Project documentation
```

> If any path differs on your branch, prefer the actual code over this diagram.

---

## 3) Quickstart

### A. Local development

```bash
# 1) Create & activate a venv
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate

# 2) Install the project in editable mode
pip install -r requirements.txt
pip install -e .

# 3) Run the training pipeline (end‑to‑end)
python main.py

# 4) Launch the demo app (after a model is trained & saved in artifacts/)
python app.py
# then open the printed URL (e.g., http://127.0.0.1:5000) in your browser
```

### B. MLflow tracking (local)

```bash
# in a separate terminal
mlflow ui --port 5001
# open http://127.0.0.1:5001 to browse experiments
```

To mirror runs to **DAGsHub**, export your repo URL and token as environment variables before running the pipeline.

---

## 4) Configuration & parameters

This project separates **static config**, **schema**, and **tunable params**:

* **`schema.yaml`** — authoritative description of input columns, types, and validation constraints. Typical fields:

  * `columns`: name → dtype (`int`, `float`, `string`, `category`, …)
  * `ranges` / `allow_nulls` rules for numerical fields
  * `allowed_values` for categoricals

* **`params.yaml`** — knobs you’ll change between experiments:

  * Data: split ratios, random seed
  * Features: imputation strategy, encoders, scalers, feature lists
  * Model: algorithm name, key hyper‑params
  * Training: epochs, batch size, early stopping
  * Evaluation: metrics to compute, cross‑val folds

* **`config/`** — configuration manager classes/functions that read the YAML files and hand typed, structured configs to components. This keeps components **stateless** and **testable**.

> Tip: keep `params.yaml` focused on *things you sweep*; move constants into `config/`.

---

## 5) Pipeline internals (how it runs)

`main.py` orchestrates the full flow by constructing each component and calling it in order, roughly like:

1. **Ingestion**: load/download source data to `artifacts/data/raw/` → return file paths
2. **Validation**: compare raw data ↔ `schema.yaml`; produce a report; raise or quarantine bad rows to `artifacts/data/invalid/`
3. **Transformation**: impute, encode, scale, generate features; persist transformers to `artifacts/preprocessing/`
4. **Training**: fit the model specified in `params.yaml`; persist to `artifacts/model/model.pkl`
5. **Evaluation**: compute metrics & plots; log to MLflow; optionally register the model

Each step logs to `logs/` and returns a small, typed output object (paths/metrics) to the next step.

---

## 6) Using the trained model for inference

* The demo app in `app.py` loads the latest model & preprocessing artifacts from `artifacts/` and exposes an HTML form or endpoint.
* Place sample payloads in `research/` for quick testing.
* For batch inference, add a small script under `src/datascience/pipeline/predict.py` that reads a CSV and writes predictions.

Example (HTTP JSON):

```bash
curl -X POST http://127.0.0.1:5000/predict \
     -H 'Content-Type: application/json' \
     -d '{"feature_1": 3.14, "feature_2": "A", ... }'
```

Response:

```json
{"prediction": 0.873, "model_version": "2025-09-28"}
```

---

## 7) Reproducibility & packaging

* Installable package: `pip install -e .` turns `src/datascience/` into an importable module.
* Python version is defined implicitly by `requirements.txt`; pin it explicitly in `pyproject.toml` or `.python-version` if needed.
* Random seeds: set them in `params.yaml` and inside trainers to keep runs deterministic.

---

## 8) Docker usage

Build an image and run the pipeline/app in an isolated environment:

```bash
# Build
docker build -t ds-e2e:latest .

# Train (mount local data/models if needed)
docker run --rm -it \
  -v "$PWD/artifacts":/app/artifacts \
  ds-e2e:latest python main.py

# Serve
docker run --rm -p 5000:5000 \
  -v "$PWD/artifacts":/app/artifacts \
  ds-e2e:latest python app.py
```

If your app uses environment variables (e.g., `MLFLOW_TRACKING_URI`), pass with `-e KEY=VALUE`.

---

## 9) Experiment tracking with MLflow & DAGsHub

* **Local MLflow:** run `mlflow ui` (see §3B). Your training code should call `mlflow.start_run()` and log params/metrics/artifacts.
* **DAGsHub integration:** set `MLFLOW_TRACKING_URI` to your DAGsHub project’s tracking URL and export `MLFLOW_TRACKING_USERNAME` / `MLFLOW_TRACKING_PASSWORD` (or token) before launching `main.py`.
* Version everything: models, metrics JSON, confusion matrices, and important plots under `artifacts/` so you can compare runs without the UI.

---

## 10) Configuration examples (edit for your dataset)

```yaml
# schema.yaml (example)
columns:
  age: int
  income: float
  city: category
allow_nulls:
  age: false
  income: false
allowed_values:
  city: [Adelaide, Sydney, Melbourne]

# params.yaml (example)
seed: 42
split:
  train: 0.7
  valid: 0.15
  test: 0.15
features:
  numerical: [age, income]
  categorical: [city]
  scaler: standard
model:
  name: xgboost
  n_estimators: 500
  max_depth: 6
training:
  early_stopping_rounds: 50
  eval_metric: rmse
logging:
  mlflow_experiment: ds-e2e
```

> Replace with your real schema and params to match the current code.

---

## 11) Testing strategy (recommended)

* **Unit tests** for each component in `src/datascience/components/`
* **Contract tests** for data validation vs `schema.yaml`
* **Golden tests** for the trainer: assert metrics on a tiny deterministic dataset
* **Smoke tests** for `main.py` to ensure the pipeline executes end‑to‑end

Add a `tests/` folder with `pytest` and wire it into GitHub Actions.

---

## 12) CI/CD (suggested GitHub Actions)

* Lint & type‑check (ruff + mypy)
* Run unit tests on pushes/PRs
* Build and push a Docker image on `main`
* Optionally, deploy the demo app to a small VM/railway/fly.io after a tagged release

Example workflow names: `.github/workflows/test.yml` and `deploy.yml`.

---

## 13) Data management & privacy

* Keep raw PII out of the repo. Load secrets from environment variables or a `.env` you never commit.
* If datasets are large/private, teach the ingestion step to read from S3/GDrive and cache to `artifacts/`.
* Store model cards (intended use, limitations, known biases) under `artifacts/reports/`.

---

## 14) Contributing

1. Create a feature branch
2. Write tests + docs alongside code
3. Run `ruff`/`black` and `pytest`
4. Submit a PR with a clear description and sample metrics/plots

---

## 15) Troubleshooting

* **MLflow UI empty** → ensure your trainer calls `mlflow.log_param`, `mlflow.log_metric`, and `mlflow.log_artifact`.
* **App can’t find model** → verify `artifacts/model/model.pkl` exists and that `app.py` points to the right path.
* **Schema errors** → update `schema.yaml` to reflect the *actual* ingested columns and datatypes.
* **Different Python versions** → rebuild your venv and reinstall `-r requirements.txt`.

---

## 16) License

This project uses the **GPL‑3.0** license. If you plan to embed this code in a closed‑source product, make sure you understand the copyleft obligations.

---

## 17) Roadmap (nice‑to‑have next)

* Add `predict.py` batch inference pipeline
* Add drift monitoring (baseline vs. production statistics)
* Add model registry & promotion flow (e.g., MLflow model registry)
* Publish a small demo dataset + sample payloads
* Wire up GitHub Actions for tests and Docker build
* Replace `ds_expirament/` with a clean `experiments/` folder name

---




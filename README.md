## Network Security – Phishing Detection (End‑to‑End ML Pipeline)

This repository contains an **end‑to‑end machine learning system** for detecting phishing / malicious network activity using structured data.  
It includes:

- **Modular training pipeline** (data ingestion → validation → transformation → model training)
- **MongoDB** as the data source
- **MLflow (via DagsHub)** experiment tracking
- **FastAPI** service exposing `/train` and `/predict` endpoints
- **S3 sync** for artifacts and models
- **Docker** configuration for containerized deployment (e.g. on AWS EC2 + ECR)

---

### 1. Project Overview

- **Problem**: Binary classification – predict whether a given network/URL record is **phishing** or **legitimate**.
- **Data**: Phishing dataset stored as CSV (`Network_Data/phisingData.csv`) and in **MongoDB** for the pipeline.
- **Solution**:
  - Ingest data from MongoDB.
  - Validate and transform it into ML‑ready features.
  - Train multiple models and select the best based on classification metrics.
  - Track runs with MLflow.
  - Serve the final model with a FastAPI application where users can upload a CSV and receive predictions.

---

### 2. Tech Stack

- **Language**: Python 3.x  
- **Core libraries**: `pandas`, `numpy`, `scikit-learn`, `pymongo`, `python-dotenv`
- **API**: `FastAPI`, `uvicorn`, `Jinja2` templates
- **Experiment tracking**: `mlflow` (backed by DagsHub)
- **Storage**: MongoDB (data source), AWS S3 (artifacts/models)
- **Containerization**: Docker
- **Cloud**: AWS (ECR, EC2)

---

### 3. Project Structure

Key directories and files:

```text
networksecurity/
├── app.py                      # FastAPI application (train & predict endpoints)
├── main.py                     # Script to run pipeline locally (manual training)
├── Dockerfile                  # Containerization config
├── requirements.txt            # Python dependencies
├── Network_Data/
│   └── phisingData.csv         # Original phishing dataset
├── final_model/
│   ├── model.pkl               # Final trained model (for API)
│   └── preprocessor.pkl        # Final preprocessor (for API)
├── prediction_output/
│   └── output.csv              # Last prediction results
├── templates/
│   └── table.html              # HTML template for prediction results
├── data_schema/
│   └── schema.yaml             # Data schema used for validation
└── networksecurity/            # Python package with core logic
    ├── components/             # Pipeline components
    │   ├── data_ingestion.py
    │   ├── data_validation.py
    │   ├── data_transformation.py
    │   └── model_trainer.py
    ├── pipeline/
    │   └── training_pipeline.py
    ├── entity/                 # Config & artifact entities
    ├── constant/               # Constants (paths, collection names, etc.)
    ├── utils/                  # Utility functions (ML utils, IO helpers, etc.)
    ├── cloud/
    │   └── s3_syncer.py        # S3 sync helper
    ├── logging/                # Central logging
    └── exception/              # Custom exception class
```

Artifacts generated during training are stored under `Artifacts/<timestamp>/...` and then optionally synced to S3.

---

### 4. Pipeline Architecture

The training pipeline is orchestrated by `TrainingPipeline` (`networksecurity/pipeline/training_pipeline.py`) and runs these stages:

1. **Data Ingestion**
   - Reads data from MongoDB (database and collection names defined in `networksecurity/constant/training_pipeline.py`).
   - Converts the collection to a `pandas` DataFrame.
   - Drops MongoDB `_id` column, replaces `"na"` with `NaN`.
   - Stores a **feature store CSV** and **train/test CSVs**.

2. **Data Validation**
   - Uses `data_schema/schema.yaml` to validate:
     - Column names
     - Data types
     - Basic integrity checks
   - Produces a `DataValidationArtifact` describing whether data is valid and paths to valid datasets.

3. **Data Transformation**
   - Applies preprocessing (encoding, scaling, etc.).
   - Saves transformed train/test as `.npy` files.
   - Saves the preprocessing object as a `.pkl` file for later inference.

4. **Model Training**
   - Loads transformed arrays.
   - Trains multiple models:
     - RandomForest, DecisionTree, GradientBoosting, LogisticRegression, AdaBoost.
   - Uses a hyperparameter grid + `evaluate_models()` utility to find the **best model**.
   - Evaluates using classification metrics (F1, precision, recall).
   - Logs metrics and models to **MLflow** (hosted on DagsHub).
   - Saves:
     - A full pipeline object in the artifacts directory.
     - The final model and preprocessor in `final_model/model.pkl` and `final_model/preprocessor.pkl`.

5. **Artifact & Model Sync to S3**
   - `S3Sync` (`networksecurity/cloud/s3_syncer.py`) is used to:
     - Sync the `Artifacts` directory to `s3://<TRAINING_BUCKET_NAME>/artifact/<timestamp>`.
     - Sync the final models directory to `s3://<TRAINING_BUCKET_NAME>/final_model/<timestamp>`.

---

### 5. API Layer (FastAPI)

The app defined in `app.py` exposes two main endpoints:

- **`GET /`**
  - Redirects to the FastAPI docs UI (`/docs`).

- **`GET /train`**
  - Instantiates `TrainingPipeline` and runs `run_pipeline()`.
  - Triggers the full training process (ingestion → validation → transformation → training → sync).
  - Returns a simple text response on success.

- **`POST /predict`**
  - Accepts a CSV file upload (`file: UploadFile`).
  - Reads the file into a DataFrame.
  - Loads the latest `final_model/preprocessor.pkl` and `final_model/model.pkl`.
  - Wraps them in a `NetworkModel` and calls `.predict()`.
  - Adds a `predicted_column` to the DataFrame.
  - Saves predictions to `prediction_output/output.csv`.
  - Renders the results as an HTML table using `templates/table.html`.

Run the API locally:

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

Or use the block at the bottom of `app.py`:

```bash
python app.py
```

Then open `http://localhost:8000/docs` in your browser.

---

### 6. Environment Variables & Configuration

Create a `.env` file in the project root and set:

- **MongoDB**

```bash
MONGODB_URL_KEY="mongodb+srv://<user>:<password>@<cluster>/<db>?retryWrites=true&w=majority"
```

- **MLflow / DagsHub**  
  (Some of these are configured directly in `model_trainer.py`, but conceptually you will need:)

```bash
MLFLOW_TRACKING_URI="https://dagshub.com/<user>/<repo>.mlflow"
MLFLOW_TRACKING_USERNAME="<dagshub-username>"
MLFLOW_TRACKING_PASSWORD="<dagshub-token>"
```

- **AWS / S3**

```bash
AWS_ACCESS_KEY_ID="<your-access-key>"
AWS_SECRET_ACCESS_KEY="<your-secret-key>"
AWS_REGION="us-east-1"
```

These credentials are typically used for:

- GitHub Actions secrets (CI/CD)
- AWS CLI / SDKs for S3 sync and ECR login

---

### 7. Local Setup & Usage

1. **Clone the repository**

```bash
git clone <this-repo-url>
cd networksecurity
```

2. **Create and activate a virtual environment** (recommended)

```bash
python -m venv venv
venv\Scripts\activate   # Windows
# source venv/bin/activate  # Linux/Mac
```

3. **Install dependencies**

```bash
pip install -r requirements.txt
```

4. **Configure `.env`**
   - Add the variables mentioned in the previous section.

5. **(Optional) Seed MongoDB**
   - Use `Network_Data/phisingData.csv` and your own script or `push_data.py` (if provided) to insert the data into your MongoDB collection matching the names in `networksecurity/constant/training_pipeline.py`.

6. **Run training locally (script mode)**

```bash
python main.py
```

7. **Run the API**

```bash
python app.py
```

Then visit `http://localhost:8000/docs` to:

- Call `/train` to retrain the model.
- Call `/predict` by uploading a CSV to get predictions.

---

### 8. Docker Usage

Build the Docker image:

```bash
docker build -t networksecurity:latest .
```

Run the container:

```bash
docker run -p 8000:8000 --env-file .env networksecurity:latest
```

Now the API is available at `http://localhost:8000`.

---

### 9. AWS Deployment (ECR + EC2)

**GitHub secrets (example)**:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION` – e.g. `us-east-1`
- `AWS_ECR_LOGIN_URI` – e.g. `788614365622.dkr.ecr.us-east-1.amazonaws.com/networkssecurity`
- `ECR_REPOSITORY_NAME` – e.g. `networkssecurity`

**On an Ubuntu EC2 instance, install Docker**:

```bash
sudo apt-get update -y
sudo apt-get upgrade -y

curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu
newgrp docker
```

Then:

1. Log in to ECR.
2. Pull the image pushed from your CI/CD or local machine.
3. Run the container (similar to the `docker run` command above).

---

### 10. Logging & Error Handling

- Centralized logging is implemented in `networksecurity/logging/logger.py`.
- Custom exception `NetworkSecurityException` in `networksecurity/exception/exception.py` is raised across components for consistent error handling.
- Training runs produce timestamped logs in the `logs/` directory and structured artifacts in the `Artifacts/` directory.

---

### 11. Future Improvements

- Add automated tests for each pipeline component and API routes.
- Implement data drift detection and automatic alerts if input data distribution changes.
- Add role‑based auth / API keys around `/train` and `/predict`.
- Extend the model comparison to include more advanced architectures (e.g. XGBoost, LightGBM).


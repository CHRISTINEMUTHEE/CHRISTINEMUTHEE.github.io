---
title: "Week 2: Configurations, Validation, and API Foundations"
date: 2026-02-17
summary: "Week 2 of the AI Engineering Mentorship: implementing environment-aware config, strict validation, typed interfaces, and testable FastAPI foundations."
tags: ["AI Engineering", "FastAPI", "Pydantic", "Mypy", "API Validation", "Configuration Management"]
categories: ["AI Engineering Journey"]
---

*Part of the [From Data Scientist to AI Engineer](/post/my-post/) series. ← [Week 1](/post/ds-to-ai-week1/)*

---

## Week 2 Focus

Week 2 shifted from experimentation to platform reliability. The goal was to replace implicit behavior with explicit contracts: configuration contracts, API contracts, and type contracts.

At a systems level, the objective was to make runtime behavior deterministic across environments while keeping local iteration fast.

What I targeted this week:

- ✅ Environment-specific config files (`.env.dev`, `.env.test`, `.env.prod`, `.env.example`)
- ✅ A clean `Settings` class powered by Pydantic
- ✅ Type hints and optional static checks with Mypy
- ✅ A FastAPI app with validated endpoints (`/healthz`, `/version`, `/validate`, `/predict`, `/feedback`)
- ✅ Stronger understanding of how real AI APIs are deployed and tested

---

## Repo Structure Upgrade

Before this week, iteration cost was high because code was copied across week folders. I moved to a single evolving codebase with week folders used mainly for write-ups.

```text
ai-engineering-mentorship-cm/
│
├── src/
│   ├── backend/
│   ├── config/
│   ├── data/
│   ├── models/
│   ├── training/
│   └── utils/
│
├── tests/
├── data/
├── notebooks/
│
├── week_1/
│   └── README.md
├── week_2/
│   └── README.md
├── week_3/
│   └── README.md
├── week_4/
│   └── README.md
│
├── .gitignore
├── .pre-commit-config.yaml
├── Makefile
├── pyproject.toml
├── requirements.txt
└── README.md
```

This reduced drift and made incremental refactors straightforward.

Code reference: [Project root structure](https://github.com/saranobrega/ai-engineering-mentorship-cm/tree/main)

Minimal pattern:

```python
# shared implementation
from src.backend.app import app

# week docs only
# week_2/README.md describes deltas, code stays in src/
```

---

## Environment-Specific Configuration

Core rule: no secrets or environment behavior in application code.

Instead, I moved to environment files:

- `.env.dev`
- `.env.test`
- `.env.prod`
- `.env.example` (safe template committed to Git)

Example values:

```env
APP_ENV=dev
DEBUG=True
API_VERSION=v1
LOG_LEVEL=info
DATABASE_URL=
PORT=8080
HOST=0.0.0.0
```

This enforces configuration parity while keeping sensitive values out of Git.

Code reference: [Environment example file](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/.env.example)

Load by environment:

```bash
export APP_ENV=dev
uvicorn src.backend.app:app --reload
```

---

## Pydantic Settings: One Source of Truth

I implemented a typed settings layer to centralize config loading and validation.

Code reference: [settings.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/config/settings.py)

```python
import os

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Application configuration settings loaded from environment variables."""

    app_env: str = Field(
        default="development",
        alias='APP_ENV',
        description="The application environment (development, testing, production).",
    )
    debug: bool = Field(
        default=False, alias="DEBUG", description="Enable or disable debug mode."
    )
    api_version: str = Field(
        default="0.1.0", alias='API_VERSION', description="The version of the API."
    )
    log_level: str = Field(
        default="info",
        alias='LOG_LEVEL',
        description="Logging level for the application.",
    )
    validation_min_len: int = Field(
        default=3,
        alias='VALIDATION_MIN_LEN',
        description="Minimum length for input validation.",
    )
    model_name: str = Field(
        default="object-detector-v1",
        alias='MODEL_NAME',
        description="Name of the machine learning model to use.",
    )
    allow_missing_inputs: bool = Field(
        default=False,
        alias='ALLOW_MISSING_INPUTS',
        description="Allow missing input files (for development purposes).",
    )
    # ---- Training Configurations ----
    train_epochs: int = Field(
        default=10,
        alias='TRAIN_EPOCHS',
        description="Number of epochs for model training.",
    )
    train_test_split: float = Field(
        default=0.2,
        alias='TRAIN_TEST_SPLIT',
        description="Proportion of data to use for testing during training.",
    )
    model_output_path: str = Field(
        default="data/models/model.pkl",
        alias='MODEL_OUTPUT_PATH',
        description="Path to save the trained model.",
    )
    model_type: str = Field(
        default="log_reg",
        alias='MODEL_TYPE',
        description="Type of the machine learning model to use. (svm, log_reg, random_forest)",
    )
    dataset_path: str = Field(
        default="/src/data/synthetic/feedback_sentiment_clean.csv",
        alias='DATASET_PATH',
        description="Path to the dataset file.",
    )

    model_config = SettingsConfigDict(
        env_file=os.getenv('ENV_FILE', '.env.dev'),  # dynamic env file selection
        env_file_encoding='utf-8',
    )


settings = Settings()
```

> **For deep thinkers & optimizers:** Once you understand the pattern above, you may wonder — do I always need this level of verbosity? Not always. Here is a more compact version that achieves the same core result with less ceremony:
> ```python
> from pydantic_settings import BaseSettings
>
> class Settings(BaseSettings):
>     app_env: str = "dev"
>     debug: bool = True
>     api_version: str = "v1"
>     log_level: str = "info"
>
>     @property
>     def is_production(self) -> bool:
>         return self.app_env == "prod"
>
> settings = Settings(_env_file=f".env.{Settings().app_env}")
> ```
> The explicit `Field()` style in the repository is preferred in team or production codebases because it self-documents each setting and makes `alias` mapping explicit. The compact style works fine for solo or early-stage projects.

Why this matters:

- Values are injected from environment files.
- Type coercion and validation happen during startup.
- Invalid config fails fast before request handling.
- App behavior remains consistent across modules.

In short, configuration now flows like this:

```text
.env.dev / .env.test / .env.prod
              |
              v
src/config/settings.py (Pydantic)
              |
              v
FastAPI app behavior
```

---

## Type Hints and Mypy

Type hints established explicit interfaces between modules and improved static tooling.

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"
```

Runtime behavior is unchanged, but annotation quality improves IDE inference and review clarity.

Then I added Mypy for static checks:

Code reference: [Makefile typecheck target](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/Makefile)

```makefile
typecheck:
	mypy src
```

Then run:

```bash
make typecheck
```

This catches signature and assignment mismatches before execution.

A key lesson here: add precise types after understanding data flow and stabilizing interfaces. That avoids noisy or misleading annotations.

---

## FastAPI Foundations with Validated Endpoints

With settings in place, I implemented baseline operational endpoints and typed payload models.

- `/healthz` for uptime probes
- `/version` to expose app/environment version
- `/validate` for schema and payload checks
- `/predict` to connect request handling to inference logic
- `/feedback` to capture downstream user signals

I used Pydantic schemas such as `ValidationRequest`, `ValidationResponse`, `PredictRequest`, and `PredictResponse` to enforce input/output contracts.

Code reference: [FastAPI app endpoints](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/backend/app.py)

Here is the full `app.py` as it exists in the repository:

```python
from __future__ import annotations

from typing import Optional

from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field

from src.config.settings import settings
from src.models.object_detector import ObjectDetector
from src.utils.io_utils import ensure_local_file_exists


# ---------- Pydantic Models ----------
class ValidationRequest(BaseModel):
    text: str = Field(..., min_length=1, description="Input text for validation.")


class ValidationResponse(BaseModel):
    valid: Optional[bool] = None
    reason: str


class PredictRequest(BaseModel):
    image_data: str = Field(..., description="Path or URL to the image for prediction.")


class PredictResponse(BaseModel):
    model_name: str = Field(..., description="Name of the model used for prediction.")
    result: dict = Field(..., description="List of detected objects.")


# ---------- FastAPI Application Instance ----------
app = FastAPI(title="AI Engineering Mentorship App", version=settings.api_version)


@app.get("/")
async def root():
    return {"message": "Welcome to the AI Engineering Mentorship App!"}


@app.get("/healthz")
def health_check():
    return {"status": "OK"}


@app.get("/version")
def get_version():
    return {
        "version": settings.api_version,
        "environment": settings.app_env,
        "debug": settings.debug,
    }


@app.post("/validate", response_model=ValidationResponse)
def validate_text(request: ValidationRequest):
    min_len = settings.validation_min_len
    if len(request.text.strip()) < min_len:
        return ValidationResponse(
            valid=False, reason=f"Text too short. Minimum length is {min_len}."
        )
    return ValidationResponse(valid=True, reason="Text is valid.")
```

> **For deep thinkers & optimizers:** The extra fields — `Field(..., min_length=1)`, `Optional[bool]`, full `description` strings — are production habits. A minimal version that teaches the same concept looks like this:
> ```python
> from fastapi import FastAPI
> from pydantic import BaseModel
>
> class ValidationRequest(BaseModel):
>     text: str
>
> class ValidationResponse(BaseModel):
>     is_valid: bool
>
> app = FastAPI()
>
> @app.post("/validate", response_model=ValidationResponse)
> def validate(payload: ValidationRequest) -> ValidationResponse:
>     return ValidationResponse(is_valid=len(payload.text.strip()) > 0)
> ```
> The difference: the repo version is explicit about `reason` (so callers know *why* validation failed), uses Pydantic's built-in `min_length` guard, and separates `valid` from `reason` as distinct fields. Both are correct; one is more communicative.

These models:

- enforce input and output structure
- validate data types automatically
- generate interactive API docs in `/docs`

This is where the chain became concrete: env files -> settings -> FastAPI behavior -> validated API contracts.

---

## Hardening the Predict Endpoint

A production inference endpoint should do three things well:

1. Accept typed requests
2. Validate inputs early
3. Fail safely with clear HTTP errors

I implemented stricter input checks and safe exception paths, then used environment flags to gate strictness. Development can be permissive; test and production stay strict.

Here is how the `/predict` endpoint is actually implemented in the repository:

```python
@app.post("/predict", response_model=PredictResponse)
def predict(request: PredictRequest):
    """
    Simulate model inference with validation and safe error handling.
    - Requires non-empty image path.
    - Checks if file exists (unless allowed in dev).
    """
    image_data = request.image_data.strip()
    allow_missing = settings.allow_missing_inputs
    if not image_data.startswith(("http://", "https://")) and not allow_missing:
        try:
            ensure_local_file_exists(image_data)
        except FileNotFoundError:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"File not found: {image_data}",
            )
    model = ObjectDetector()
    results = model.predict("fake_image_data")
    return PredictResponse(model_name=settings.model_name, result=results['objects'])
```

Notice: the request field is `image_data` (a path or URL to an image), not generic `text`. The endpoint validates that a local file exists before calling the model — unless `ALLOW_MISSING_INPUTS=True` in your environment (useful in dev to skip file checks). The response includes `model_name` so every prediction is traceable back to the exact model version.

> **For deep thinkers & optimizers:** The repository version extracts the file-existence check into `ensure_local_file_exists()` in `src/utils/io_utils.py` (single-responsibility principle). A self-contained version of the same logic that teaches the control flow equally well:
> ```python
> from fastapi import HTTPException
> from pathlib import Path
>
> @app.post("/predict", response_model=PredictResponse)
> def predict(req: PredictRequest):
>     image_path = req.image_path.strip()
>     allow_missing = getattr(settings, "allow_missing_inputs", False)
>     if not image_path.startswith(("http://", "https://")):
>         if not Path(image_path).exists() and not allow_missing:
>             raise HTTPException(status_code=400, detail=f"File not found: {image_path}")
>     model = ObjectDetector()
>     result = model.predict(image_path)
>     return PredictResponse(**result)
> ```
> The repository's version is preferred because it keeps the `predict` function focused on request handling, and the file-check utility can be reused elsewhere (DRY).

This pattern mirrors real deployment workflows where behavior changes by environment without modifying application code.

---

## Exercises

### Exercise 1 — Configurable Validation Threshold

Right now `/validate` uses `validation_min_len` which defaults to `3`. The goal of this exercise is to make that threshold fully environment-driven so you can change validation behavior without touching code.

Code reference: [settings.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/config/settings.py)

**Step 1:** The field is already in `Settings`:

```python
validation_min_len: int = Field(
    default=3,
    alias='VALIDATION_MIN_LEN',
    description="Minimum length for input validation.",
)
```

**Step 2:** Add the variable to both env files:

```env
# .env.dev
VALIDATION_MIN_LEN=3

# .env.test
VALIDATION_MIN_LEN=5
```

**Step 3:** Confirm that `/validate` reads from config (it already does in the repo):

```python
min_len = settings.validation_min_len
if len(request.text.strip()) < min_len:
    return ValidationResponse(
        valid=False, reason=f"Text too short. Minimum length is {min_len}."
    )
```

**Step 4:** Run the app and test it:

```bash
make run
```

**Step 5:** Test the endpoint in three ways:

**Option A — Swagger UI (easiest)**

Open `http://localhost:8000/docs`, find `/validate`, click "Try it out", and send:

```json
{"text": "hi!"}
```

Expected response:
```json
{"valid": true, "reason": "Text is valid."}
```

**Option B — Postman**

1. Open Postman → New Request
2. Method: `POST`, URL: `http://localhost:8000/validate`
3. Body → raw → JSON → paste `{"text": "hi"}`
4. Click **Send**

**Option C — curl**

```bash
curl -X POST http://localhost:8000/validate \
     -H "Content-Type: application/json" \
     -d '{"text": "hi"}'
```

Now switch to the test environment and try again:

```bash
ENV_FILE=.env.test make run
```

With `VALIDATION_MIN_LEN=5`, sending `{"text": "hi"}` should now return:

```json
{"valid": false, "reason": "Text too short. Minimum length is 5."}
```

This mirrors how real ML APIs roll out behavioral changes via environment config, not code edits.

---

### Exercise 2 — Harden `/predict` and Expose Model Metadata

**Step 1:** Add `ALLOW_MISSING_INPUTS` to your env files (already in `Settings`, just need the env values):

```env
# .env.dev
ALLOW_MISSING_INPUTS=True

# .env.test
ALLOW_MISSING_INPUTS=False
```

**Step 2:** Add `MODEL_NAME` to your env files:

```env
# .env.dev
MODEL_NAME=object-detector-v1
```

**Step 3:** Run in dev mode and test `/predict` in Swagger or curl:

```bash
curl -X POST http://localhost:8000/predict \
     -H "Content-Type: application/json" \
     -d '{"image_data": "some/fake/path.jpg"}'
```

With `ALLOW_MISSING_INPUTS=True` in dev, you should get:

```json
{"model_name": "object-detector-v1", "result": {"objects": ["car", "person"]}}
```

**Step 4:** Now run in strict mode and send the same request:

```bash
ENV_FILE=.env.test make run
```

Expected response:

```json
{"detail": "File not found: some/fake/path.jpg"}
```

**Think:** Why would engineers expose `model_name` in production API responses?  
*(Hint: it ties every prediction back to a specific model version — critical for debugging, audits, and A/B rollouts.)*

---

## Testing: Config and Endpoint Reliability

Week 2 targeted the two most critical infrastructure layers: configuration and API contracts.

Code reference: [Tests directory](https://github.com/saranobrega/ai-engineering-mentorship-cm/tree/main/tests)

### Test Configuration Loading

Create `tests/test_settings.py`:

```python
from src.config.settings import settings

def test_settings_load_correctly():
    """Test that the configuration is loaded correctly from the .env file."""
    assert settings.validation_min_len == 3   # from .env.dev
    assert settings.model_name == "object-detector-v1"  # from .env.dev
```

**Exercise:** Run it with `pytest -v tests/test_settings.py` — this tells you whether Pydantic is reading your `.env.dev` file correctly.

### Test API Endpoints

Create `tests/test_endpoints.py`:

```python
from fastapi import status
from fastapi.testclient import TestClient
from src.main import app

client = TestClient(app)


def test_health_check():
    response = client.get("/healthz")
    assert response.status_code == status.HTTP_200_OK
    assert response.json() == {"status": "OK"}


def test_version_endpoint():
    response = client.get("/version")
    assert response.status_code == status.HTTP_200_OK
    data = response.json()
    assert "version" in data
    assert "environment" in data
    assert "debug" in data


def test_validate_text_dev():
    """Test /validate with dev environment (VALIDATION_MIN_LEN=3)."""
    response = client.post("/validate", json={"text": "hi"})
    assert response.status_code == status.HTTP_200_OK
    assert response.json() == {"valid": False, "reason": "Text too short. Minimum length is 3."}


# Run this one separately: ENV_FILE=.env.test pytest -k test_validate_text_test
def test_validate_text_test():
    """Test /validate with test environment (VALIDATION_MIN_LEN=5)."""
    response = client.post("/validate", json={"text": "hi"})
    assert response.status_code == status.HTTP_200_OK
    assert response.json() == {"valid": False, "reason": "Text too short. Minimum length is 5."}


def test_predict_valid_image():
    """Test /predict with a valid image path."""
    response = client.post("/predict", json={"image_data": "data/fake_image.jpg"})
    assert response.status_code == status.HTTP_200_OK
    data = response.json()
    assert "model_name" in data
    assert "result" in data


def test_predict_invalid_image():
    """Your turn! Write a test that sends a non-existent path and expects HTTP 400."""
    pass
```

A key gotcha: Pydantic's `Settings` loads the `.env` file **once**, when your app is imported. Running `test_validate_text_test` in the same process as `test_validate_text_dev` will not automatically switch environments. So you run each in the correct environment:

```bash
pytest -k test_validate_text_dev
ENV_FILE=.env.test pytest -k test_validate_text_test
```

**Exercise 1:** Implement `test_predict_invalid_image` — send a path that does not exist and assert you get a `400` response.

**Exercise 2:** Run the full test suite:

```bash
make test
```

This test layer reduced regression risk and made refactoring safer.

---

## Engineering Habits That Paid Off

Code reference: [Engineering practices](https://github.com/saranobrega/ai-engineering-mentorship-cm/tree/main/week_2)

These practices came directly from code review feedback applied to this project.

---

### 1) Avoid Magic Numbers and Strings

Hardcoded values make code cryptic and brittle. Use named constants or settings.

**Bad (hidden rule):**

```python
if len(request.text.strip()) < 3:
    return {"valid": False, "reason": "Too short (min 3)."}
```

**Good (explicit rule via settings):**

```python
min_len = settings.validation_min_len  # from .env, defaults to 3
if len(request.text.strip()) < min_len:
    return ValidationResponse(valid=False, reason=f"Text too short. Minimum length is {min_len}.")
```

**Exercise:** Search for `3`, `5`, and other meaningful literals in `src/backend/app.py` and `src/training/`. Promote any that represent business rules to `Settings` or named module-level constants. Add a short comment explaining *why* the default exists.

---

### 2) Use Meaningful, Consistent Names

Names should reveal intent.

**Less clear:**
```python
def predict(req: PredictRequest):
    p = req.image_data
```

**Better:**
```python
def predict(request: PredictRequest):
    image_data = request.image_data.strip()
```

Stick to `snake_case` for variables and functions, and keep the same terms everywhere (`image_data`, not `imgData` somewhere else).

---

### 3) Favor Early Returns Over Nested Logic

**Nested (harder to scan):**
```python
if image_data:
    if image_data.startswith(("http://", "https://")) or Path(image_data).exists():
        return PredictResponse(...)
    else:
        raise HTTPException(400, "File not found")
else:
    raise HTTPException(400, "image_data must not be empty")
```

**Early returns (clean):**
```python
if not image_data:
    raise HTTPException(400, "image_data must not be empty")

if not image_data.startswith(("http://", "https://")):
    if not Path(image_data).exists() and not allow_missing:
        raise HTTPException(400, f"File not found: {image_data}")

return PredictResponse(...)
```

The repository already uses early returns in `/validate` and `/predict`. Keep that pattern.

---

### 4) Avoid Long Parameter Lists — Use Models

You already model request bodies with Pydantic. Keep related inputs grouped in request models (`PredictRequest`, `ValidationRequest`). If you later add `threshold`, `return_embeddings`, or `model_version`, add them to the **request model**, not the function signature.

---

### 5) Keep Functions Small and Focused

**The file-check logic embedded in the endpoint:**
```python
# inside predict() in app.py
if not image_data.startswith(("http://", "https://")):
    p = Path(image_data)
    if not p.exists() and not allow_missing:
        raise HTTPException(status_code=400, detail=f"File not found: {image_data}")
```

**Extracted into a reusable utility (what the repo does):**
```python
# src/utils/io_utils.py
from pathlib import Path

def ensure_local_file_exists(path: str) -> None:
    if not Path(path).exists():
        raise FileNotFoundError(path)
```

```python
# src/backend/app.py
if not image_data.startswith(("http://", "https://")) and not allow_missing:
    try:
        ensure_local_file_exists(image_data)
    except FileNotFoundError:
        raise HTTPException(400, detail=f"File not found: {image_data}")
```

Single responsibility: `predict` handles the request; `ensure_local_file_exists` handles the file check.

---

### 6) DRY — Don't Repeat Yourself

If you have post-processing logic (threshold filtering, label mapping), put it in one helper and reuse it across training, evaluation, and the backend.

```python
# src/utils/pred_utils.py
from typing import Any

def normalize_prediction(result: dict[str, Any]) -> dict[str, Any]:
    # future: thresholding, label mapping, etc.
    return result
```

---

### 7) Comment Why, Not What

Your code is self-describing with type hints and Pydantic models. Comments should explain reasoning, not restate code.

**Helpful:**
```python
# Accept missing files in dev to speed local testing; enforce existence in test/prod.
allow_missing = settings.allow_missing_inputs
```

**Unnecessary:**
```python
# Set allow_missing to settings.allow_missing_inputs
allow_missing = settings.allow_missing_inputs
```

---

### 8) Write Good Commit Messages

Good commits state the *what* in the subject and the *why* in the body.

```
Extract file existence check into io_utils and add tests

- Move local file validation from /predict into utils (single responsibility)
- Add unit tests to ensure FileNotFoundError is raised consistently
- Clarify behavior: allow missing files in dev via ALLOW_MISSING_INPUTS
```

**Exercise — Commit Hygiene:** Make two small, focused commits applying the practices above. Each should have a clear imperative subject and a short body explaining why.

---

## Week 2 Wrap-Up

By the end of Week 2, the project moved from runnable scripts to an operable service baseline.

I built:

- Environment-aware configuration management
- Type-safe settings and request validation
- A FastAPI foundation with practical operational endpoints
- Better predict endpoint safety and error handling
- Tests that verify both configuration and API behavior

The main technical takeaway: reliable AI services depend on deterministic configuration, explicit schemas, static checks, and predictable error semantics.

**Continue to Week 3 →** [Modularising Model Training & Building Evaluation Pipelines](/post/ds-to-ai-week3/)

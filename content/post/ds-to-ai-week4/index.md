---
title: "Week 4: Deployable API, Training Artifacts, and CI Thinking"
date: 2026-03-03
summary: "Week 4 of the AI Engineering Mentorship: hardening the API, making training and evaluation produce artifacts, and thinking through Docker and CI as portfolio-level engineering requirements."
tags: ["AI Engineering", "FastAPI", "MLOps", "Docker", "CI", "Deployment", "Evaluation"]
categories: ["AI Engineering Journey"]
---

*Part of the [From Data Scientist to AI Engineer](/post/my-post/) series. ← [Week 3](/post/ds-to-ai-week3/)*

---

## Week 4 Focus

Week 4 was about finishing the project like an engineer, not just like a learner.

The goal was no longer “add more features.” The goal was to make the system feel deployable, observable, and verifiable. That meant looking at the API, the training pipeline, the artifacts, and the developer workflow as one complete portfolio project.

The mentorship framed the shift like this:

> Move from “it works on my machine” to “it runs anywhere, fails predictably, and can be checked automatically.”

That framing mattered. It pushed me to think less about isolated code files and more about the trustworthiness of the whole system.

---

## What Week 4 Was Trying to Complete

The Week 4 README defines four major deliverable areas:

1. FastAPI hardening
2. training and evaluation support
3. Docker packaging
4. CI workflow basics

By the end of the week, the intended outcome was a project where someone else could:

- run the API locally
- hit operational endpoints like `/healthz` and `/version`
- train and evaluate the model offline
- inspect saved artifacts
- package the service for container execution
- rely on automated checks to catch regressions

That is a very different level of completeness from simply having a notebook or a script that runs once.

---

## Serving Layer: From Working API to Deployable API

The core FastAPI app was already in place from Week 2. In Week 4, the focus became the gap between “works” and “production-shaped.”

Code reference: [app.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/backend/app.py)

```python
from __future__ import annotations

from typing import Optional

from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field

from src.config.settings import settings
from src.models.object_detector import ObjectDetector
from src.utils.io_utils import ensure_local_file_exists


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


app = FastAPI(title="AI Engineering Mentorship App", version=settings.api_version)


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


@app.post("/predict", response_model=PredictResponse)
def predict(request: PredictRequest):
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

This code already has several good habits:

- typed request and response models
- explicit `HTTPException` usage
- health and version endpoints
- configuration-driven behavior through `settings`

But Week 4 is where I learned to see the remaining engineering gaps more clearly.

### The Important API Questions

The Week 4 mentorship materials call out four things that matter in a deployable service:

1. Is the response schema consistent with what the endpoint actually returns?
2. Is the model loaded once or recreated on every request?
3. Can the service expose readiness separately from liveness?
4. Do failures return a consistent error shape?

That reframed the way I looked at `/predict`. The endpoint works, but it also reveals exactly what production intuition starts to notice:

- the model is initialized per request
- readiness is not checked separately
- error formatting is still mostly local to the endpoint
- the serving contract deserves stricter hardening as the project matures

That was the real learning: Week 4 taught me how to critique my own API from an operations perspective.

---

## Training and Evaluation Became Supporting Infrastructure

In Week 4, the API stopped being the only main character. Training and evaluation became supporting infrastructure that had to produce artifacts a service could eventually depend on.

The repository README summarizes the intended end-to-end pipeline like this:

1. data sources live under `src/data/`
2. loading and preprocessing happen in `src/data/dataset_utils.py` and the training flow
3. the dataset is split using config-driven settings
4. training runs through `src/training/train.py`
5. evaluation runs through `src/training/evaluate.py`
6. artifacts are saved for later use
7. the API serves inference from that larger system context

That pipeline thinking is what makes the project feel like engineering rather than experimentation.

### Training Loop

Code reference: [train.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/training/train.py)

```python
from pathlib import Path

import joblib
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.model_selection import train_test_split

from src.config.settings import settings
from src.data.dataset_utils import load_dataset
from src.models.logistic_regresion import LogisticRegressionModel


def train_model(data_path: Path) -> None:
    print(f"Loading dataset from {data_path}...")
    df = load_dataset(data_path)

    vectorizer = TfidfVectorizer()
    df['text'] = vectorizer.fit_transform(df['text']).toarray()
    df['label'] = df['label'].map({'negative': 0, 'positive': 1})

    X = df.drop(columns=['label'])
    y = df['label']

    train_test_split_ratio = settings.train_test_split
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=train_test_split_ratio
    )

    if settings.model_type == "log_reg":
        model = LogisticRegressionModel()
    else:
        raise ValueError(f"Unsupported model type: {settings.model_type}")

    epochs = settings.train_epochs
    for epoch in range(epochs):
        print(f"Epoch {epoch + 1}/{epochs}")
        model.train(X_train, y_train)

    predictions = model.predict(X_test)
    accuracy = (predictions == y_test).mean()
    mismatches = (predictions != y_test).sum()

    model_output_path = Path(settings.model_output_path)
    model_output_path.parent.mkdir(parents=True, exist_ok=True)
    joblib.dump(model, model_output_path)
```

This script is still intentionally simple, but it already captures several production habits:

- load data from a configured path
- split data using a configured ratio
- choose model type through config
- train for a configured number of epochs
- save an artifact to disk

Week 4 sharpened the meaning of these steps for me. They are not just “ML steps.” They are the parts of the pipeline that make reproducibility possible.

### Evaluation Loop

Code reference: [evaluate.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/training/evaluate.py)

```python
from typing import Callable

from src.config.settings import settings
from src.models.logistic_regresion import LogisticRegressionModel


def evaluate_validation_model(
    model: Callable[[str], dict[str, object]], dataset: list[dict[str, object]]
) -> dict[str, float]:
    if settings.model_type == "log_reg":
        model = LogisticRegressionModel()
    else:
        raise ValueError(f"Unsupported model type: {settings.model_type}")

    sample_data = [item['text'] for item in dataset]
    predictions = model.predict(sample_data)

    accuracy = 0.0
    miss_matches = 0
    False_positives = 0
    False_negatives = 0

    print(
        f"EVALUATION REPORT\n----------------------\n Samples: {len(dataset)}, \n Count of Predictions: {len(predictions)} Accuracy: {accuracy}, Mismatches: {miss_matches}, False Positives: {False_positives}, False Negatives: {False_negatives}"
    )
    return {"accuracy": accuracy, "mismatches": miss_matches}
```

Week 4 did not require a sophisticated evaluation platform. It required the habit of separating training from evaluation and making the metrics explicit outputs.

The mentorship README pushes this further by recommending a clearer binary-sentiment setup, saved metrics, and a reusable scikit-learn `Pipeline` for text vectorization and logistic regression. Even where the current code is still lightweight, the engineering lesson is already visible: training and evaluation should become stable, inspectable stages, not just ad hoc experimentation.

### Tests Still Matter Here Too

Code reference: [test_eval.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/tests/test_eval.py)

```python
from src.models.object_detector import ObjectDetector
from src.training.evaluate import evaluate_validation_model

dummy_model = ObjectDetector()


def test_evaluate_model_returns_dict():
    dataset = [{"text": "hi", "expected": False}]
    result = evaluate_validation_model(dummy_model, dataset)
    assert isinstance(result, dict)
    assert "accuracy" in result
```

Even a small test like this does something important: it locks in the contract of the evaluation function and keeps the pipeline from drifting silently.

---

## Automation Through the Makefile

The repository already uses a Makefile to standardize common commands.

Code reference: [Makefile](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/Makefile)

```makefile
setup:
	pip install --upgrade pip
	pip install -r requirements.txt
	pre-commit install

test:
	echo "Running tests!"
	pytest -v --maxfail=1

run:
	python -m src.main

docker-build:
	docker build -t ai-mentorship-app:latest .

serve-local:
	docker compose up --build

train:
	python -m src.training.train
```

I liked this section because it made the project feel easier to hand off. Instead of telling someone “first install this, then remember that script, then run something else,” the repo starts giving them stable entry points.

Week 4 made me appreciate that automation is not only about speed. It is also about reducing ambiguity.

---

## Docker and CI as Portfolio Requirements

One of the most useful Week 4 lessons was that containerization and CI are not “nice-to-have extras” for a portfolio project. They are signals of engineering maturity.

The Week 4 README treats these as explicit deliverables:

- a `Dockerfile` that runs the service
- a `.dockerignore`
- local validation via `docker build` and `docker run`
- a GitHub Actions workflow that checks linting and tests on push and pull request

The mentoring material even gives the expected Docker flow:

```bash
docker build -t ai-mentorship-app:latest .
docker run --env-file .env.dev -p 8000:8000 ai-mentorship-app:latest
```

And the service validation checks it expects are straightforward:

```bash
curl http://localhost:8000/healthz
curl http://localhost:8000/readyz
curl http://localhost:8000/version
```

What mattered most to me here was not memorizing Docker syntax. It was understanding why these pieces matter:

- Docker makes the runtime environment reproducible
- CI makes code quality checks automatic instead of optional
- health and readiness endpoints make operations observable
- automated checks make the repo safer to change over time

Even when those pieces are still being completed, learning to think of them as required parts of the system is a big step up.

---

## What Week 4 Changed for Me

Week 4 taught me to see a difference between a project that demonstrates knowledge and a project that demonstrates engineering judgment.

That difference shows up in questions like:

- can another person run this without tribal knowledge?
- can I inspect the health of the service quickly?
- can I trust tests to catch a regression?
- are artifacts saved in a way that future work can build on?
- does the API expose predictable contracts and predictable failures?

Those questions are what make a small project feel real.

---

## Final Wrap-Up

By the end of this mentorship, the biggest transformation was not just technical. It was mental.

I moved from thinking mainly about scripts and models to thinking about complete systems:

- repositories that are structured intentionally
- APIs with explicit contracts
- configuration that lives outside code
- training and evaluation as reproducible stages
- metrics tied to business and product outcomes
- deployment and CI as part of engineering, not afterthoughts

That is the main thing I will carry forward: AI engineering is not just about building a model that works. It is about building a system that can be understood, tested, operated, and improved.

*← [Back to the Introduction](/post/my-post/)*

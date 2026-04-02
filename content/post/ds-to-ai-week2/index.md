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
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_env: str = "dev"
    debug: bool = True
    api_version: str = "v1"
    log_level: str = "info"
    host: str = "0.0.0.0"
    port: int = 8080

    @property
    def is_production(self) -> bool:
        return self.app_env == "prod"

settings = Settings(_env_file=f".env.{Settings().app_env}")
```

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

Example endpoint contract:

```python
from fastapi import FastAPI
from pydantic import BaseModel

class ValidationRequest(BaseModel):
    text: str

class ValidationResponse(BaseModel):
    is_valid: bool

app = FastAPI()

@app.post("/validate", response_model=ValidationResponse)
def validate(payload: ValidationRequest) -> ValidationResponse:
    return ValidationResponse(is_valid=len(payload.text.strip()) > 0)
```

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

Compact pattern:

Code reference: [Predict endpoint implementation](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/backend/app.py)

```python
from fastapi import HTTPException

@app.post("/predict", response_model=PredictResponse)
def predict(payload: PredictRequest) -> PredictResponse:
    if settings.app_env != "dev" and len(payload.text) < settings.min_text_length:
        raise HTTPException(status_code=422, detail="Input text too short")
    try:
        score = pseudo_infer(payload.text)
        return PredictResponse(score=score)
    except Exception as exc:
        raise HTTPException(status_code=500, detail="Inference failed") from exc
```

This pattern mirrors real deployment workflows where behavior changes by environment without modifying application code.

---

## Exercises I Used to Internalize the Concepts

### 1) Configurable Validation Threshold

I tested behavior by varying validation thresholds through environment values instead of code edits.

Why this matters:

- Enables behavior changes without redeploying code logic
- Supports staged rollout and test/prod divergence
- Builds confidence in environment-driven controls

Example:

Code reference: [Config and threshold wiring](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/config/settings.py)

```env
MIN_TEXT_LENGTH=5
```

```python
if len(payload.text) < settings.min_text_length:
    raise HTTPException(status_code=422, detail="Input below threshold")
```

### 2) Configure Model and Test Inference

I treated inference as a strict contract: typed input, validated execution path, typed output, and explicit failure modes.

Quick test call:

Code reference: [API app for local endpoint testing](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/backend/app.py)

```bash
curl -X POST http://127.0.0.1:8000/predict \
    -H "Content-Type: application/json" \
    -d '{"text":"sample prompt"}'
```

This helps catch integration bugs early when frontend or external services call the API.

---

## Testing: Config and Endpoint Reliability

Week 2 reused Week 1 testing principles and targeted infrastructure-critical components.

Code reference: [Tests directory](https://github.com/saranobrega/ai-engineering-mentorship-cm/tree/main/tests)

I focused tests on:

- Configuration loading correctness
- API endpoint behavior
- Edge-case handling and safe failures

In practical terms, these tests answer:

- Are env variables loaded as expected?
- Do endpoints return correct status codes and response schema?
- Do invalid payloads fail gracefully?

Minimal tests looked like:

```python
def test_healthz(client):
    r = client.get("/healthz")
    assert r.status_code == 200

def test_validate_rejects_empty(client):
    r = client.post("/validate", json={"text": ""})
    assert r.status_code in (200, 422)
```

This test layer reduced regression risk and made refactoring safer.

---

## Engineering Habits That Paid Off

A few engineering practices had direct payoff this week:

Code reference: [Engineering practices checklist](https://github.com/saranobrega/ai-engineering-mentorship-cm/tree/main/week_2)

- Avoid magic numbers and strings; prefer named constants or settings
- Use meaningful, consistent names across modules
- Prefer early returns over deeply nested logic
- Avoid long parameter lists; group with models/objects
- Keep functions focused on one responsibility
- Do not repeat yourself (DRY)
- Comment the reason, not the obvious action
- Write commit messages that explain intent

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

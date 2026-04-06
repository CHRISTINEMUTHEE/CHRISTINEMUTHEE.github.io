---
title: "Week 3: KPIs, Architecture Thinking, and My First Training Pipeline"
date: 2026-02-24
summary: "Week 3 of the AI Engineering Mentorship: moving from API building to AI systems thinking through KPIs, architecture diagrams, synthetic data, training, and offline evaluation."
tags: ["AI Engineering", "MLOps", "FastAPI", "Training Pipelines", "Evaluation", "KPIs"]
categories: ["AI Engineering Journey"]
---

*Part of the [From Data Scientist to AI Engineer](/post/my-post/) series. ← [Week 2](/post/ds-to-ai-week2/)*

---

## Week 3 Focus

Week 3 changed the way I think about AI work.

Up to this point, I had been focused on building code that runs: endpoints, validation, configuration, and a basic backend. This week pushed me into a more senior mindset: AI engineering is not just about writing model code, it is about designing systems that solve real problems and can be measured end to end.

This week mixed two themes:

- product and architecture thinking
- my first real ML engineering pipeline: synthetic data -> training -> evaluation -> artifacts

By the end of the week, the project no longer felt like “just an API.” It started to feel like a small but complete AI system.

---

## Product and Architecture Mindset

The biggest mental shift was this:

> Good AI engineers do not start with “Which model should I train?”  
> They start with “What problem are we solving, and how will we know it works?”

That sounds simple, but it changes everything.

Instead of obsessing over training first, I had to reason about the tradeoffs companies actually care about:

- **Quality**: is the output correct?
- **Latency**: is it fast enough for the user experience?
- **Cost**: can the system scale without becoming too expensive?
- **Safety**: does it fail predictably and gracefully?
- **User impact**: does it solve a real problem for a real user?

This is the kind of thinking that shows up heavily in interviews. Two candidates may both know Python and FastAPI, but the stronger engineer is usually the one who can explain why a system matters, where it can break, and what metric proves it is useful.

### Reflection Questions I Used

These are the exact style of questions that sharpen product intuition:

1. Which endpoint from Week 2 would break first if traffic increased by 10x? Why?
2. Would the `/predict` endpoint be acceptable in a production app? What is missing?
3. For this model scenario, what matters most: latency, accuracy, or cost?
4. If the model is wrong 1% of the time, what is the real-world consequence?
5. What type of user would actually rely on the output of the model?

These look like reflection prompts, but they are also design prompts. They force you to connect code decisions to user risk and business outcomes.

---

## AI KPIs and Metrics

Before training anything, I needed to answer two questions:

- How will I measure quality?
- How will I measure business impact?

The mentorship split this into three layers: offline metrics, online metrics, and business KPIs.

### Offline Metrics

Offline metrics are measured before deployment, using historical or synthetic data. They tell you how the model behaves in controlled conditions.

Examples discussed in the mentorship:

- accuracy
- precision, recall, and F1
- ROC-AUC
- latency per prediction
- training time
- model size
- memory footprint

These answer one question:

> Is the model good under controlled conditions?

### Online Metrics

Online metrics are measured after deployment, using real traffic and real users. They tell you what happens when the model meets reality.

Examples discussed in the mentorship:

- production error rate
- user satisfaction or rating
- fallback rate
- conversion uplift
- time saved
- API latency at peak load
- incremental revenue
- cost per 1,000 inferences

These answer a different question:

> Is the model valuable and usable in real conditions?

### Business KPIs

Business KPIs are the product-level outcomes the team actually cares about. These depend on the use case.

For classification systems such as fraud detection, churn prediction, or credit scoring, useful KPIs might be:

- lower false positives so fewer legitimate users are blocked
- lower false negatives so fewer risky cases slip through
- reduced manual review time per case
- reduced financial loss
- faster triaging for support teams
- lower customer churn through earlier intervention

For chatbots or LLM assistants, useful KPIs might be:

- reduced manual agent workload
- lower handling time per user question
- improved CSAT or NPS
- higher task completion rate
- lower cost per interaction

The lesson was simple: model performance only matters if it maps to business value.

### Exercise 1

I treated this as the most important conceptual exercise of the week.

If the model became a real product feature, write down:

1. Three offline metrics you would track.
2. Three online metrics you would track.
3. One business KPI that proves the model adds value.
4. Which metric you would optimize first, and why.

This is the kind of exercise that builds metric intuition. It also gives you stronger answers in system design and ML interview conversations.

---

## My First Architecture Diagram

By Week 2, I had already built several important pieces:

- a backend
- API endpoints
- typed request and response models
- configurable behavior
- utilities
- the early skeleton of training and model modules

Week 3 asked me to stop looking at these as isolated files and start seeing them as a connected system.

The mentorship gave this high-level flow:

```text
      ┌────────────┐
      │   Client   │
      └─────┬──────┘
            │ HTTP Request
            ▼
     ┌───────────────┐
     │    FastAPI    │
     └──────┬────────┘
            │
   Validate / Preprocess
            ▼
   ┌───────────────────┐
   │ Model Inference   │
   └───────┬───────────┘
           │
    Post-Processing
           ▼
   ┌───────────────────┐
   │ Response + Logs   │
   └───────────────────┘
```

Then Week 3 expanded that thinking into a larger engineering flow:

```text
Synthetic Data -> Training -> Evaluation -> Artifacts -> API Inference
```

That was a key learning moment for me. An AI system is not just one prediction function. It is a chain of stages, and each stage can fail, drift, slow down, or become expensive.

### Exercise 2

The exercise was to draw an end-to-end architecture diagram while keeping it conceptual.

The rules were:

- think in stages, not implementation details
- think in modules, not functions
- think in flows, not loops or syntax
- keep each box to one or two words

Suggested boxes:

- FastAPI
- Validation
- Model
- Synthetic Data
- Training
- Evaluation
- Artifacts
- Settings

The reflection question behind the diagram was the real lesson:

> What part of the architecture is most likely to break if I scale this system, and why?

---

## Synthetic Data: My First Dataset

The project still was not using real production data, and that was intentional.

The goal was not to build a state-of-the-art model. The goal was to practice the shape of real ML work safely:

- define a problem
- define an input and output schema
- load data consistently
- split it into train and test sets
- compute offline metrics
- connect it later to the API

The mentorship recommended small, simple working problems such as:

- text validation labels
- feedback sentiment
- routing flags such as `ai_can_handle` vs `escalate_to_human`

The repository itself points the training flow to a sentiment-style CSV via settings:

Code reference: [settings.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/config/settings.py)

```python
dataset_path: str = Field(
    default="/src/data/synthetic/feedback_sentiment_clean.csv",
    alias='DATASET_PATH',
    description="Path to the dataset file.",
)
```

And data loading is intentionally simple:

Code reference: [dataset_utils.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/data/dataset_utils.py)

```python
from pathlib import Path

import pandas as pd


def load_dataset(csv_path: Path) -> pd.DataFrame:
    '''Load dataset from a CSV file'''
    path = Path(csv_path)
    if not path.exists():
        raise FileNotFoundError(f"File not found: {csv_path}")
    df = pd.read_csv(csv_path)
    print(f"Loaded {len(df)} rows from {csv_path} with shape {df.shape}")
    return df
```

That simplicity is the point. Before adding feature stores, data contracts, or orchestration tools, I first needed to understand the flow clearly.

---

## Building My First Configurable Training Loop

Once the dataset and system shape were clear, the next step was training.

This week reinforced a principle I now care about a lot: training behavior should be controlled by configuration, not hardcoded decisions in the script.

In the repository, the relevant settings are already defined in `Settings`:

Code reference: [settings.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/config/settings.py)

```python
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
```

The training loop then uses those settings directly.

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
    '''Function to train the selected model'''

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

What I liked about this stage is that it made the training pipeline concrete:

- data comes from a configured path
- the split ratio is configured
- model selection is configured
- epochs are configured
- artifact output path is configured

That is the beginning of reproducibility.

To make it easy to run, the repository adds a dedicated Make target:

Code reference: [Makefile](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/Makefile)

```makefile
train:
	python -m src.training.train
```

This is a small detail, but it matters. A repeatable command is part of good engineering hygiene.

---

## Building the Offline Evaluation Loop

Training alone is not enough. Week 3 also introduced the idea that every training step should be paired with evaluation, even if the implementation starts small.

The repository's evaluation module is still intentionally lightweight, but it already shows the expected shape of the logic:

Code reference: [evaluate.py](https://github.com/saranobrega/ai-engineering-mentorship-cm/blob/main/src/training/evaluate.py)

```python
from typing import Callable

from src.config.settings import settings
from src.models.logistic_regresion import LogisticRegressionModel


def evaluate_validation_model(
    model: Callable[[str], dict[str, object]], dataset: list[dict[str, object]]
) -> dict[str, float]:
    '''Evaluate the /validate logic on your dataset.
    compute accuracy, precision , recall etc'''

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

What matters here is not that the metrics are fully mature yet. What matters is that the evaluation stage exists as a separate concern.

That separation teaches several important habits:

- evaluation should be callable independently of training
- metrics should be explicit outputs, not hidden side effects
- the model should be compared against expected behavior on a dataset
- reporting should be structured enough to evolve over time

The first test for that module is also already in place:

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

This was a helpful lesson in itself: even a placeholder evaluation pipeline is worth testing early, because it forces you to think clearly about what the function returns and how future metrics will be surfaced.

---

## What Week 3 Changed for Me

Week 3 was less about perfect code and more about engineering reasoning.

I learned how to zoom out and ask better questions:

- what metric proves this model is useful?
- what part of the system is likely to fail first at scale?
- what should be configurable vs hardcoded?
- how do training and evaluation become reproducible instead of ad hoc?
- how do I connect ML outputs to product outcomes?

That mindset shift mattered more than any one function or file.

---

## Week 3 Wrap-Up

By the end of the week, I had a clearer picture of what AI engineering actually involves.

I built or reasoned through:

- KPI and metrics thinking for AI systems
- product-oriented reflection questions
- a high-level system architecture model
- a synthetic-data-driven training pipeline
- configuration-driven training behavior
- an initial offline evaluation loop
- early tests for the evaluation layer

The main takeaway: the jump from data science to AI engineering is not just about adding APIs or deployment. It is about learning to connect metrics, architecture, training, inference, and business outcomes into one system.

**Continue to Week 4 →** [Automating and Deploying the ML Workflow](/post/ds-to-ai-week4/)

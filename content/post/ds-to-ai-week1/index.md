---
title: "Week 1: Building the Foundation — Repo Structure, Tooling & Your First Tests"
date: 2026-02-10
summary: "Week 1 of the AI Engineering Mentorship: a crash course in everything a bootcamp should have taught — clean repo structure, Makefiles, pre-commit hooks, type checking, and your first passing tests."
tags: ["AI Engineering", "Unit Testing", "Python", "Repository Structure", "Type Checking", "mypy", "pytest", "FastAPI", "Makefile", "pre-commit", "Git Workflow"]
categories: ["AI Engineering Journey"]
---

*This post is part of a series on my transition from Data Scientist to AI Engineer. If you are new here, start with the [Introduction](/post/my-post/).*

---

## What I Set Out to Learn

Week 1 had a clear, challenging mandate: stop coding like a lone Data Scientist and start coding like a software engineer who works on a team.

In practice, that meant building several foundational muscles at once:

- **Repository structure** — How do professional ML projects organise their code?
- **Makefile automation** — How do you eliminate the "but how do I run this?" problem?
- **Linting, formatting and type checking** — How do you enforce code quality before code review?
- **Pre-commit hooks** — How do you make quality checks automatic rather than manual?
- **Unit testing** — How do you test code that involves data and models?
- **Git workflow** — How does real collaborative development actually work, branch by branch?

None of these felt urgent to me before this week. By the end of it, I could not imagine building without any of them.

Honestly? This week felt like a crash course in everything my bootcamp should have taught me.

---

## How the Mentorship Worked: Feature Branches & Pull Requests

Before I get into the technical content, it is worth describing *how* this mentorship was structured — because the format was itself a lesson in software engineering.

Each week followed the same collaborative pattern:

1. I created a **feature branch** for the week's work: `week1_feature`
2. I made all my changes on that branch, pushing regularly to the remote
3. My mentor **reviewed my pull request** — commenting on design, suggesting better approaches, and questioning my decisions
4. Once the work was satisfactory, I requested a merge into the `dev` branch
5. Over four weeks, all the work accumulated in `dev` — merging cleanly into `main` at the end

```
main
 └── dev
      ├── week1_feature  ← create, build, push, PR, review, merge
      ├── week2_feature
      ├── week3_feature
      └── week4_feature
```

I found this format fascinating — and, honestly, fun. There is something deeply satisfying about seeing a pull request reviewed, discussed, and merged. It made the work feel real in a way that solo notebook sessions never quite do.

I should also mention: at some point during this mentorship, I deleted an entire repository. Completely. Gone. *Thank God for AI* 😂 — I genuinely do not know how the engineers who came before us maintained a normal heart rate without it.

---

## The Challenge: Letting Go of the Notebook Mindset

My instinct, honed over years of Data Science, was to keep everything in one place: one notebook, everything visible, cells run top to bottom. It felt clean. It felt *fast*.

That instinct is the enemy of production code.

Week 1 forced me to sit with a hard truth: **a notebook is a powerful exploration tool, but a terrible deployment unit.** You cannot import a notebook function into another file. You cannot test a notebook cell in isolation. You cannot meaningfully diff a notebook in a pull request.

The mental shift required was not about learning new syntax. It was about accepting that the code you write is not just for *you today* — it is for *your team, your future self, and the infrastructure that will run it tomorrow.*

---

## Step 1: Setting Up the Repository

Repository setup is not just creating a folder and running `git init`. Done properly, it involves:

- Creating and activating a **virtual environment** to isolate dependencies
- Defining the **final directory structure** and the responsibility of every folder
- Setting up the **configuration files** that tools like linters and type checkers will read
- Writing the **`.gitignore`** so local data and environment files never end up on GitHub

Here is the actual structure I built for my project:

```
ai-engineering-mentorship-cm/
│
├── src/                        # All project source code
│   ├── __init__.py
│   ├── main.py                 # App entry point (FastAPI server via Uvicorn)
│   │
│   ├── backend/                # Serving logic (FastAPI routes)
│   │   ├── __init__.py
│   │   └── app.py
│   │
│   ├── data/                   # Data access & management
│   │   ├── __init__.py
│   │   ├── preprocessing.py    # Cleaning, resizing, transformations
│   │   └── dataset_utils.py    # Functions to load or prepare datasets
│   │
│   ├── models/                 # Model definition & loading
│   │   ├── __init__.py
│   │   ├── base_model.py       # Abstract base class or interface
│   │   └── object_detector.py  # Placeholder for the detection model
│   │
│   ├── training/               # Training and evaluation workflows
│   │   ├── __init__.py
│   │   ├── train.py            # Training loop
│   │   └── evaluate.py         # Evaluation script
│   │
│   └── utils/                  # Generic helpers
│       ├── __init__.py
│       └── io_utils.py         # File and logging utilities
│
├── tests/                      # All test files live here
│   ├── test_data_utils.py
│   └── test_example.py
│
├── notebooks/                  # Prototyping & experiments (git-ignored)
│   └── .gitkeep
│
├── data/                       # Local data folder (git-ignored)
│   └── .gitkeep
│
├── requirements.txt
├── pyproject.toml              # Project metadata, tool configs
├── Makefile                    # Command automation
├── .pre-commit-config.yaml     # Pre-commit hook definitions
├── .gitignore
└── README.md
```

A few things worth highlighting:

- `notebooks/` still exists. Exploration is valid. But it is **git-ignored** and clearly separated from anything that ships to production.
- `data/` is also git-ignored. Raw datasets do not belong in version control.
- Every directory has an `__init__.py`. This is what makes `src` a proper Python package you can import from.
- `main.py` serves as the entry point for the **FastAPI** application, which is run using **Uvicorn** — the standard ASGI server for FastAPI applications.

In a real engineering team, different engineers would likely own different parts of this structure. I was working alone, so I built all of it — which turned out to be an excellent way to understand how every piece connects.

---

## Step 2: pyproject.toml — One File to Configure Them All

Before this mentorship, I had a vague awareness of `pyproject.toml` but had never really used it properly. It is the modern Python standard for describing your project *and* configuring all the tools that work on it — in one place.

Here is a simplified version of what mine contained:

```toml
[project]
name = "ai-engineering-mentorship-cm"
version = "0.1.0"
description = "An AI engineering project following SE best practices"
requires-python = ">=3.10"

[tool.mypy]
strict = true
ignore_missing_imports = true

[tool.ruff]
line-length = 88
select = ["E", "F", "I"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Having all tool configurations in one file means your linter, type checker, and test runner all read from the same source of truth. No more scattered `.mypy.ini`, `.flake8`, and `pytest.ini` files cluttering the root.

---

## Step 3: The Makefile — Automation as Documentation

A `Makefile` is, at its core, a collection of named commands. But it is also something more: **executable documentation.** Any engineer picking up the repository for the first time can run `make help` and immediately understand how to interact with the project.

My Makefile included targets for:

```makefile
.PHONY: setup test lint run-app docker-build ci

setup:
	python -m venv .venv && source .venv/bin/activate && pip install -e ".[dev]"

lint:
	ruff check src/ && mypy src/

test:
	pytest tests/ -v

run-app:
	uvicorn src.main:app --reload

docker-build:
	docker build -t ai-mentorship-app .

ci: lint test
```

The value became immediately obvious. Before the Makefile, I had to remember — or look up — the exact command to run linting, start the app, or execute the tests. After the Makefile, everything is a single, memorable `make <target>` command. And because the Makefile lives in version control, the commands are the same for every developer, on every machine, in every environment.

---

## Step 4: Pre-commit Hooks — Quality Checks That Run Themselves

A **pre-commit hook** is a script that runs automatically every time you attempt a `git commit`. If the checks fail, the commit is blocked until you fix the issues.

This was one of the most satisfying things I set up all week, because it removes a class of problems entirely. You no longer have to *remember* to run the linter before committing — the hook does it for you.

My `.pre-commit-config.yaml` configured:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff          # Linting
      - id: ruff-format   # Formatting

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.9.0
    hooks:
      - id: mypy          # Type checking

  - repo: https://github.com/pycqa/isort
    rev: 5.13.2
    hooks:
      - id: isort         # Import sorting
```

Every commit now automatically runs linting, formatting, type checking, and import sorting. If any of them fail, the commit does not go through. This means no unformatted code ever reaches the remote branch — and no reviewer ever has to leave a comment about a missing newline at the end of a file.

---

## Step 5: Unit Testing — What It Actually Means in ML

I had a deeply-held misconception about unit testing in machine learning: *"How do you unit test something probabilistic? Models are not deterministic."*

This is exactly the wrong frame. **You are not testing the model. You are testing the code around the model.**

The helper functions I unit tested in Week 1 were in `src/utils/io_utils.py` and `src/data/dataset_utils.py` — things like loading data from a file path, writing outputs to a directory, and validating data schemas. Entirely deterministic. Entirely testable.

```python
# tests/test_data_utils.py
import pandas as pd
from src.data.dataset_utils import load_dataset

def test_load_dataset_returns_dataframe(tmp_path):
    # Create a small CSV in a temporary directory
    sample = pd.DataFrame({"x": [1, 2, 3], "y": [4, 5, 6]})
    filepath = tmp_path / "sample.csv"
    sample.to_csv(filepath, index=False)

    result = load_dataset(filepath)
    assert isinstance(result, pd.DataFrame)
    assert result.shape == (3, 2)

def test_load_dataset_raises_on_missing_file():
    with pytest.raises(FileNotFoundError):
        load_dataset("data/nonexistent.csv")
```

The payoff was immediate. Every time I refactored something, `pytest` told me within seconds whether I had broken a contract. That confidence is genuinely addictive.

---

## Step 6: Type Checking — The Gift of Being Explicit

Before this week, a typical function in my codebase looked like this:

```python
def load_data(path, config, transform):
    ...
```

What is `path`? A string? A `Path` object? What is `transform`? A function? A class? A flag? The signature communicated nothing.

Type hints changed this completely:

```python
from pathlib import Path
import pandas as pd
from collections.abc import Callable

def load_data(
    path: Path,
    config: dict[str, str],
    transform: Callable[[pd.DataFrame], pd.DataFrame] | None = None,
) -> pd.DataFrame:
    ...
```

Running `mypy src/` surfaced bugs I would never have caught until runtime. More importantly, writing type hints forced me to *think clearly* about what flows through each function — which turns out to be one of the most effective ways to catch bad design early.

---

## My Honest Reflections

This week was humbling in the best possible way. I had been writing Python for years, and I thought I was reasonably good at it. Turns out I was good at writing *exploratory* Python. Production Python is a different discipline.

The most valuable thing I took away was not a specific tool. It was the discipline of **making things explicit**: explicit structure, explicit commands, explicit types, explicit contracts. Every act of making something explicit in code reduces the cognitive load on every person — including future me — who has to read or run it later.

And the pull request workflow? I cannot go back to working without it. Having another set of eyes on my design decisions, and having to explain *why* I made a choice in a PR description, made my thinking sharper every single week.

---

## What I Walked Away With

By the end of Week 1:

- ✅ A clean, professional ML project structure with `src/`, `tests/`, `notebooks/`, and `data/` clearly separated
- ✅ A `pyproject.toml` configuring the project, linter, type checker, and test runner in one place
- ✅ A `Makefile` encoding setup, linting, testing, and deployment commands
- ✅ Pre-commit hooks running linting, formatting, type checks, and import sorting on every commit automatically
- ✅ At least two passing unit tests on helper functions for loading and writing data
- ✅ A first FastAPI server running locally via Uvicorn
- ✅ My first feature branch, pull request, and approved merge

My repository went from looking like a school project to looking like something I would not be embarrassed to put in front of a senior engineer.

---

## Try It Yourself

You can explore the Week 1 codebase — and try to reproduce it — here:
[github.com/saranobrega/ai-engineering-mentorship-cm](https://github.com/saranobrega/ai-engineering-mentorship-cm.git)

---

## Up Next

Week 2 brought a different kind of challenge: how do you manage configurations across dozens of experiments without hardcoding values everywhere? And how do you think about model training not just as a technical problem, but as a *business decision*?

**Continue to Week 2 →** [Configuration Management & Thinking Like an Engineer](/post/ds-to-ai-week2/)

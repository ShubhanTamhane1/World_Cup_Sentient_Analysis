# World Cup Sentiment Tracker — Project Context

## What This Is
A real-time sentiment analysis pipeline that ingests live Reddit commentary during World Cup matches, runs sentiment scoring using both a trained sklearn/HuggingFace model and the Anthropic API in parallel, tracks experiments with MLflow, serves results via a FastAPI layer, and is containerized with Docker and deployed on AWS ECS.

This project exists to fill specific resume gaps: reliable data pipeline construction, model packaging, experiment tracking, and containerized deployment.

---

## Architecture Overview

```
Reddit (praw) → Raw JSON → S3 (partitioned by match/timestamp)
                                      ↓
                            Sentiment Processing
                            ├── sklearn / HuggingFace model
                            └── Anthropic API (claude-sonnet-4-6)
                                      ↓
                               MLflow (experiment tracking)
                                      ↓
                          PostgreSQL on AWS RDS (enriched records)
                                      ↓
                              FastAPI (serving layer)
                                      ↓
                           Docker → AWS ECS (deployment)
```

---

## Data Strategy

**Primary source:** Reddit live match threads via `praw` (free, generous rate limits)
- Subreddits: r/soccer, r/worldcup, r/football
- Target: live comment threads during matches, updated in real time

**Secondary source:** Kaggle World Cup 2022 tweet dataset
- Used for model training, evaluation, and MLflow experiment logging
- Gives proper labeled data for sklearn/HuggingFace model comparison vs. Anthropic API

**Why not X/Twitter:** API access is prohibitively expensive at meaningful volume.

---

## Storage Layout

| Layer | Storage | Purpose |
|---|---|---|
| Raw ingestion | S3 | JSON comment dumps, partitioned by `matches/{date}/{match}/raw/` |
| Processed/enriched | PostgreSQL (RDS) | Comments + sentiment scores, served by FastAPI |
| MLflow artifacts | S3 | Model weights, prompt configs, run metadata |

**Avoid:** Writing raw data directly to DynamoDB — expensive for analytical queries.

---

## Sentiment Processing

Two models run in parallel on every comment, results logged to MLflow:
1. **sklearn or HuggingFace model** — trained on Kaggle tweet dataset, fast and cheap per inference
2. **Anthropic API** — `claude-sonnet-4-6`, used for nuanced/contextual scoring

Schema for enriched records stored in Postgres:
```
comment_id, match_id, subreddit, body, timestamp, match_minute,
sentiment_sklearn, confidence_sklearn,
sentiment_anthropic, confidence_anthropic,
model_version, prompt_version
```

---

## Experiment Tracking (MLflow)

MLflow tracks:
- sklearn/HuggingFace model variants (vectorizer choice, classifier type, hyperparams)
- Anthropic prompt versions (system prompt wording, temperature, output format)
- Agreement rate between the two models
- Per-match accuracy against any available ground truth

This is intentionally rigorous — the dual-model comparison is what makes the MLflow usage defensible in an interview.

---

## Tech Stack

| Category | Tools |
|---|---|
| Data ingestion | `praw`, Python |
| Storage | AWS S3, PostgreSQL on RDS |
| ML / NLP | scikit-learn, HuggingFace Transformers |
| LLM | Anthropic API (`claude-sonnet-4-6`) |
| Experiment tracking | MLflow |
| API layer | FastAPI |
| Containerization | Docker |
| Deployment | AWS ECS |
| Orchestration | AWS EventBridge / Lambda (optional) |

---

## Developer Background (Shubhan)

- UConn Statistical Data Science, graduating December 2026
- Internship at Travelers Insurance (Data Engineering): AWS Lambda, DynamoDB, EventBridge, Snowflake, Databricks
- Projects: UConn Baseball pitcher fatigue detection (logistic regression, ROC-AUC 0.976), NFL QB archetypes clustering (PCA + K-Means), Auto Insights Generator (Anthropic API, open-source PyPI package)
- Strong in: Python (sklearn, pandas, statsmodels), SQL, AWS, Anthropic API
- New with this project: Docker, ECS, FastAPI, MLflow, streaming pipelines

---

## Resume Gaps This Project Fills

- Reliable end-to-end data pipeline (S3 → processing → Postgres)
- Model packaging and serving (Docker + FastAPI)
- Experiment tracking (MLflow with real model comparison, not just prompt versioning)
- Containerized cloud deployment (Docker → ECS)

---

## Rough Build Order

1. Reddit ingestion → S3 (get data flowing before anything else)
2. Sentiment processing (sklearn model trained on Kaggle data + Anthropic API call)
3. MLflow logging for both model paths
4. Postgres schema + enriched record writes
5. FastAPI endpoints (serve sentiment by match, by time interval, etc.)
6. Dockerize the application
7. Deploy to AWS ECS
8. Polish: dashboard, writeup, README

**Don't containerize until the pipeline works end to end locally.**

---

## Cost Notes

- S3: near-free at this scale
- RDS t3.micro: ~$15/mo or free tier eligible
- Anthropic API: batch processing comments, not per-keystroke — keep costs low by processing asynchronously
- ECS: minimal at low traffic; shut down between matches
- Snowflake: intentionally excluded — overkill and expensive for this scale

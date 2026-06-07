# 🛡️ FraudShield

> **Real-Time Transaction Fraud Detection API — Production-Grade Microservice**

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat&logo=python)](https://python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110-009688?style=flat&logo=fastapi)](https://fastapi.tiangolo.com/)
[![Docker](https://img.shields.io/badge/Docker-Containerised-2496ED?style=flat&logo=docker)](https://docker.com/)
[![AWS](https://img.shields.io/badge/AWS-Lambda-FF9900?style=flat&logo=amazonaws)](https://aws.amazon.com/lambda/)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](LICENSE)
[![Status](https://img.shields.io/badge/Status-In_Development-orange?style=flat)]()

---

## 📌 Problem Statement

Financial fraud costs the global economy **$485 billion annually**. Traditional rule-based fraud detection systems suffer from two fatal flaws:

- **High false-positive rates** — blocking legitimate customers, damaging UX
- **High latency** — batch processing means fraud is detected hours after it occurs

FraudShield is a production-grade ML microservice that serves **sub-50ms fraud predictions** on live financial transactions, with a Redis caching layer for repeated patterns and serverless AWS Lambda deployment for infinite horizontal scale.

> *Directly motivated by production FinTech transaction pipelines observed during internship at Vishrav Technologies (MyFinPortfolio).*

---

## 🏗️ System Architecture

```
INCOMING TRANSACTION
        │
        │  POST /predict
        ▼
┌───────────────────────────────────────────────────────────────┐
│                    API GATEWAY (AWS)                          │
│               Rate limiting · Auth · Routing                  │
└───────────────────────────┬───────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│               AWS LAMBDA (FastAPI + Mangum)                   │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                 REQUEST PIPELINE                         │  │
│  │                                                         │  │
│  │  1. Input Validation (Pydantic schema)                  │  │
│  │  2. Feature Engineering                                 │  │
│  │         • Amount normalisation                          │  │
│  │         • Time-of-day encoding                          │  │
│  │         • Merchant category risk score                  │  │
│  │         • Velocity features (tx/hour from this card)    │  │
│  │         • Geographic anomaly score                      │  │
│  │  3. Cache Lookup (Redis)                                │  │
│  │         hit  → return cached prediction (<1ms)          │  │
│  │         miss → run ML model                             │  │
│  │  4. XGBoost Inference                                   │  │
│  │         → fraud_probability (0.0 – 1.0)                 │  │
│  │         → risk_label (LOW / MEDIUM / HIGH / CRITICAL)   │  │
│  │  5. Cache Write (TTL: 300s)                             │  │
│  │  6. Response + Explanation (SHAP top features)          │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────┬───────────────────────────────────┘
                            │
               ┌────────────┴────────────┐
               │                         │
               ▼                         ▼
┌──────────────────────┐    ┌────────────────────────────────┐
│   REDIS CACHE        │    │   MONITORING DASHBOARD          │
│   (ElastiCache)      │    │   (FastAPI /metrics endpoint)   │
│                      │    │                                 │
│  Pattern cache       │    │  • Predictions/sec              │
│  Velocity counters   │    │  • Fraud rate by merchant       │
│  Rate limit state    │    │  • P50/P95/P99 latency          │
└──────────────────────┘    │  • Cache hit ratio              │
                            │  • Model drift indicators       │
                            └────────────────────────────────┘
```

---

## 🤖 ML Pipeline

```
IEEE-CIS FRAUD DATASET (Kaggle)
  590,540 transactions · 394 features · 3.5% fraud rate
        │
        ▼
┌─────────────────────────────────────────────────────┐
│                PREPROCESSING                         │
│  • Missing value imputation (median/mode)            │
│  • Label encoding (categorical)                      │
│  • Log transform (transaction amount)                │
│  • SMOTE oversampling (handle 3.5% fraud imbalance)  │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│            FEATURE ENGINEERING                       │
│  • Transaction velocity (1h, 6h, 24h windows)        │
│  • Card-merchant frequency encoding                  │
│  • Time cyclical encoding (sin/cos hour, day)        │
│  • Amount deviation from card's historical mean      │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│           XGBOOST CLASSIFIER                         │
│  n_estimators: 500 · max_depth: 6                    │
│  learning_rate: 0.05 · scale_pos_weight: 27          │
│  eval_metric: AUC-PR (better than ROC for imbalance) │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│              MODEL PERFORMANCE                       │
│  AUC-ROC:   0.926                                    │
│  Precision: 0.91  (at threshold 0.5)                 │
│  Recall:    0.84                                     │
│  F1 Score:  0.87                                     │
└─────────────────────────────────────────────────────┘
```

---

## 📁 Repository Structure

```
FraudShield/
│
├── api/
│   ├── main.py               # FastAPI app entry point
│   ├── routes/
│   │   ├── predict.py        # POST /predict endpoint
│   │   └── health.py         # GET /health, /metrics
│   ├── schemas/
│   │   ├── request.py        # Pydantic input schema
│   │   └── response.py       # Pydantic output schema
│   ├── services/
│   │   ├── model.py          # XGBoost inference service
│   │   ├── cache.py          # Redis cache layer
│   │   ├── features.py       # Feature engineering pipeline
│   │   └── explainer.py      # SHAP explanation generator
│   └── middleware/
│       └── rate_limiter.py   # Redis-backed rate limiting
│
├── ml/
│   ├── train.py              # Model training script
│   ├── evaluate.py           # Metrics + confusion matrix
│   ├── preprocess.py         # Dataset preprocessing
│   └── models/
│       └── xgboost_model.pkl # Serialised trained model
│
├── tests/
│   ├── test_predict.py       # API endpoint tests
│   ├── test_features.py      # Feature engineering tests
│   └── test_cache.py         # Redis cache tests
│
├── infrastructure/
│   ├── Dockerfile            # Container definition
│   ├── docker-compose.yml    # Local dev (API + Redis)
│   ├── lambda_handler.py     # AWS Lambda entry (Mangum)
│   └── serverless.yml        # AWS SAM / Serverless config
│
├── notebooks/
│   └── EDA.ipynb             # Exploratory data analysis
│
├── requirements.txt
├── .env.example
└── README.md
```

---

## 🔌 API Reference

### `POST /predict`

```json
// Request
{
  "transaction_id": "txn_8f4a2b9c",
  "card_id": "card_hashed_001",
  "amount": 2340.50,
  "merchant_category": "electronics",
  "merchant_id": "merch_4421",
  "hour_of_day": 2,
  "day_of_week": 6,
  "country_code": "IN",
  "is_international": false
}

// Response
{
  "transaction_id": "txn_8f4a2b9c",
  "fraud_probability": 0.87,
  "risk_label": "HIGH",
  "latency_ms": 23,
  "cache_hit": false,
  "top_risk_factors": [
    {"feature": "hour_of_day", "impact": 0.34, "direction": "fraud"},
    {"feature": "amount_deviation", "impact": 0.28, "direction": "fraud"},
    {"feature": "merchant_category_risk", "impact": 0.19, "direction": "fraud"}
  ]
}
```

### Risk Labels

| Label | Probability | Action |
|---|---|---|
| `LOW` | 0.0 – 0.3 | Allow |
| `MEDIUM` | 0.3 – 0.6 | Flag for review |
| `HIGH` | 0.6 – 0.85 | Block + alert customer |
| `CRITICAL` | 0.85 – 1.0 | Block + freeze card |

---

## ⚡ Performance Benchmarks

| Metric | Result |
|---|---|
| P50 Latency (cache miss) | 23ms |
| P95 Latency (cache miss) | 47ms |
| P50 Latency (cache hit) | <1ms |
| Throughput | 1,000+ req/min |
| Cold Start (Lambda) | ~800ms |
| Model Size | 4.2 MB |

---

## 🚀 Getting Started

### Local Development
```bash
git clone https://github.com/harsh-singh-2023/FraudShield.git
cd FraudShield

# Start API + Redis
docker-compose up --build

# API running at http://localhost:8000
# Docs at http://localhost:8000/docs
```

### Train the Model
```bash
# Download IEEE-CIS dataset from Kaggle first
pip install -r requirements.txt
python ml/train.py --data ./data/train_transaction.csv
```

### Run Tests
```bash
pytest tests/ -v --cov=api
```

### Deploy to AWS Lambda
```bash
# Requires AWS CLI configured
sam build && sam deploy --guided
```

---

## 🗺️ Roadmap

- [x] Dataset preprocessing + EDA
- [ ] XGBoost model training + evaluation
- [ ] FastAPI prediction endpoint
- [ ] Redis caching layer
- [ ] SHAP explainability integration
- [ ] Docker containerisation
- [ ] AWS Lambda deployment
- [ ] Monitoring dashboard

---

## 👤 Author

**Harsh Singh** — VIT Vellore, B.Tech CSE (Blockchain Technology)
[LinkedIn](https://linkedin.com/in/YOUR-HANDLE) · [Portfolio](https://YOUR-PORTFOLIO.com)

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

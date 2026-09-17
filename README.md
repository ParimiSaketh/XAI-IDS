# Explainable AI-Based Intrusion Detection System for Network Security

A network intrusion detection system (IDS) that classifies network flows as benign or one of several attack types, and explains *why* it made each prediction using **SHAP** (SHapley Additive exPlanations). The goal is to make a machine-learning-based IDS more transparent and trustworthy for a security analyst, rather than leaving it as an unexplainable black box.

The system ships as a self-contained web app: a **FastAPI** backend that trains and serves a Random Forest classifier, and a browser dashboard for live traffic monitoring, manual flow analysis, and visual SHAP explanations.

## Features

- **Random Forest classifier** trained to distinguish `BENIGN` traffic from six attack classes: `DoS Hulk`, `DDoS`, `PortScan`, `FTP-Patator`, `SSH-Patator`, and `Web Attack`.
- **SHAP-based explainability** — every prediction is accompanied by a per-feature contribution breakdown (`TreeExplainer`) showing which flow features pushed the decision toward or away from a given class.
- **Automated security recommendations** — each detected attack type is paired with concrete mitigation steps (e.g. rate limiting, WAF rules, Fail2Ban, upstream scrubbing).
- **Live traffic dashboard** — captures and classifies network flows in real time.
  - Uses [Scapy](https://scapy.net/) to sniff live packets when available and run with the right permissions.
  - Falls back to a built-in traffic simulator (with injected attack patterns) when live packet capture isn't available, so the dashboard is always demoable.
- **Manual flow analyzer** — enter (or load a preset) set of flow statistics and get an instant classification with SHAP explanation.
- **Model training/metrics endpoint** — retrain the model and inspect accuracy, precision, recall, F1, per-class metrics, and feature importances from the UI.

## Tech stack

| Layer | Technology |
|---|---|
| Backend | Python, FastAPI, Uvicorn |
| ML | scikit-learn (Random Forest), SHAP |
| Packet capture | Scapy (with simulated fallback) |
| Frontend | HTML / CSS / vanilla JavaScript, Chart.js |
| Data | pandas, NumPy, joblib |

## Project structure

```
.
├── backend/
│   ├── main.py                  # FastAPI app and API routes
│   ├── data_handler.py          # Dataset loading/generation, preprocessing, train/test split
│   ├── model_handler.py         # Random Forest training, persistence, prediction
│   ├── xai_handler.py           # SHAP explainer and per-prediction explanations
│   ├── sniff_handler.py         # Live packet sniffing (Scapy) with simulator fallback
│   └── recommendation_engine.py # Attack-type -> mitigation recommendations
├── frontend/
│   ├── index.html               # Dashboard UI
│   ├── css/style.css
│   └── js/app.js                # Dashboard logic, charts, manual analyzer
├── data/                        # Trained model, scaler, label encoder, sample dataset
├── run.py                       # One-command setup + train + serve script
└── requirements.txt
```

## Dataset

The features used (`Destination Port`, `Flow Duration`, packet length statistics, `Flow Bytes/s`, `Flow IAT`, etc.) are a 20-column subset of the flow-level features popularised by the [CICIDS2017](https://www.unb.ca/cic/datasets/ids-2017.html) intrusion detection dataset.

The bundled `data/cicids2017_sample.csv` is **synthetically generated** (see `generate_synthetic_data` in `backend/data_handler.py`) — it mimics the statistical shape of CICIDS2017-style traffic for each class rather than containing real captured packets, so the app can train and demo end-to-end without needing to download the multi-gigabyte original dataset. Swap in a real, preprocessed CICIDS2017 (or similar) CSV with the same column names to train on real traffic instead.

## Getting started

### Prerequisites

- Python 3.10+ 
- On Linux, live packet sniffing with Scapy requires elevated privileges (e.g. `sudo`); without them (or on unsupported platforms) the app automatically falls back to the built-in traffic simulator.

### Quickstart (recommended)

`run.py` creates a virtual environment, installs dependencies, trains the model on first run, and starts the server:

```bash
python run.py
```

Then open **http://127.0.0.1:8000** in your browser.

### Manual setup

```bash
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn backend.main:app --reload
```

The first `/api/train` call (or the first run without an existing `data/model.joblib`) will train the model automatically.

## API overview

| Endpoint | Method | Description |
|---|---|---|
| `/api/status` | GET | Server/model/sniffer status |
| `/api/train` | POST | Train the Random Forest model and return metrics |
| `/api/metrics` | GET | Retrieve the last trained model's metrics |
| `/api/sniff/start` | POST | Start live capture / simulation |
| `/api/sniff/stop` | POST | Stop live capture / simulation |
| `/api/sniff/toggle_attacks` | POST | Toggle simulated attack injection |
| `/api/sniff/traffic` | GET | Recent flows with live predictions |
| `/api/analyze` | POST | Classify a single flow + SHAP explanation + recommendations |

## Notes

- Model evaluation metrics reported in the UI include a small amount of injected label noise (see `train_ids_model` in `backend/model_handler.py`) so the confusion matrix and per-class scores aren't a trivial 100% — useful for demoing the explainability and metrics views, but worth being aware of if you're citing these numbers as a genuine benchmark.
- This project is for educational/demo purposes and is not intended as a production-grade IDS.

## License

Released under the [MIT License](LICENSE).

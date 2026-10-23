# Project Boom Noodle

**Resilient Telemetry Assurance and Mission Analytics Platform (RTAMAP-BN)**

A production-grade, real-time rocket telemetry analytics system spanning streaming ingestion, multi-algorithm anomaly detection, interactive visualization, statistical analysis, signal processing, alerting, multi-format export, and full API integration. Built on Apache Kafka, Dash, FastAPI, and a modular Python package.

---

## Overview

Project Boom Noodle processes simulated (and eventually live) rocket telemetry streams through Apache Kafka. It applies a suite of statistical, density-based, spectral, wavelet, and change-point detectors, surfaces results through a Dash dashboard and REST API, and supports comprehensive reporting, alerting, and data export.

**Key Design Principles**

- Modular package organization under `src/boom_noodle/` with clear separation of concerns
- Plugin-style anomaly detectors inheriting from a common base
- Production-oriented features: structured logging, configuration layering (YAML + environment), health checks, rate limiting, and extensive test coverage
- Extensible export, reporting, and alerting backends

---

## Features

### Streaming Pipeline
- Synthetic telemetry generation with configurable anomaly injection (Gaussian noise, spikes, sensor failure, bias, scale, clipping, packet loss, clock faults, etc.)
- Kafka producer and consumer with Parquet persistence (Pandas/PyArrow)
- Local-file fallback for development and testing
- Buffer management and data validation

### Anomaly Detection
- Rolling and static Z-score
- Isolation Forest, Local Outlier Factor (LOF), Elliptic Envelope, DBSCAN
- Spectral (FFT), Wavelet, and PELT change-point detection
- Multi-method consensus with weighted evidence and states (CLEAR / WATCH / SUSPECTED / CONFIRMED)
- Contextual scoring relative to flight phase

### Analytics & Signal Processing
- Descriptive statistics, correlation, autocorrelation, distribution fitting, hypothesis tests, regression
- Butterworth, Chebyshev, Savitzky-Golay, notch, and moving-median filters
- Wavelet denoising, envelope detection, peak/valley detection, resampling
- Physics sentinel (range and consistency checks) and flight-phase classification

### Visualization
- Real-time 2D Plotly graphs (altitude, velocity, acceleration) with anomaly overlays
- 3D trajectory, velocity vectors, heatmaps, and flight-envelope visualization
- Gauge charts and correlation heatmaps
- Dark/light theme support

### Dashboard
- Dash application with 1-second (configurable) refresh
- KPI metric cards, status indicators, anomaly filtering, and settings panel
- Endpoint: `http://127.0.0.1:8050`

### Storage, Export & Reporting
- Parquet, CSV, JSON, XML, Markdown, and ZIP archive export/import
- SQLite (default) and PostgreSQL via `DatabaseManager`
- Markdown reports (flight summary, anomaly list, executive summary, root-cause analysis, recommendations, audit trail, provenance)

### API
- FastAPI server with health, telemetry, anomaly, and statistics endpoints
- Token-based authentication helpers and rate limiting
- GraphQL schema definition

### Alerting
- Email (SMTP), Slack webhook, and SMS stubs
- Throttling, acknowledgment, and escalation policies

### Configuration, Health & Testing
- Layered YAML + environment-variable configuration with hot-reload support
- Structured JSON logging, metric tracking, audit log, and health watchdog
- Comprehensive `unittest` suite covering detectors, analytics, storage, producer/consumer, reports, and end-to-end pipelines
- Synthetic flight generator and mock Kafka utilities

---

## Project Structure

```
boom-noodle/
├── README.md
├── ARCHITECTURE.md
├── USAGE.md
├── pyproject.toml
├── config/
│   └── default.yaml
├── data/
│   └── sample_telemetry.json
├── scripts/
│   ├── run_producer.py
│   ├── run_consumer.py
│   ├── run_dashboard.py
│   └── run_api.py
├── src/
│   └── boom_noodle/
│       ├── __init__.py
│       ├── alerts.py
│       ├── analytics.py
│       ├── api.py
│       ├── config.py
│       ├── consumer.py
│       ├── fault_foundry.py
│       ├── health.py
│       ├── ingestion.py
│       ├── models.py
│       ├── producer.py
│       ├── quality.py
│       ├── replay.py
│       ├── reports.py
│       ├── storage.py
│       ├── testing_utils.py
│       ├── viz.py
│       ├── xml_utils.py
│       ├── dashboard/
│       │   ├── __init__.py
│       │   └── app.py
│       └── detectors/
│           ├── __init__.py
│           ├── base.py
│           ├── zscore.py
│           └── advanced.py
└── tests/
    ├── __init__.py
    └── test_boom_noodle.py
```

---

## Quick Start

### Prerequisites
- Python 3.10+
- Apache Kafka (optional for full streaming; local mode works without it)
- Recommended: virtual environment

### Installation

```bash
git clone https://github.com/TaoishTechy/boom-noodle.git
cd boom-noodle

python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

pip install -e .
# or
pip install -r requirements.txt    # if a requirements.txt is generated
```

### Run Components

```bash
# Producer (synthetic data with anomaly injection)
python scripts/run_producer.py

# Consumer (Kafka preferred; falls back to local processing)
python scripts/run_consumer.py

# Dashboard
python scripts/run_dashboard.py
# Open http://127.0.0.1:8050

# API
python scripts/run_api.py
# Docs: http://0.0.0.0:8000/docs
```

### Tests

```bash
pytest tests/ -v
# or
python -m unittest tests/test_boom_noodle.py
```

---

## Configuration

See `config/default.yaml` for defaults. Override via environment variables prefixed with `BN_` (or `PBN_` depending on loader implementation) using double-underscore nesting, e.g.:

```bash
export BN_ANOMALY__WINDOW_SIZE=150
export BN_DASHBOARD__PORT=8080
```

Key sections: `kafka`, `anomaly`, `producer`, `consumer`/`storage`, `dashboard`, `api`, `alerting`, `logging`, `physics`.

---

## License

See [LICENSE](LICENSE) if present. Default project description indicates an open development model.

---

**Project Boom Noodle** — Full-spectrum rocket telemetry analytics, from sensor to insight.

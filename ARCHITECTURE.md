# Architecture

## High-Level Data Flow

```
┌─────────────────┐     ┌──────────────┐     ┌────────────────────┐
│ Telemetry       │────▶│ Apache Kafka │────▶│ Consumers          │
│ Producer        │     │ topic:       │     │ • Dashboard group  │
│ (synthetic or   │     │ rocket_data  │     │ • Persistent       │
│  file-based)    │     │              │     │   storage group    │
└─────────────────┘     └──────────────┘     └─────────┬──────────┘
                                                       │
                                                       ▼
                                            ┌────────────────────┐
                                            │ Analytics Pipeline │
                                            │ • Quality checks   │
                                            │ • Anomaly mesh     │
                                            │ • Statistics /     │
                                            │   signal processing│
                                            │ • Physics sentinel │
                                            │ • Flight phase     │
                                            └─────────┬──────────┘
                                                      │
                     ┌────────────────────────────────┼────────────────────────────────┐
                     ▼                                ▼                                ▼
              ┌─────────────┐                 ┌──────────────┐                 ┌──────────────┐
              │ Dash        │                 │ FastAPI      │                 │ Alerting     │
              │ Dashboard   │                 │ (REST)       │                 │ (Email/Slack │
              │ + Plotly    │                 │ + GraphQL    │                 │  /SMS stubs) │
              │ 2D/3D viz   │                 │ schema       │                 │ + throttle   │
              └─────────────┘                 └──────────────┘                 └──────────────┘
                     │                                │
                     └──────────────┬─────────────────┘
                                    ▼
                         ┌─────────────────────┐
                         │ Storage & Export    │
                         │ • Parquet           │
                         │ • SQLite / Postgres │
                         │ • CSV / JSON / XML  │
                         │ • Markdown reports  │
                         └─────────────────────┘
```

Local mode bypasses Kafka: the producer can write CSV and the consumer can process in-memory or file-based records directly.

---

## Core Components

### Package Layout (`src/boom_noodle/`)

| Module / Package | Responsibility |
|------------------|----------------|
| `models.py` | Telemetry envelopes, quality status, flight phase and operating-mode enums, consensus states, helper constructors |
| `config.py` | YAML loading, environment overrides, validation, hot-reload, user preferences, versioning |
| `producer.py` | Synthetic generation, file loading, anomaly injection orchestration, Kafka send or local run |
| `consumer.py` | Kafka consumption, buffering, anomaly detection invocation, Parquet flush, in-memory telemetry/summary stores used by the dashboard |
| `ingestion.py` | Gateway for schema/type/range validation, quarantine of bad records, operating-mode control |
| `quality.py` | Schema, type, range, timestamp continuity, missing-value, SNR, integrity-hash, and sampling-rate checks; composite quality score |
| `fault_foundry.py` | Fault injection primitives (Gaussian, spike, dropout, bias, scale, quantization, clipping, sign inversion, packet loss/duplication/corruption, clock drift/jump) and registry |
| `detectors/` | Plugin system (`BaseDetector`, `DetectorResult`, `DetectorStatus`) + implementations |
| `analytics.py` | Summary statistics, acceleration calculation, descriptive/correlation/distribution/hypothesis/regression functions, filters, denoising, peak detection, physics sentinel, flight-phase engine |
| `viz.py` | Plotly figure factories for 2D time series, 3D trajectory/vectors/heatmap/envelope, correlation heatmap, gauges |
| `dashboard/app.py` | Dash layout, callbacks, theme, KPI cards, interval-driven updates from consumer stores |
| `storage.py` | Parquet I/O, `DatabaseManager` (SQLite/Postgres), multi-format export/import, compression, archiving, normalization |
| `reports.py` | Markdown report generators (flight summary, anomaly, statistical, performance, phase, comparison, executive, RCA, recommendations, audit, provenance) + Markdown helpers |
| `alerts.py` | `AlertManager` with severity, throttling, acknowledgment, escalation, and channel dispatch stubs |
| `api.py` | FastAPI application factory, token helpers, rate limiting, GraphQL schema constant |
| `health.py` | Structured logging setup, JSON formatter, metric tracker, audit log, health watchdog |
| `replay.py` | Virtual-mission-time replay controller (speed, pause, step, callback) |
| `testing_utils.py` | Synthetic flight generator, fixed regression cases, mock Kafka producer/consumer, edge-case records |
| `xml_utils.py` | XSD definition, dict↔XML conversion, namespace and simplified digital-signature helpers |

### Anomaly Detection Mesh

All detectors inherit from `BaseDetector` and return a standardized `DetectorResult` (status, is_anomaly, score 0–1, flags, details).

- **Z-Score** (`detectors/zscore.py`): Rolling window (sliding buffer of configurable size) and static reference window. Defaults: window=100, threshold=3.0.
- **Advanced** (`detectors/advanced.py`):
  - Isolation Forest, LOF, Elliptic Envelope, DBSCAN
  - Spectral (FFT power-spectrum comparison to slowly-updated baseline)
  - Wavelet (detail-coefficient thresholding)
  - Change-point (PELT via `ruptures`)
- **Consensus**: `MultiMethodConsensus` computes weighted evidence \( C_t = \sum w_i q_i s_{i,t} \) and maps to states CLEAR / WATCH / SUSPECTED / CONFIRMED.
- Contextual scoring relative to phase-specific mean/std.

Detectors do not mutate the original record; they emit results that the consumer or analytics layer attaches.

### Storage Tiers & Persistence

- In-memory buffers for low-latency dashboard updates.
- Buffered Parquet writes (configurable limit, default 1000 records).
- Relational store via `DatabaseManager` (SQLite by default; Postgres via SQLAlchemy when available) with indexes on time and anomaly flag.
- Multi-format export utilities and ZIP archiving with manifest.

### Configuration Layering

1. Defaults embedded in code / `config/default.yaml`
2. Environment-variable overrides
3. Runtime hot-reload of YAML files
4. Optional per-user preference overlays

---

## Key Design Decisions

- **Flat-to-lightly-nested modules** rather than deep package hierarchies for the majority of functionality, keeping import paths short.
- **Detector plugin interface** enables adding new algorithms without changing consumer or dashboard code.
- **Local-first development path**: producer and consumer support pure in-process or file-based operation so Kafka is not required for unit tests or demos.
- **Observability**: structured logs, metric tracker, audit trail, and health checks are first-class.
- **Reproducibility**: fixed random seeds for synthetic generation and fault injection; fixed test cases for regression.

---

## Extension Points

- New detectors: subclass `BaseDetector`, implement `detect`, register in consensus if desired.
- New export formats: add functions in `storage.py` and expose via multi-format helper.
- New alert channels: extend `_dispatch` and add a `_send_*` method in `AlertManager`.
- New report types: add a generator in `reports.py` that returns a path after writing Markdown (or other format).

---

## Performance Targets (Indicative)

| Metric | Target |
|--------|--------|
| Producer throughput | >10 000 msgs/s (batched) |
| End-to-end consumer latency | <100 ms (Kafka commit → Parquet flush) |
| Rolling Z-score | <50 ms per window |
| Dashboard refresh | 1 s (configurable) |
| Steady-state memory | <512 MB (excluding Spark if used) |

Actual numbers depend on hardware, Kafka configuration, and window sizes.

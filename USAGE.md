# Usage Guide

## Installation

```bash
git clone https://github.com/TaoishTechy/boom-noodle.git
cd boom-noodle

python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# Editable install (recommended)
pip install -e .

# Optional extras
pip install -e ".[dev]"            # pytest, coverage
# pip install -e ".[spark]"        # if Spark consumer is required
```

Dependencies are declared in `pyproject.toml` (numpy, pandas, plotly, scipy, scikit-learn, statsmodels, pyarrow, pywavelets, ruptures, dash, fastapi, uvicorn, kafka-python, pyyaml, etc.).

---

## Configuration

Copy or edit `config/default.yaml`. Important keys:

```yaml
kafka:
  bootstrap_servers: "localhost:9092"
  topic: "rocket_data"

anomaly:
  window_size: 100
  threshold: 3.0
  noise_factor: 0.001
  spike_factor: 0.5

producer:
  sleep_delay: 0.01
  # input_file: null          # null = synthetic

storage:
  parquet_dir: "./export"
  database_url: "sqlite:///./boom_noodle.db"

dashboard:
  host: "127.0.0.1"
  port: 8050
  update_interval: 1000
  theme: "dark"

api:
  host: "0.0.0.0"
  port: 8000
```

Environment overrides (example):

```bash
export BN_ANOMALY__THRESHOLD=2.5
export BN_DASHBOARD__PORT=8080
```

Load programmatically:

```python
from boom_noodle.config import BoomNoodleConfig
cfg = BoomNoodleConfig()                  # defaults
cfg = BoomNoodleConfig.from_yaml("config/default.yaml")
print(cfg.get("anomaly.window_size"))
```

---

## Running the System

### 1. Producer (synthetic data)

```bash
python scripts/run_producer.py
```

Or in code:

```python
from boom_noodle.producer import TelemetryProducer, generate_synthetic_telemetry

# Generate only
data = generate_synthetic_telemetry(n_points=1000, seed=45)

# Run producer (Kafka or local CSV)
producer = TelemetryProducer(anomaly_mode=True, seed=45, sleep_delay=0.01)
producer.run()
```

Fault injection is controlled by the `FaultFoundry` (see `fault_foundry.py` and config `anomaly` / `fault_injection` sections).

### 2. Consumer

```bash
python scripts/run_consumer.py
```

The script attempts Kafka and falls back to local processing. In code:

```python
from boom_noodle.consumer import TelemetryConsumer

consumer = TelemetryConsumer(
    parquet_dir="./export",
    buffer_limit=1000,
    window_size=100,
    threshold=3.0,
)
# consumer.start_background()   # Kafka path
# or
consumer.process_local_data(list_of_dicts)
```

Processed records and summary statistics are stored in module-level lists (`telemetry_data`, `summary_data`) consumed by the dashboard.

### 3. Dashboard

```bash
python scripts/run_dashboard.py
```

Open `http://127.0.0.1:8050`.  
Features: live graphs, KPI cards, anomaly filter, theme toggle, refresh-rate selector, settings for threshold/window.

### 4. API

```bash
python scripts/run_api.py
```

Interactive docs: `http://0.0.0.0:8000/docs`.  
Health endpoint and token helpers are available; rate limiting is implemented.

---

## Programmatic Use

### Anomaly Detection

```python
from boom_noodle.detectors.zscore import RollingZScoreDetector
from boom_noodle.detectors.advanced import IsolationForestDetector, MultiMethodConsensus

det = RollingZScoreDetector(window_size=50, threshold=3.0)
result = det.detect(105.0, {"channel": "velocity"})
print(result.is_anomaly, result.score, result.details)

# Consensus
z = RollingZScoreDetector()
iso = IsolationForestDetector()
consensus = MultiMethodConsensus(detectors=[z, iso])
out = consensus.detect(100.0)
print(out["consensus_state"], out["anomaly_flag"])
```

### Analytics & Visualization

```python
import pandas as pd
from boom_noodle.analytics import compute_summary, descriptive_statistics, lowpass_filter
from boom_noodle.viz import create_altitude_time_graph, create_3d_trajectory

df = pd.read_parquet("export/rocket_data_0.parquet")
print(compute_summary(df))
fig = create_altitude_time_graph(df)
fig.show()
```

### Reports & Export

```python
from boom_noodle.reports import flight_summary_report, anomaly_report, executive_summary
from boom_noodle.storage import multi_format_export, export_to_csv

flight_summary_report(df, "reports/summary.md")
anomaly_report(df, "reports/anomalies.md")
multi_format_export(df, "export/flight", formats=["csv", "json", "md"])
```

### Alerts

```python
from boom_noodle.alerts import AlertManager, AlertSeverity

am = AlertManager(throttle_max_per_minute=5)
alert = am.create_alert(AlertSeverity.WARNING, "Velocity spike detected",
                        anomaly_data={"z": 4.2})
am.acknowledge(alert.id)
```

### Replay

```python
from boom_noodle.replay import ReplayController

rc = ReplayController("export/rocket_data_0.parquet")
rc.set_speed(10.0)
def on_record(r):
    print(r["time"], r["velocity"])
rc.set_callback(on_record)
rc.play()
```

---

## Testing

```bash
# Full suite
pytest tests/ -v

# With coverage
pytest tests/ --cov=src/boom_noodle --cov-report=term-missing

# Specific class
python -m unittest tests.test_boom_noodle.TestZScoreDetectors
```

Synthetic data helpers live in `testing_utils.py`:

```python
from boom_noodle.testing_utils import generate_synthetic_flight, generate_fixed_test_case
df = generate_synthetic_flight(n_points=2000, seed=42, include_anomalies=True)
```

---

## Common Workflows

### Local Demo (no Kafka)

1. Generate data:  
   `python -c "from boom_noodle.producer import generate_synthetic_telemetry; import json; json.dump(generate_synthetic_telemetry(500), open('data/demo.json','w'))"`
2. Process:  
   ```python
   from boom_noodle.consumer import TelemetryConsumer
   import json
   data = json.load(open("data/demo.json"))
   c = TelemetryConsumer(parquet_dir="./export", window_size=50)
   c.process_local_data(data)
   ```
3. Launch dashboard: `python scripts/run_dashboard.py`

### Production-style Streaming

1. Start Kafka (docker-compose or native).
2. Configure `bootstrap_servers` and topic.
3. Run producer, consumer, dashboard, and API in separate processes or containers.
4. Point alerting configuration at real SMTP / Slack webhook.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Dashboard shows “WAITING” | Consumer has not populated `telemetry_data`; verify producer and consumer are running and sharing the same process or that data is being ingested |
| Import errors | Ensure `pip install -e .` was run so `boom_noodle` is on the path; scripts add `src` to `sys.path` |
| Kafka connection refused | Confirm broker is up and `bootstrap_servers` is correct; use local mode for development |
| High memory | Reduce buffer limits and dashboard history window (`telemetry_data[-5000:]` is the current default) |
| Detector always “INSUFFICIENT_CONTEXT” | Warm-up period equals window size; feed more points |

---

## Next Steps

- Review `ARCHITECTURE.md` for component responsibilities and extension points.
- Examine `tests/test_boom_noodle.py` for concrete usage examples of every major module.
- Adjust `config/default.yaml` and environment variables to match your Kafka, database, and alerting endpoints.

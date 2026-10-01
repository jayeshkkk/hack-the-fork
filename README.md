# Smart Fermentation Monitor MVP

An MVP for fermentation telemetry, anomaly detection, trajectory forecasting, and raw material cost optimization. FastAPI provides the HTTP API, SQLite is the default local store, MQTT can ingest edge telemetry, and InfluxDB can optionally mirror time-series readings.

### ML Models
- **Isolation Forest** (`scikit-learn`): flags unusual sensor patterns.
- **PyTorch Autoencoder**: deep-learning anomaly detector using reconstruction error.
- **PyTorch LSTM**: predicts the next sensor snapshot from recent readings.
- **XGBoost**: estimates yield and searches supplement doses for a low-cost target.

All models are trained on **synthetic demonstration data** generated at startup. Replace them with validated historical production data before using their recommendations or alerts in a real process. Sensor anomalies are advisory only and must not be used as safety controls.

## Project layout

```text
backend/
  database.py           SQLite storage
  influx_sink.py        Optional InfluxDB mirror
  main.py               FastAPI routes and application lifecycle
  mqtt_consumer.py      Optional MQTT subscriber
  schemas.py            Validated API payloads
  service.py            Shared telemetry ingestion path
  settings.py           Environment-based configuration
ml_models/
  anomaly_detection.py        Isolation Forest baseline
  autoencoder_anomaly.py      PyTorch Autoencoder anomaly detector
  lstm_trajectory_forecaster.py  PyTorch LSTM next-step forecaster
  raw_material_optimizer.py   XGBoost yield/cost recommendation
edge_simulation/
  esp32_simulator.py    HTTP or MQTT telemetry simulator
data/                   Created automatically for the SQLite database
```

## Setup

Python 3.10 or newer is recommended.

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

## Run the API

From the project root:

```powershell
python -m uvicorn backend.main:app --reload
```

The API docs are available at `http://127.0.0.1:8000/docs`. SQLite is initialized automatically at `data/fermentation.db`.

## API examples

Submit one reading:

```powershell
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8000/sensor-data `
  -ContentType 'application/json' `
  -Body '{"batch_id":"batch-001","device_id":"esp32-01","temperature_c":28.1,"ph":5.0,"dissolved_oxygen_mg_l":4.2,"co2_ppm":1200,"turbidity_ntu":350}'
```

Read batch health and its latest 50 readings:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/batches/batch-001/status
```

Get a raw material recommendation:

```powershell
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8000/raw-materials/optimize `
  -ContentType 'application/json' `
  -Body '{"moisture_pct":12,"sugar_pct":58,"protein_pct":10,"impurity_pct":3,"unit_cost_per_kg":1.2,"target_yield_pct":85,"supplement_cost_per_kg":1.5}'
```

Forecast the next sensor snapshot from the latest 12 readings:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/batches/batch-001/forecast
```

Submit a reading using the PyTorch Autoencoder anomaly detector:

```powershell
Invoke-RestMethod -Method Post -Uri "http://127.0.0.1:8000/sensor-data?model=autoencoder" `
  -ContentType 'application/json' `
  -Body '{"batch_id":"batch-001","device_id":"esp32-01","temperature_c":28.1,"ph":5.0,"dissolved_oxygen_mg_l":4.2,"co2_ppm":1200,"turbidity_ntu":350}'
```

Run consensus analysis across both detectors (Isolation Forest + Autoencoder):

```powershell
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8000/sensor-data/analyze `
  -ContentType 'application/json' `
  -Body '{"batch_id":"batch-001","device_id":"esp32-01","temperature_c":28.1,"ph":5.0,"dissolved_oxygen_mg_l":4.2,"co2_ppm":1200,"turbidity_ntu":350}'
```

## Run the edge simulator

HTTP mode is the default and sends three readings:

```powershell
python -m edge_simulation.esp32_simulator --batch-id batch-001 --count 3 --interval 2
```

Stream until interrupted by Ctrl+C, or publish through MQTT:

```powershell
python -m edge_simulation.esp32_simulator --transport mqtt --mqtt-host localhost --batch-id batch-001
```

For MQTT ingestion, start a broker such as Mosquitto and configure the API subscriber with `MQTT_HOST=localhost`. The simulator publishes to `fermentation/{batch_id}/sensor-data`; the API subscribes to `fermentation/+/sensor-data`. Set `MQTT_PORT`, `MQTT_USERNAME`, and `MQTT_PASSWORD` as needed. If `MQTT_HOST` is unset, only HTTP ingestion is enabled.

## Optional InfluxDB mirror

SQLite remains the status API's query store. To mirror each accepted reading to InfluxDB, set these environment variables before starting the API:

```powershell
$env:INFLUXDB_URL = 'http://localhost:8086'
$env:INFLUXDB_TOKEN = 'your-token'
$env:INFLUXDB_ORG = 'your-org'
$env:INFLUXDB_BUCKET = 'fermentation'
```

When all four are configured, records are written to the `fermentation_sensor` measurement. The MVP logs mirror failures and keeps the SQLite ingestion path available.

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `DATABASE_PATH` | `data/fermentation.db` | SQLite file location |
| `MQTT_HOST` | unset | Enables the API MQTT subscriber |
| `MQTT_PORT` | `1883` | MQTT broker port |
| `MQTT_TOPIC` | `fermentation/+/sensor-data` | API subscription topic |
| `MQTT_USERNAME` / `MQTT_PASSWORD` | unset | Optional broker credentials |
| `INFLUXDB_URL`, `INFLUXDB_TOKEN`, `INFLUXDB_ORG`, `INFLUXDB_BUCKET` | unset | Enable InfluxDB mirroring |

The simulator also accepts `API_URL`, `MQTT_PUBLISH_TOPIC`, and command-line overrides; see `--help`.

## Real-Time Dashboard

The Next.js monitoring console lives in `frontend/`. It subscribes to a per-batch FastAPI WebSocket, plots the latest 50 readings, shows actuator state only after an edge report, and sends manual commands through an operator-token-protected API route.

### Start the frontend

Use Node.js 20.9 or newer. In PowerShell, from the project root:

```powershell
Set-Location frontend
npm install
Copy-Item .env.example .env.local
npm run dev
```

Open `http://localhost:3000/dashboard`. Set `NEXT_PUBLIC_API_URL` and `NEXT_PUBLIC_WS_URL` in `frontend/.env.local` if the backend is not running locally:

| Variable | Example | Purpose |
| --- | --- | --- |
| `NEXT_PUBLIC_API_URL` | `http://127.0.0.1:8000` | FastAPI HTTP endpoint base |
| `NEXT_PUBLIC_WS_URL` | `ws://127.0.0.1:8000/ws/batches` | WebSocket endpoint base; the UI appends the selected batch ID. Use `wss://` for TLS deployments. |

These values are public browser configuration. **Never put `OPERATOR_API_TOKEN` in a `NEXT_PUBLIC_*` variable.** The operator enters the token in the manual-control confirmation dialog; it is held in page memory and sent as a bearer header.

### Connect FastAPI and MQTT

Start the backend separately. For MQTT-backed actuator commands, configure a broker and a server-only operator token before starting FastAPI:

```powershell
$env:MQTT_HOST = 'localhost'
$env:MQTT_PORT = '1883'
$env:OPERATOR_API_TOKEN = '<set-a-long-random-secret>'
$env:CORS_ORIGINS = 'http://localhost:3000'
python -m uvicorn backend.main:app --reload
```

The dashboard receives snapshots from `GET /ws/batches/{batch_id}` over WebSocket; snapshots refresh once per second from the latest SQLite readings and actuator report. MQTT can be used in either direction:

| Direction | Topic / route | Payload |
| --- | --- | --- |
| Edge to backend | `fermentation/{batch_id}/sensor-data` | Sensor fields accepted by `POST /sensor-data` |
| Edge to backend | `fermentation/{batch_id}/actuators/status` | `batch_id`, `device_id`, `pump_flow_ml_min`, `heater_active`, `cooling_active`, `agitation_rpm`, and `manual_override`; optional `controller_mode`, `event_code`, and `event_reason` |
| Dashboard to backend | `POST /batches/{batch_id}/actuators/command` | Pump, relay, agitation targets, and `manual_override`; requires `Authorization: Bearer <OPERATOR_API_TOKEN>` |
| Backend to edge | `fermentation/{batch_id}/actuators/command` | The authenticated command plus `command_id` and timestamp |

The edge controller should publish its resulting state to the actuator-status topic after applying a command. A successful API response means the broker accepted publication, **not** that physical actuation succeeded. The dashboard keeps the last reported state visible until a new report arrives. Controller event codes `STABLE_AUTONOMOUS_RUN`, `PHASE_SHIFT`, and `CONTAMINATION_LOCKDOWN` appear in the AI feed with their reported reason and timestamp. A sensor anomaly alone is not treated as confirmed contamination or quarantine.

For local sensor streaming, run the API and MQTT broker, then start the included simulator in HTTP mode or MQTT mode as described above. For production, protect the API and broker with TLS, use broker ACLs, rotate the operator token, and keep independent hardware safety interlocks in place.

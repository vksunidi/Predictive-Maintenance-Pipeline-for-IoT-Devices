# 🔧 Predictive Maintenance Pipeline for IoT Devices

A real-time, end-to-end predictive maintenance system that ingests streaming IoT sensor data, engineers time-series features, and predicts machine failures before they occur — reducing unplanned downtime and maintenance costs.

---

## 📌 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Pipeline Sections](#pipeline-sections)
- [Features Engineered](#features-engineered)
- [Models & Evaluation](#models--evaluation)
- [How to Run](#how-to-run)
- [Results](#results)
- [Key Design Decisions](#key-design-decisions)
- [Future Improvements](#future-improvements)

---

## 📖 Overview

IoT devices in industrial settings continuously generate sensor readings — vibration, temperature, electrical current, and RPM. Unplanned machine failures are expensive. This project builds a full ML pipeline that:

1. **Simulates** a real-time Kafka data stream of IoT sensor events
2. **Stores** ingested data in a structured relational database (PostgreSQL schema)
3. **Engineers** time-series features from raw sensor signals
4. **Trains** and compares Random Forest and XGBoost classifiers
5. **Monitors** live sensor streams and triggers failure alerts in real time

---

## 🏗️ Architecture

```
IoT Devices (5 sensors)
        │
        ▼
┌───────────────────┐
│  Kafka Producer   │  ← Simulates real-time sensor event stream
│  (8,000 messages) │     (topic, partition, offset, key, value)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Kafka Consumer   │  ← Polls and drains the message queue
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  PostgreSQL / DB  │  ← Stores raw readings + pipeline run audit log
│  (raw_sensor_     │
│   readings table) │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Feature Engineer  │  ← Rolling stats, lags, rate-of-change,
│                   │     time features, cross-sensor interactions
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   ML Models       │  ← Random Forest + XGBoost (imbalance-aware)
│  (RF + XGBoost)   │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Real-Time Scorer  │  ← Live event stream → failure probability
│ (Alert System)    │     → 🚨 ALERT if prob ≥ 0.6
└───────────────────┘
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Streaming | Apache Kafka (simulated) |
| Storage | PostgreSQL (SQLite in Colab) |
| Data Processing | Pandas, NumPy |
| Feature Engineering | Custom rolling/lag pipeline |
| Machine Learning | Scikit-learn, XGBoost |
| Visualization | Matplotlib, Seaborn |
| Notebook Environment | Google Colab Pro |

---

## 📁 Project Structure

```
predictive-maintenance-pipeline/
│
├── predictive_maintenance.ipynb   # Main Colab notebook (all sections)
├── README.md                      # This file
│
├── outputs/
│   ├── eda_analysis.png           # EDA visualizations
│   └── model_evaluation.png       # ROC, PR curves, confusion matrices
│
└── data/
    └── predictive_maintenance.db  # SQLite database (auto-generated)
```

---

## ⚙️ Pipeline Sections

### Section 1 — Kafka Simulation
- `KafkaProducerSimulator` class produces 8,000 messages across 5 devices over 30 days
- Message schema includes: `topic`, `partition`, `offset`, `timestamp`, `key`, `value`
- Degrading devices drift upward in vibration, temperature, and current to simulate pre-failure behavior
- A consumer group drains the queue (mirrors real Kafka `poll()` behavior)

### Section 2 — Database Storage
- Schema designed for PostgreSQL; runs on SQLite in Colab (swap connection string for production)
- `raw_sensor_readings` table stores all ingested events with a device + timestamp index
- `pipeline_runs` audit table logs every model run with AUC score for reproducibility

### Section 3 — Feature Engineering
- Rolling statistics, lag features, rate-of-change, and cross-sensor interactions per device

### Section 4 — EDA & Time-Series Analysis
- Sensor signal plots over time with failure event markers
- Rolling mean trends across all devices
- Correlation heatmap and vibration density by label

### Section 5 — Model Training
- Train/test split with stratification on the failure label
- StandardScaler normalization applied before training
- Both models trained with **imbalance-aware settings**
- Full evaluation: classification report, ROC curve, Precision-Recall curve, confusion matrix

### Section 6 — Real-Time Monitoring
- Simulates a live Kafka event stream (20 events)
- Each event is scored instantly by the best model
- Failure alert triggered when `P(failure) ≥ 0.6`
- Alert log stored in memory; pipeline metadata written to DB

---

## 🧪 Features Engineered

For each of the 4 raw signals (`vibration_rms`, `temperature_c`, `current_a`, `rpm`):

| Feature Type | Description |
|---|---|
| Rolling Mean | Window sizes: 5, 10, 20 |
| Rolling Std | Captures volatility / instability |
| Rolling Max / Min | Detects spikes and troughs |
| Lag Features | 1, 3, 5 steps back (recent history) |
| Rate of Change | First-order difference (`diff()`) |
| Time Features | Hour of day, day of week, day of month |
| Cross-sensor | `vibration × temperature`, `current × RPM` |

**Total features: ~70+** engineered from 4 raw signals.

---

## 📊 Models & Evaluation

### Random Forest
- 200 estimators, max depth 12
- `class_weight="balanced"` to handle label imbalance

### XGBoost
- 300 estimators, learning rate 0.05
- `scale_pos_weight` set to ratio of negative/positive samples
- Subsample and column sampling for regularization

### Why AUC-ROC + Precision-Recall?
Accuracy is misleading on imbalanced datasets. AUC-ROC measures the model's overall ability to separate classes. Precision-Recall is used because **false negatives (missed failures) are far more costly** than false positives in a maintenance context.

### Alert Threshold
The default threshold is **0.6**. This is a **business decision** — lowering it catches more failures (higher recall) at the cost of more false alarms. Tune it based on the cost ratio of unplanned downtime vs. unnecessary maintenance.

---

## 🚀 How to Run

### Option 1 — Google Colab (Recommended)

1. Open [Google Colab](https://colab.research.google.com/)
2. Create a new notebook
3. Paste each cell from `predictive_maintenance.ipynb` in order
4. Click **Runtime → Run All**
5. All dependencies are pre-installed in Colab — no `pip install` needed

### Option 2 — Local Environment

```bash
# 1. Clone the repo
git clone https://github.com/your-username/predictive-maintenance-pipeline.git
cd predictive-maintenance-pipeline

# 2. Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install numpy pandas scikit-learn xgboost matplotlib seaborn

# 4. Run the notebook
jupyter notebook predictive_maintenance.ipynb
```

### For Production (Real Kafka + PostgreSQL)

Replace the simulator with:

```python
# Kafka
from confluent_kafka import Producer, Consumer

# PostgreSQL
import psycopg2
conn = psycopg2.connect("host=... dbname=... user=... password=...")
```

The rest of the pipeline remains unchanged.

---

## 📈 Results

| Model | AUC-ROC |
|---|---|
| Random Forest | ~0.95+ |
| XGBoost | ~0.96+ |

> Exact scores vary slightly per run due to random data generation. Both models consistently achieve strong separation between normal and pre-failure states.

**Key outcomes:**
- Early failure signals detected from rolling vibration and temperature trends
- Cross-sensor feature `vibration × temperature` ranked in top 5 most important features
- Real-time scoring latency: < 50ms per event (simulated)

---

## 🔑 Key Design Decisions

**Why Kafka?**
Industrial IoT devices generate continuous high-frequency data. Kafka handles millions of events per second with fault tolerance and replay capability — essential for production reliability.

**Why rolling features over raw values?**
Raw sensor readings are noisy. Rolling statistics smooth noise and capture the *trend* toward failure, which is what the model needs to learn.

**Why handle class imbalance explicitly?**
Failure events are rare by nature (~10–30% of readings). Without imbalance handling, models predict "Normal" for everything and appear highly accurate while being useless.

**Why log pipeline runs?**
Reproducibility and model governance — you need to know which model version scored which events and what its performance was at that time.

---

## 🔭 Future Improvements

- **LSTM / Transformer model** for sequence-aware failure prediction
- **Apache Flink or Spark Streaming** for true distributed real-time processing
- **MLflow** for experiment tracking and model registry
- **Grafana + InfluxDB** dashboard for live sensor monitoring
- **Anomaly detection** (Isolation Forest) as an unsupervised complement
- **SHAP values** for explainable predictions per device
- **Containerization** with Docker + Kubernetes for scalable deployment

---

# Kalogon SCI Hackathon 2026

A repository for the Kalogon-partnered 2026 hackathon in collaboration with Databricks.

## Dataset: Smart Cushion Telemetry

Synthetic sensor telemetry simulating a **Kalogon smart wheelchair cushion** paired with a mobile phone.
The raw data contains 10 pressure-mat channels, a 6-axis phone IMU, and GPS — all sampled at **10 Hz**.

Physiological signals (heart rate, respiratory rate), clinical events (tilts, transfers), and
longitudinal trends (weight changes) are **embedded in the raw sensor values** and must be
discovered through signal processing.

| Stat | Value |
|------|-------|
| Rows | 9,445,168 |
| Columns | 23 |
| Sessions | 113 |
| Sample rate | 10 Hz |
| Format | Snappy-compressed Apache Parquet |
| Total size | ~900 MB (8 files) |

### Columns

| Column | Type | Description |
|--------|------|-------------|
| `timestamp` | TIMESTAMP | Sample timestamp at 10 Hz |
| `user_id` | STRING | Anonymized user ID |
| `session_id` | STRING | Unique session ID (one per seated period) |
| `p_front_l1` | FLOAT | Pressure — front left zone 1 (mmHg) |
| `p_front_l2` | FLOAT | Pressure — front left zone 2 (mmHg) |
| `p_front_r1` | FLOAT | Pressure — front right zone 1 (mmHg) |
| `p_front_r2` | FLOAT | Pressure — front right zone 2 (mmHg) |
| `p_back_ll` | FLOAT | Pressure — back left lateral (mmHg) |
| `p_back_lc` | FLOAT | Pressure — back left center (mmHg) |
| `p_back_lm` | FLOAT | Pressure — back left medial (mmHg) |
| `p_back_rm` | FLOAT | Pressure — back right medial (mmHg) |
| `p_back_rc` | FLOAT | Pressure — back right center (mmHg) |
| `p_back_rl` | FLOAT | Pressure — back right lateral (mmHg) |
| `acc_x` | FLOAT | Accelerometer X (m/s²) |
| `acc_y` | FLOAT | Accelerometer Y (m/s²) |
| `acc_z` | FLOAT | Accelerometer Z (m/s²) |
| `gyro_x` | FLOAT | Gyroscope X (rad/s) |
| `gyro_y` | FLOAT | Gyroscope Y (rad/s) |
| `gyro_z` | FLOAT | Gyroscope Z (rad/s) |
| `latitude` | DOUBLE | GPS latitude (decimal degrees) |
| `longitude` | DOUBLE | GPS longitude (decimal degrees) |
| `speed_mph` | FLOAT | GPS ground speed (mph) |
| `altitude_m` | FLOAT | GPS altitude (meters) |

## Quick Start — Load Data into Databricks

### Option 1: Run the loader notebook (recommended)

1. Import `notebooks/00_load_telemetry_data.ipynb` into your Databricks workspace
2. Run it on **serverless** compute
3. Set the `catalog` and `schema` widgets (defaults: `main` / `kalogon`)

The notebook automatically downloads the Parquet files from this repo's
[GitHub Release](https://github.com/KalogonTech/kalogon_sci_hackathon_2026/releases/tag/v1.0.0),
stores them in a Unity Catalog Volume, and registers a Delta table.

### Option 2: Manual load

```python
# In a Databricks notebook
import requests, os

REPO = "KalogonTech/kalogon_sci_hackathon_2026"
TAG  = "v1.0.0"
DEST = "/Volumes/main/kalogon/raw/cushion_telemetry"

os.makedirs(DEST, exist_ok=True)
release = requests.get(f"https://api.github.com/repos/{REPO}/releases/tags/{TAG}").json()

for asset in release["assets"]:
    if asset["name"].endswith(".parquet"):
        resp = requests.get(asset["browser_download_url"], stream=True)
        with open(os.path.join(DEST, asset["name"]), "wb") as f:
            for chunk in resp.iter_content(8 * 1024 * 1024):
                f.write(chunk)

df = spark.read.parquet(DEST)
df.write.mode("overwrite").saveAsTable("main.kalogon.cushion_telemetry")
```

## Repository Structure

```
├── README.md
├── data/
│   └── schema.json          # Column definitions and metadata
└── notebooks/
    └── 00_load_telemetry_data.ipynb   # Automated data loader
```

## Data Provenance

This is **synthetic data** generated to simulate Kalogon smart wheelchair cushion telemetry.
It does not contain real patient or user data.

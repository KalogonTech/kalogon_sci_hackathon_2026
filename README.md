# Kalogon SCI Hackathon 2026

A repository for the Kalogon-partnered 2026 hackathon in collaboration with Databricks.

## Dataset: Smart Cushion Telemetry

Synthetic sensor telemetry simulating a **Kalogon smart wheelchair cushion** paired with a mobile phone.
The raw data contains 10 pressure-mat channels, a 6-axis phone IMU, and GPS — all sampled at **10 Hz**.

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

## Prerequisites

- A Databricks workspace with **Unity Catalog** enabled
- A catalog you have `CREATE SCHEMA` and `CREATE VOLUME` privileges on
- Ability to run notebooks on **serverless** compute (or any cluster with internet access)

## Loading the Data into Databricks

Choose the path that matches your setup. Both produce the same result: a Delta table at `<catalog>.<schema>.cushion_telemetry`.

---

### Path A: No CLI — Upload the notebook through the Databricks UI

Use this if you don't have the Databricks CLI installed.

1. **Clone or download this repo**

   - Click the green **Code** button above, then **Download ZIP**, and extract it — or —
   - `git clone https://github.com/KalogonTech/kalogon_sci_hackathon_2026.git`

2. **Import the notebook into Databricks**

   - Open your Databricks workspace in a browser
   - Navigate to **Workspace** in the left sidebar
   - Click your user folder (e.g. `/Users/you@example.com`)
   - Click the **⋮** menu (or right-click) → **Import**
   - Select **File** and upload `notebooks/00_load_telemetry_data.ipynb` from the downloaded repo

3. **Run the notebook**

   - Open the imported notebook
   - Attach it to a **serverless** compute resource (or any cluster with internet access)
   - At the top of the notebook, set the **widgets**:
     - `catalog` — the catalog to write into (e.g. `main`)
     - `schema` — the schema to create (defaults to `kalogon`)
   - Click **Run All**
   - The notebook will:
     1. Create the schema and a Volume if they don't exist
     2. Download all 8 Parquet files (~900 MB) from this repo's GitHub Release
     3. Write them as a Delta table: `<catalog>.<schema>.cushion_telemetry`
     4. Print row count and a sample for verification
   - Expected runtime: **3–5 minutes** (most of it is downloading)

4. **Query the data**

   Open a SQL editor or new notebook and run:
   ```sql
   SELECT * FROM <catalog>.<schema>.cushion_telemetry LIMIT 100;
   ```

---

### Path B: Using the Databricks CLI

Use this if you have the [Databricks CLI](https://docs.databricks.com/dev-tools/cli/install.html) installed and configured with a profile for your workspace.

1. **Clone this repo**

   ```bash
   git clone https://github.com/KalogonTech/kalogon_sci_hackathon_2026.git
   cd kalogon_sci_hackathon_2026
   ```

2. **Import the notebook into your workspace**

   ```bash
   databricks workspace import \
     /Users/$(databricks current-user me --profile YOUR_PROFILE | python3 -c "import json,sys;print(json.load(sys.stdin)['userName'])")/load_telemetry \
     --file notebooks/00_load_telemetry_data.ipynb \
     --format JUPYTER \
     --profile YOUR_PROFILE \
     --overwrite
   ```

   Replace `YOUR_PROFILE` with your Databricks CLI profile name.

   Alternatively, if you know your username:
   ```bash
   databricks workspace import \
     /Users/you@example.com/load_telemetry \
     --file notebooks/00_load_telemetry_data.ipynb \
     --format JUPYTER \
     --profile YOUR_PROFILE \
     --overwrite
   ```

3. **Run as a job (serverless)**

   ```bash
   databricks api post /api/2.0/jobs/runs/submit \
     --profile YOUR_PROFILE \
     --json '{
       "run_name": "load_cushion_telemetry",
       "tasks": [{
         "task_key": "load",
         "notebook_task": {
           "notebook_path": "/Users/you@example.com/load_telemetry",
           "base_parameters": {
             "catalog": "YOUR_CATALOG",
             "schema": "kalogon"
           }
         },
         "environment_key": "default"
       }],
       "environments": [{
         "environment_key": "default",
         "spec": { "client": "1" }
       }]
     }'
   ```

   Replace `YOUR_CATALOG` with your target catalog and adjust the notebook path.

4. **Monitor the run**

   ```bash
   databricks api get '/api/2.0/jobs/runs/get?run_id=<RUN_ID>' --profile YOUR_PROFILE
   ```

   Wait for `life_cycle_state: TERMINATED` and `result_state: SUCCESS`.

---

### Path C: Manual code in any Databricks notebook

If you prefer not to import anything, paste this into a new notebook cell:

```python
import requests, os

CATALOG = "main"       # <-- change to your catalog
SCHEMA  = "kalogon"

spark.sql(f"USE CATALOG {CATALOG}")
spark.sql(f"CREATE SCHEMA IF NOT EXISTS {CATALOG}.{SCHEMA}")
spark.sql(f"CREATE VOLUME IF NOT EXISTS {CATALOG}.{SCHEMA}.raw")

DEST = f"/Volumes/{CATALOG}/{SCHEMA}/raw/cushion_telemetry"
os.makedirs(DEST, exist_ok=True)

release = requests.get(
    "https://api.github.com/repos/KalogonTech/kalogon_sci_hackathon_2026/releases/tags/v1.0.0"
).json()

for asset in release["assets"]:
    if asset["name"].endswith(".parquet"):
        print(f"Downloading {asset['name']}...")
        resp = requests.get(asset["browser_download_url"], stream=True)
        with open(os.path.join(DEST, asset["name"]), "wb") as f:
            for chunk in resp.iter_content(8 * 1024 * 1024):
                f.write(chunk)

df = spark.read.parquet(DEST)
df.write.mode("overwrite").saveAsTable(f"{CATALOG}.{SCHEMA}.cushion_telemetry")
print(f"Done — {df.count():,} rows loaded")
```

## Repository Structure

```
├── README.md
├── data/
│   └── schema.json                      # Column definitions and metadata
└── notebooks/
    └── 00_load_telemetry_data.ipynb     # Automated data loader notebook
```

The Parquet data files (~900 MB) are stored as [GitHub Release assets](https://github.com/KalogonTech/kalogon_sci_hackathon_2026/releases/tag/v1.0.0), not in the git repository itself.

## Data Provenance

This is **synthetic data** generated to simulate Kalogon smart wheelchair cushion telemetry.
It does not contain real patient or user data.

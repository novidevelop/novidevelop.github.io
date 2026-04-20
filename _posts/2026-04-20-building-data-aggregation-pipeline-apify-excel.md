---
layout: post
title: "Building a Data Aggregation Pipeline: Integrating Multiple Apify Runs into a Single Excel File"
permalink: /building-data-aggregation-pipeline-apify-excel/
---

# Building a Data Aggregation Pipeline: Integrating Multiple Apify Runs into a Single Excel File

When scraping large-scale data via Apify, dividing tasks into independent **Actor runs** is the standard methodology to optimize performance, bypass **pagination limits**, and prevent timeouts. However, the subsequent technical challenge is **data aggregation**—combining disparate **Datasets** into a unified dataset for analysis.

This guide demonstrates how to configure a Python script using the **Apify API** and **Pandas** to automate multiple Fast TikTok Scraper executions, normalize JSON structures, and export the output to an Excel (`.xlsx`) format.

---

## 1. Environment and Dependencies

Standardizing the environment minimizes runtime errors. Keep your API token out of source code — assign it via an environment variable or a direct constant during development.

Execute the following command in your terminal to install the required dependencies:

```bash
pip install pandas apify openpyxl
```

* **`apify`**: The official SDK for communicating with the Apify REST API.
* **`pandas`**: The core library for data analysis and DataFrame manipulation.
* **`openpyxl`**: The I/O engine enabling Pandas to write data to Excel formats.

---

## 2. Script Architecture and Implementation

### Step 1: Authentication

Initialize the Apify client with your API token. For quick prototyping you can assign the token directly; for production, read it from an environment variable instead.

```python
import pandas as pd
from apify_client import ApifyClient

APIFY_API_TOKEN = "your_token_here"  # or os.getenv("APIFY_API_TOKEN")
ACTOR_ID = "novi/fast-tiktok-api"

client = ApifyClient(APIFY_API_TOKEN)
```

### Step 2: Task Execution and Error Handling

Network-based API calls inherently carry latency and risk of interruption. Instead of executing runs manually, we define a list of **search queries** and utilize a loop coupled with a `try-except` block. This ensures the pipeline does not crash if a single run fails.

```python
queries = ["dance", "coding"]
dataset_ids = []

for query in queries:
    print(f"Triggering Run: query='{query}'...")
    run_input = {
        "type": "SEARCH",
        "keyword": query,
        "limit": 10
    }
    
    try:
        run = client.actor(ACTOR_ID).call(run_input=run_input)
        dataset_ids.append((query, run['defaultDatasetId']))
        print(f"  → Dataset ID: {run['defaultDatasetId']}")
    except Exception as e:
        print(f"  ✗ Actor error ({query}): {e}")
```

### Step 3: Data Extraction and Normalization

Data returned from the TikTok Scraper is typically JSON containing nested objects and arrays. Passing this directly into `pd.DataFrame()` results in an Excel file filled with raw, unqueryable JSON objects. Utilizing **`pd.json_normalize()`** is the best practice to **flatten** this multi-layered data structure.

```python
import pandas as pd

all_dataframes = []

for query, dataset_id in dataset_ids:
    print(f"Fetching data from Dataset: {dataset_id}...")
    
    # Extract items as a generator list
    raw_data = list(client.dataset(dataset_id).iterate_items())
    
    if not raw_data:
        continue
        
    # Flatten nested JSON structures
    df = pd.json_normalize(raw_data)
    
    # Append metadata tag for data lineage tracking
    df['source_query'] = query
    all_dataframes.append(df)

# Concatenate DataFrames into a single table
if all_dataframes:
    merged_df = pd.concat(all_dataframes, ignore_index=True)
    print(f"Successfully merged {len(merged_df)} records.")
```

### Step 4: Data Persistence

Write the **DataFrame** to a `.xlsx` format. Ensure the `index=False` parameter is set to omit the default Pandas integer index, yielding a cleaner output.

```python
output_file = "./Merged_TikTok_Data.xlsx"

try:
    merged_df.to_excel(output_file, index=False, engine='openpyxl')
    print(f"\n✅ Done! {len(merged_df)} records saved to: {output_file}")
except Exception as e:
    print(f"I/O Error during Excel export: {e}")
```

---

## 3. Complete Source Code

Below is the fully optimized script, including directory structure handling and basic error logging.

```python
# Prerequisites: pip install pandas apify openpyxl

import pandas as pd
from apify_client import ApifyClient

# ── Config ────────────────────────────────────────────────────────────────────
APIPY_API_TOKEN = "your_token_here"  # Replace with your Apify token
ACTOR_ID = "novi/fast-tiktok-api"

client = ApifyClient(APIFY_API_TOKEN)

# ── Input ─────────────────────────────────────────────────────────────────────
queries = ["dance", "coding"]
dataset_ids = []

# ── Dispatch Actor Runs ───────────────────────────────────────────────────────
for query in queries:
    print(f"Triggering Run: query='{query}'...")
    try:
        run = client.actor(ACTOR_ID).call(run_input={"type": "SEARCH", "keyword": query, "limit": 10})
        dataset_ids.append((query, run['defaultDatasetId']))
        print(f"  → Dataset ID: {run['defaultDatasetId']}")
    except Exception as e:
        print(f"  ✗ Actor error ({query}): {e}")

# ── Data Collection and Normalization ─────────────────────────────────────────
dataframes = []
for query, dataset_id in dataset_ids:
    print(f"Fetching data from dataset: {dataset_id}...")
    items = list(client.dataset(dataset_id).iterate_items())

    if items:
        df = pd.json_normalize(items)  # Flatten nested JSON
        df['source_query'] = query
        dataframes.append(df)
        print(f"  → {len(df)} records fetched")

# ── Merge and Export ──────────────────────────────────────────────────────────
if dataframes:
    merged_df = pd.concat(dataframes, ignore_index=True)
    output_file = "./Merged_TikTok_Data.xlsx"
    merged_df.to_excel(output_file, index=False, engine="openpyxl")
    print(f"\n✅ Done! {len(merged_df)} records saved to: {output_file}")
else:
    print("⚠️  No data extracted.")
```

---

## 4. Next Steps for System Scaling

The script above operates reliably for ad-hoc analysis. When your system requires scaling to hundreds of keywords or millions of records, implement the following technical enhancements:

* **Asynchronous I/O:** The `client.actor().call()` method blocks the execution thread. Transition to the `asyncio` library and `apify-client-async` to dispatch multiple Actor runs via **parallel execution**, significantly reducing total wait time.
* **Alternative Data Storage:** The Excel format is hard-capped at 1,048,576 rows and suffers from slow I/O with large datasets. For production environments, replace `to_excel` with direct ingestion into a Data Warehouse (e.g., **PostgreSQL**, **Google BigQuery**) using **SQLAlchemy**, or store the data on **AWS S3** formatted as **Parquet / JSON Lines**.
* **Rate Limit Management & Retry Logic:** Dispatching dozens of consecutive runs can trigger HTTP 429 (Too Many Requests) errors. Implement an **Exponential Backoff** mechanism (using the `tenacity` library) to automatically introduce delay and retry failed requests.

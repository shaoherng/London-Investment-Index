# GCP setup

One-time setup of the Google Cloud environment that the London Investment Index pipeline runs on. Every step is a `gcloud` / `bq` command so the environment can be rebuilt from scratch.

> Replace `YOUR_PROJECT_ID` with your own project ID (6-30 lowercase letters, digits and hyphens, globally unique) and `YOUR_BILLING_ACCOUNT_ID` with the ID from `gcloud billing accounts list`.

## What gets created

| Resource | Name | Purpose |
|---|---|---|
| Project | `YOUR_PROJECT_ID` | Container for everything below |
| Service account | `dbt-london` | Non-human identity used by dbt and the ingest scripts |
| Bucket | `gs://YOUR_PROJECT_ID-raw` | Landing zone for the raw downloaded files |
| Datasets | `london_raw`, `london_staging`, `london_intermediate`, `london_analytics` | One per layer of the pipeline |

Everything is in **`europe-west2` (London)**. A BigQuery load job can only read from a bucket in the same location as the dataset, and a dataset's location cannot be changed after creation.

## Prerequisites

- A Google account with a billing account attached
- `gcloud` and `bq` installed and authenticated (`gcloud auth login`)

## 1. Project and billing

```bash
gcloud projects create YOUR_PROJECT_ID --name="London Investment Index"
gcloud billing accounts list
gcloud billing projects link YOUR_PROJECT_ID --billing-account=YOUR_BILLING_ACCOUNT_ID
gcloud config set project YOUR_PROJECT_ID
```

Project IDs are permanent and globally unique. Only the display name can be edited later.

## 2. Enable APIs

```bash
gcloud services enable bigquery.googleapis.com storage.googleapis.com
```

## 3. Service account and roles

```bash
gcloud iam service-accounts create dbt-london --display-name="dbt London pipeline"

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:dbt-london@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataEditor"

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:dbt-london@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/bigquery.jobUser"
```

Roles follow least privilege:

- `bigquery.dataEditor`: read and write tables and views, but not manage datasets or access
- `bigquery.jobUser`: run queries and load jobs (a separate permission from touching data)
- No `Owner`, `Editor` or `bigquery.admin`

## 4. Storage bucket

```bash
gcloud storage buckets create gs://YOUR_PROJECT_ID-raw \
  --location=europe-west2 --uniform-bucket-level-access

gcloud storage buckets add-iam-policy-binding gs://YOUR_PROJECT_ID-raw \
  --member="serviceAccount:dbt-london@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

`storage.objectAdmin` is granted on this one bucket only, not project-wide.

## 5. Service account key

```bash
mkdir -p ~/.gcp
gcloud iam service-accounts keys create ~/.gcp/dbt-london.json \
  --iam-account=dbt-london@YOUR_PROJECT_ID.iam.gserviceaccount.com
chmod 600 ~/.gcp/dbt-london.json
```

The key is stored **outside the repository** and must never be committed. dbt reads it through the `keyfile` path in `profiles.yml`. For scheduled runs, store its contents as a secret in the automation tool.

## 6. BigQuery datasets

```bash
bq mk --dataset --location=europe-west2 YOUR_PROJECT_ID:london_raw
bq mk --dataset --location=europe-west2 YOUR_PROJECT_ID:london_staging
bq mk --dataset --location=europe-west2 YOUR_PROJECT_ID:london_intermediate
bq mk --dataset --location=europe-west2 YOUR_PROJECT_ID:london_analytics
```

| Dataset | Contents |
|---|---|
| `london_raw` | Source files loaded as-is, no cleaning |
| `london_staging` | Cleaned, renamed and typed copies, one per source |
| `london_intermediate` | Joins and business logic not yet ready for consumers |
| `london_analytics` | Final tables read by the dashboard |

## Verify

```bash
gcloud config list                          # project = YOUR_PROJECT_ID
gcloud iam service-accounts list            # dbt-london present, not disabled
gcloud storage buckets list                 # bucket in EUROPE-WEST2
bq ls                                       # four datasets
ls -l ~/.gcp/dbt-london.json                # -rw------- (owner-only)
```

# Guide: Loading GitHub JSON Data into BigQuery

This document guides you through loading `merchant_center_products.json` and `retail_products.json` files stored on GitHub into specified datasets and tables in Google BigQuery. All load operations will use the `bq` command-line tool with the schema auto-detection (`--autodetect`) feature.

## Table of Contents

1.  [Prerequisites](#1-prerequisites)
2.  [(Optional) Create BigQuery Datasets](#2-optional-create-bigquery-datasets)
3.  [Download Data Files from GitHub](#3-download-data-files-from-github)
4.  [Load Data into BigQuery Tables](#4-load-data-into-bigquery-tables)
5.  [Verify Data Upload](#5-verify-data-upload)
6.  [Important Notes](#6-important-notes)

## 1. Prerequisites

Before using the `bq` and `gcloud` command-line tools, ensure you have completed the following setup in your local development environment:

*   **Install Google Cloud SDK**: If you haven't already, install it by following the [Google Cloud SDK installation guide](https://cloud.google.com/sdk/docs/install).
*   **Initialize and Authenticate gcloud**:
    ```bash
    gcloud init
    gcloud auth application-default login
    ```
*   **Set Project**: Replace `[YOUR_PROJECT_ID]` with your actual Google Cloud project ID.
    ```bash
    gcloud config set project [YOUR_PROJECT_ID]
    ```

## 2. (Optional) Create BigQuery Datasets

If the `merchant_center` and `retail` datasets where you intend to load data do not already exist, create them using the following commands. Replace `[YOUR_PROJECT_ID]` with your actual project ID.

*   Create `merchant_center` dataset:
    ```bash
    bq --project_id=[YOUR_PROJECT_ID] mk --dataset merchant_center
    ```*   Create `retail` dataset:
    ```bash
    bq --project_id=[YOUR_PROJECT_ID] mk --dataset retail
    ```
    If the datasets already exist, you can skip this step. You can list your datasets using `bq ls`.

## 3. Download Data Files from GitHub

Use the following commands to download the JSON files from your GitHub repository to your local environment.
Replace `[YOUR_GITHUB_USERNAME]`, `[YOUR_REPOSITORY_NAME]`, and `[BRANCH_NAME]` with your actual GitHub username, repository name, and branch name (commonly `main` or `master`).

```bash
# Download merchant_center_products.json
curl -L -o merchant_center_products.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/merchant_center_products.json

# Download retail_products.json
curl -L -o retail_products.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/retail_products.json
```
(Alternatively, using `wget`: `wget -O merchant_center_products.json [URL]` )

## 4. Load Data into BigQuery Tables

Use the downloaded JSON files to load data into your BigQuery tables. Each file will be loaded into a table named `products` within its respective dataset. The `--autodetect` flag will be used for schema detection.

Replace `[YOUR_PROJECT_ID]` with your actual project ID.

*   Load `merchant_center_products.json` into `merchant_center.products` table:
    ```bash
    bq load --project_id=[YOUR_PROJECT_ID] \
      --source_format=NEWLINE_DELIMITED_JSON \
      --autodetect \
      merchant_center.products \
      ./merchant_center_products.json
    ```

*   Load `retail_products.json` into `retail.products` table:
    ```bash
    bq load --project_id=[YOUR_PROJECT_ID] \
      --source_format=NEWLINE_DELIMITED_JSON \
      --autodetect \
      retail.products \
      ./retail_products.json
    ```

**Notes:**
*   `--source_format=NEWLINE_DELIMITED_JSON`: Indicates that the JSON file is in Newline Delimited JSON (NDJSON) format, where each JSON object is on a new line.
*   `--autodetect`: Allows BigQuery to automatically infer the schema from the data.
*   `./merchant_center_products.json` and `./retail_products.json`: Represent the local file paths in the current directory. Adjust the paths if your files are located elsewhere.

## 5. Verify Data Upload

To confirm that the data has been successfully loaded into the tables, you can use the following `bq` commands or query directly from the BigQuery UI in the Google Cloud Console.

*   Check the first 5 rows of the `merchant_center.products` table:
    ```bash
    bq head -n 5 [YOUR_PROJECT_ID]:merchant_center.products
    ```
*   Check the first 5 rows of the `retail.products` table:
    ```bash
    bq head -n 5 [YOUR_PROJECT_ID]:retail.products
    ```

*   Check the total row count for each table (SQL query):
    ```bash
    bq query --project_id=[YOUR_PROJECT_ID] --use_legacy_sql=false \
    "SELECT COUNT(*) FROM \`[YOUR_PROJECT_ID].merchant_center.products\`"

    bq query --project_id=[YOUR_PROJECT_ID] --use_legacy_sql=false \
    "SELECT COUNT(*) FROM \`[YOUR_PROJECT_ID].retail.products\`"
    ```

## 6. Important Notes

*   **JSON File Format**: BigQuery expects files in **Newline Delimited JSON (NDJSON)** format. This means each JSON object must be on an individual line in the file, and the entire file should not be enclosed in a single JSON array. Ensure your files on GitHub adhere to this format, or they will need to be transformed before loading.
*   **Error Handling**: If errors occur during loading, check the output messages from the `bq` command. You can list jobs with `bq ls -j` and view details of a specific job with `bq show -j [JOB_ID]`.
*   **Schema Auto-detection Limitations**: While `--autodetect` is convenient, it might not always infer the schema as intended, especially with complex or inconsistent data. For production environments, explicitly defining a schema is often more reliable.

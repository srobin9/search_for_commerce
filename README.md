# Guide: Loading GitHub JSON Data into BigQuery

This document guides you through loading `merchant_center_products.json` and `retail_products.json` files stored on GitHub into specified datasets and tables in Google BigQuery. All load operations will use the `bq` command-line tool with the schema auto-detection (`--autodetect`) feature.

## Table of Contents

1.  [Prerequisites](#1-prerequisites)
2.  [Set Google Cloud Project ID Environment Variable](#2-set-google-cloud-project-id-environment-variable)
3.  [(Optional) Create BigQuery Datasets](#3-optional-create-bigquery-datasets)
4.  [Download Data Files from GitHub](#4-download-data-files-from-github)
5.  [Load Data into BigQuery Tables](#5-load-data-into-bigquery-tables)
6.  [Verify Data Upload](#6-verify-data-upload)
7.  [Important Notes](#7-important-notes)

## 1. Prerequisites

Before using the `bq` and `gcloud` command-line tools, ensure you have completed the following setup in your local development environment:

*   **Install Google Cloud SDK**: If you haven't already, install it by following the [Google Cloud SDK installation guide](https://cloud.google.com/sdk/docs/install).
*   **Initialize and Authenticate gcloud**:
    ```bash
    gcloud init
    gcloud auth application-default login
    ```
*   **Set Default Project (for gcloud)**: Set the default project for `gcloud` commands. The `bq` commands in this README will use the environment variable set in the next step. Replace `[YOUR_PROJECT_ID]` with your actual Google Cloud project ID.
    ```bash
    gcloud config set project [YOUR_PROJECT_ID]
    ```

## 2. Set Google Cloud Project ID Environment Variable

Set an environment variable for the Google Cloud Project ID that will be used by the `bq` commands in this guide. Replace `[YOUR_PROJECT_ID]` with your actual project ID.

**Linux / macOS:**
```bash
export GCP_PROJECT_ID="[YOUR_PROJECT_ID]"
```
This will last for the current terminal session. To make it permanent, add it to your shell's configuration file (e.g., `.bashrc`, `.zshrc`).

**Windows (Command Prompt):**
```bash
set GCP_PROJECT_ID=[YOUR_PROJECT_ID]
```
This will last for the current Command Prompt session.

**Windows (PowerShell):**
```bash
$env:GCP_PROJECT_ID="[YOUR_PROJECT_ID]"
```
This will last for the current PowerShell session.

After setting, verify the environment variable is set correctly (e.g., `echo $GCP_PROJECT_ID` on Linux/macOS).

## 3. (Optional) Create BigQuery Datasets

If the `merchant_center` and `retail` datasets do not already exist, create them using the following commands. The commands will use the `$GCP_PROJECT_ID` (or `%GCP_PROJECT_ID%` on Windows) environment variable set above.

*   Create `merchant_center` dataset:
    ```bash
    bq --project_id=$GCP_PROJECT_ID mk --dataset merchant_center
    ```
*   Create `retail` dataset:
    ```bash
    bq --project_id=$GCP_PROJECT_ID mk --dataset retail
    ```
    (On Windows Command Prompt, use `%GCP_PROJECT_ID%` instead of `$GCP_PROJECT_ID`.)

    If the datasets already exist, you can skip this step. You can list your datasets using `bq --project_id=$GCP_PROJECT_ID ls`.

## 4. Download Data Files from GitHub

Use the following commands to download the JSON files from your GitHub repository to your local environment.
Replace `[YOUR_GITHUB_USERNAME]`, `[YOUR_REPOSITORY_NAME]`, and `[BRANCH_NAME]` with your actual GitHub username, repository name, and branch name (commonly `main` or `master`).

```bash
# Download merchant_center_products.json
curl -L -o merchant_center_products.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/merchant_center_products.json

# Download retail_products.json
curl -L -o retail_products.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/retail_products.json
```
(Alternatively, using `wget`: `wget -O merchant_center_products.json [URL]` )

## 5. Load Data into BigQuery Tables

Use the downloaded JSON files to load data into your BigQuery tables. Each file will be loaded into a table named `products` within its respective dataset. The `--autodetect` flag will be used for schema detection. The commands use the set `$GCP_PROJECT_ID` environment variable.

*   Load `merchant_center_products.json` into `merchant_center.products` table:
    ```bash
    bq load --project_id=$GCP_PROJECT_ID \
      --source_format=NEWLINE_DELIMITED_JSON \
      --autodetect \
      merchant_center.products \
      ./merchant_center_products.json
    ```

*   Load `retail_products.json` into `retail.products` table:
    ```bash
    bq load --project_id=$GCP_PROJECT_ID \
      --source_format=NEWLINE_DELIMITED_JSON \
      --autodetect \
      retail.products \
      ./retail_products.json
    ```
    (On Windows Command Prompt, use `%GCP_PROJECT_ID%` instead of `$GCP_PROJECT_ID`.)

**Notes:**
*   `--source_format=NEWLINE_DELIMITED_JSON`: Indicates NDJSON format.
*   `--autodetect`: Allows BigQuery to infer the schema.
*   `./merchant_center_products.json` and `./retail_products.json`: Local file paths.

## 6. Verify Data Upload

To confirm successful data loading, use the following `bq` commands or query directly from the BigQuery UI.

*   Check the first 5 rows of `merchant_center.products`:
    ```bash
    bq head -n 5 $GCP_PROJECT_ID:merchant_center.products
    ```
*   Check the first 5 rows of `retail.products`:
    ```bash
    bq head -n 5 $GCP_PROJECT_ID:retail.products
    ```

*   Check total row count for each table (SQL query):
    ```bash
    bq query --project_id=$GCP_PROJECT_ID --use_legacy_sql=false \
    "SELECT COUNT(*) FROM \`$GCP_PROJECT_ID.merchant_center.products\`"

    bq query --project_id=$GCP_PROJECT_ID --use_legacy_sql=false \
    "SELECT COUNT(*) FROM \`$GCP_PROJECT_ID.retail.products\`"
    ```
    (On Windows Command Prompt, use `%GCP_PROJECT_ID%` instead of `$GCP_PROJECT_ID`. It's important to use double quotes (") for the SQL query string if it contains `$GCP_PROJECT_ID` so the shell expands the variable. If you must use single quotes, place the variable outside or use string concatenation.)

## 7. Important Notes

*   **JSON File Format**: BigQuery expects **Newline Delimited JSON (NDJSON)**.
*   **Error Handling**: Check `bq` output for errors. Use `bq ls -j --project_id=$GCP_PROJECT_ID` and `bq show -j --project_id=$GCP_PROJECT_ID [JOB_ID]` for details.
*   **Schema Auto-detection Limitations**: `--autodetect` is convenient, but explicit schema definition is more reliable for production.

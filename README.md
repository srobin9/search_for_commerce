# Guide: Uploading Product and Event JSON Data from GitHub to BigQuery and GCS

This document guides you on how to load the `retail_products.json` (product catalog) file stored on GitHub into the `retail.products` table in Google BigQuery, and how to process the `recent_retail_events.json` (user events) file and upload it to Google Cloud Storage (GCS). These datasets are sample data intended for use with Vertex AI Search for Retail.

## Table of Contents

1.  [Prerequisites](#1-prerequisites)
2.  [Set Google Cloud Project ID Environment Variable](#2-set-google-cloud-project-id-environment-variable)
3.  [(Optional) Create `retail` BigQuery Dataset](#3-optional-create-retail-bigquery-dataset)
4.  [Download Required Files from GitHub](#4-download-required-files-from-github)
5.  [Process Event Data and Upload to GCS (using `update_event_time.py`)](#5-process-event-data-and-upload-to-gcs-using-update_event_timepy)
6.  [Verify Event Data Uploaded to GCS](#6-verify-event-data-uploaded-to-gcs)
7.  [Load Product Catalog Data into BigQuery `retail.products` Table](#7-load-product-catalog-data-into-bigquery-retailproducts-table)
8.  [Verify BigQuery Product Data Upload](#8-verify-bigquery-product-data-upload)
9.  [Important Notes](#9-important-notes)

## 1. Prerequisites

Before using the `bq` and `gcloud` command-line tools, ensure you have completed the following setup in your local development environment:

*   **Install Google Cloud SDK**: If you haven't already, install it by following the [Google Cloud SDK installation guide](https://cloud.google.com/sdk/docs/install).
*   **Initialize and Authenticate gcloud**:
    ```bash
    gcloud init
    gcloud auth application-default login
    ```
*   **Set Default Project (for gcloud)**: Set the default project for `gcloud` commands. Replace `[YOUR_PROJECT_ID]` with your actual Google Cloud project ID.
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

## 3. (Optional) Create `retail` BigQuery Dataset

If the `retail` dataset for loading product catalog data does not already exist, create it using the following command. The command will use the `$GCP_PROJECT_ID` (or `%GCP_PROJECT_ID%` on Windows) environment variable set above.

*   Create `retail` dataset:
    ```bash
    bq --project_id=$GCP_PROJECT_ID mk --dataset retail
    ```
    (On Windows Command Prompt, use `%GCP_PROJECT_ID%` instead of `$GCP_PROJECT_ID`.)

    If the dataset already exists, you can skip this step. You can list your datasets using `bq --project_id=$GCP_PROJECT_ID ls`.

## 4. Download Required Files from GitHub

Use the following commands to download the necessary JSON files, schema file, and Python script from your GitHub repository to your local environment.
Replace `[YOUR_GITHUB_USERNAME]`, `[YOUR_REPOSITORY_NAME]`, and `[BRANCH_NAME]` with your actual GitHub username, repository name, and branch name (commonly `main` or `master`).

```bash
# Download product catalog data
curl -L -o retail_products.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/retail_products.json

# Download product catalog schema file
curl -L -o retail_products_schema.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/retail_products_schema.json

# Download original user event data file
curl -L -o recent_retail_events.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/recent_retail_events.json

# Download Python script
curl -L -o update_event_time.py https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/update_event_time.py
```
(Alternatively, using `wget`: `wget -O retail_products.json [URL]` )

## 5. Process Event Data and Upload to GCS (using `update_event_time.py`)

Use the `update_event_time.py` Python script to modify the `eventTime` in the downloaded `recent_retail_events.json` file and upload it to Google Cloud Storage (GCS). This script modifies the `eventTime` field in each JSON line to be linearly distributed over the last 33 days and randomizes the time of day. This processed event data, stored in GCS, will be used directly by the Vertex AI Search for Retail service.

**Prerequisites for the script:**
*   Python 3
*   Google Cloud SDK installed and authenticated (`gsutil` is used for uploading).

**Setup:**
1.  **Script Input File:** The `update_event_time.py` script is hardcoded to use `recent_retail_events.json` as input by default. Ensure this file is in the same directory as the script. (If necessary, modify the `INPUT_JSON_FILE` variable within the script.)
2.  **Set GCS Bucket Path Environment Variable:** The script requires the `GCS_BUCKET_PATH` environment variable to be set. This variable must define the GCS bucket and path where the processed file will be uploaded. The filename on GCS will be the same as the input filename (i.e., `recent_retail_events.json`).

    **Linux / macOS:**
    ```bash
    export GCS_BUCKET_PATH="your-gcs-bucket-name/optional-path"
    # Example: export GCS_BUCKET_PATH="my-retail-data-bucket/processed_events"
    # Example (bucket only): export GCS_BUCKET_PATH="my-retail-data-bucket"
    ```

    **Windows (Command Prompt):**
    ```bash
    set GCS_BUCKET_PATH=your-gcs-bucket-name\optional-path
    ```

    **Windows (PowerShell):**
    ```bash
    $env:GCS_BUCKET_PATH="your-gcs-bucket-name/optional-path"
    ```
    *Note: The script will prepend `gs://` if it's missing and handle trailing slashes appropriately.*

**Running the script:**
```bash
python3 update_event_time.py
```
After the script runs, the processed `recent_retail_events.json` file will be uploaded to the specified GCS location, ready for use by Vertex AI Search for Retail.

## 6. Verify Event Data Uploaded to GCS

Verify that the event data uploaded to GCS by the `update_event_time.py` script has been processed correctly.

1.  **Check a few lines of the file content from GCS:**
    Construct the GCS URI using the `GCS_BUCKET_PATH` environment variable and the `recent_retail_events.json` filename.
    ```bash
    # Assuming GCS_BUCKET_PATH environment variable is set
    PROCESSED_EVENT_FILE_GCS_URI=""
    TEMP_GCS_PATH="$GCS_BUCKET_PATH" # Temporary storage for the environment variable value

    # Add gs:// prefix if missing
    if [[ ! "$TEMP_GCS_PATH" == gs://* ]]; then
        TEMP_GCS_PATH="gs://$TEMP_GCS_PATH"
    fi

    # Add trailing slash if missing and append filename
    if [[ "${TEMP_GCS_PATH: -1}" != "/" ]]; then
        PROCESSED_EVENT_FILE_GCS_URI="${TEMP_GCS_PATH}/recent_retail_events.json"
    else
        PROCESSED_EVENT_FILE_GCS_URI="${TEMP_GCS_PATH}recent_retail_events.json"
    fi

    echo "Verifying file: $PROCESSED_EVENT_FILE_GCS_URI"
    gsutil cat "$PROCESSED_EVENT_FILE_GCS_URI" | head -n 5
    ```
    Check that the `eventTime` field in the output JSON objects has been changed to a date within the last 33 days and a random time.

2.  **(Optional) Check file size and metadata on GCS:**
    ```bash
    gsutil ls -lh "$PROCESSED_EVENT_FILE_GCS_URI"
    ```
    Confirm that the file size is as expected.

3.  **(Optional) Download locally for a full review:**
    ```bash
    gsutil cp "$PROCESSED_EVENT_FILE_GCS_URI" ./downloaded_events_check.json
    # less ./downloaded_events_check.json or review with another text editor
    rm ./downloaded_events_check.json # Delete after checking
    ```

If the data appears to be modified as expected, it is ready for use by Vertex AI Search for Retail.

## 7. Load Product Catalog Data into BigQuery `retail.products` Table

Load the downloaded product catalog data (`retail_products.json`) and its corresponding schema file (`retail_products_schema.json`) into the BigQuery `retail.products` table. The command uses the set `$GCP_PROJECT_ID` environment variable.

*   **Load product catalog data (`retail_products.json`):**
    ```bash
    bq load --project_id=$GCP_PROJECT_ID \
      --source_format=NEWLINE_DELIMITED_JSON \
      --schema=./retail_products_schema.json \
      retail.products \
      ./retail_products.json
    ```
    (On Windows Command Prompt, use `%GCP_PROJECT_ID%` instead of `$GCP_PROJECT_ID` and adjust file paths accordingly.)

**Note:**
*   `--source_format=NEWLINE_DELIMITED_JSON`: Indicates that the JSON file is in Newline Delimited JSON (NDJSON) format.
*   `--schema=./retail_products_schema.json`: Specifies the local schema file that BigQuery will use to define the table schema.

## 8. Verify BigQuery Product Data Upload

Confirm that the data has been successfully loaded into the `retail.products` table by using the following `bq` commands or by querying directly from the BigQuery UI in the Google Cloud Console.

*   Check the first 5 rows of the `retail.products` table:
    ```bash
    bq head -n 5 $GCP_PROJECT_ID:retail.products
    ```

*   Check the total row count of the `retail.products` table (SQL query):
    ```bash
    bq query --project_id=$GCP_PROJECT_ID --use_legacy_sql=false \
    "SELECT COUNT(*) FROM \`$GCP_PROJECT_ID.retail.products\`"
    ```
    (On Windows Command Prompt, use `%GCP_PROJECT_ID%` instead of `$GCP_PROJECT_ID`. It's important to use double quotes (") for the SQL query string if it contains `$GCP_PROJECT_ID` so the shell expands the variable.)

## 9. Important Notes

*   **JSON File Format**: BigQuery expects files in **Newline Delimited JSON (NDJSON)** format. Each JSON object must be on an individual line in the file.
*   **Error Handling**: If errors occur during `bq load` operations, check the command output messages. Use `bq ls -j --project_id=$GCP_PROJECT_ID` and `bq show -j --project_id=$GCP_PROJECT_ID [JOB_ID]` for details.
*   **Schema Management**: This guide recommends using an explicit schema file (`retail_products_schema.json`) for the product catalog. This helps ensure data consistency by precisely controlling data types and field modes (NULLABLE, REQUIRED, etc.).

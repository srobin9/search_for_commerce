# GitHub의 제품 및 이벤트 JSON 데이터를 BigQuery 및 GCS로 업로드 가이드

이 문서는 GitHub에 저장된 `retail_products.json` (제품 카탈로그) 파일을 Google BigQuery의 `retail.products` 테이블로 로드하고, `recent_retail_events.json` (사용자 이벤트) 파일을 처리하여 Google Cloud Storage(GCS)에 업로드하는 방법을 안내합니다. GCS에 업로드된 이벤트 데이터는 Vertex AI Search for Retail(구 Search for Commerce)에서 직접 사용될 수 있습니다.

## 목차

1.  [사전 준비 사항](#1-사전-준비-사항)
2.  [Google Cloud 프로젝트 ID 환경 변수 설정](#2-google-cloud-프로젝트-id-환경-변수-설정)
3.  [(선택 사항) `retail` BigQuery 데이터 세트 생성](#3-선택-사항-retail-bigquery-데이터-세트-생성)
4.  [GitHub에서 필요 파일 다운로드](#4-github에서-필요-파일-다운로드)
5.  [이벤트 데이터 처리 및 GCS 업로드 (`update_event_time.py` 사용)](#5-이벤트-데이터-처리-및-gcs-업로드-update_event_timepy-사용)
6.  [GCS에 업로드된 이벤트 데이터 검증](#6-gcs에-업로드된-이벤트-데이터-검증)
7.  [BigQuery `retail.products` 테이블로 제품 카탈로그 데이터 로드](#7-bigquery-retailproducts-테이블로-제품-카탈로그-데이터-로드)
8.  [BigQuery 제품 데이터 업로드 확인](#8-bigquery-제품-데이터-업로드-확인)
9.  [중요 참고 사항](#9-중요-참고-사항)

## 1. 사전 준비 사항

`bq` 및 `gcloud` 명령줄 도구를 사용하기 전에 로컬 개발 환경에서 다음 설정을 완료해야 합니다.

*   **Google Cloud SDK 설치**: 아직 설치하지 않았다면, [Google Cloud SDK 설치 가이드](https://cloud.google.com/sdk/docs/install)를 참조하여 설치합니다.
*   **gcloud 초기화 및 인증**:
    ```bash
    gcloud init
    gcloud auth application-default login
    ```
*   **기본 프로젝트 설정 (gcloud용)**: 다음 명령어로 `gcloud` 명령어의 기본 프로젝트를 설정합니다. `[YOUR_PROJECT_ID]`를 실제 Google Cloud 프로젝트 ID로 변경합니다.
    ```bash
    gcloud config set project [YOUR_PROJECT_ID]
    ```

## 2. Google Cloud 프로젝트 ID 환경 변수 설정

이 가이드의 `bq` 명령어에서 사용할 Google Cloud 프로젝트 ID에 대한 환경 변수를 설정합니다. `[YOUR_PROJECT_ID]`를 실제 프로젝트 ID로 바꿔주세요.

**Linux / macOS:**
```bash
export GCP_PROJECT_ID="[YOUR_PROJECT_ID]"
```
터미널 세션 동안 유지됩니다. 영구적으로 설정하려면 셸 설정 파일(예: `.bashrc`, `.zshrc`)에 추가하세요.

**Windows (Command Prompt):**
```bash
set GCP_PROJECT_ID=[YOUR_PROJECT_ID]
```
현재 Command Prompt 세션 동안 유지됩니다.

**Windows (PowerShell):**
```bash
$env:GCP_PROJECT_ID="[YOUR_PROJECT_ID]"
```
현재 PowerShell 세션 동안 유지됩니다.

설정 후, 환경 변수가 올바르게 설정되었는지 확인합니다 (예: Linux/macOS에서 `echo $GCP_PROJECT_ID`).

## 3. (선택 사항) `retail` BigQuery 데이터 세트 생성

제품 카탈로그 데이터를 로드할 `retail` 데이터 세트가 아직 없다면 다음 명령어를 사용하여 생성합니다. 명령어는 위에서 설정한 `$GCP_PROJECT_ID` (또는 Windows의 경우 `%GCP_PROJECT_ID%`) 환경 변수를 사용합니다.

*   `retail` 데이터 세트 생성:
    ```bash
    bq --project_id=$GCP_PROJECT_ID mk --dataset retail
    ```
    (Windows Command Prompt에서는 `$GCP_PROJECT_ID` 대신 `%GCP_PROJECT_ID%`를 사용하세요.)

    데이터 세트가 이미 존재한다면 이 단계는 건너뛰어도 됩니다. `bq --project_id=$GCP_PROJECT_ID ls` 명령어로 데이터 세트 목록을 확인할 수 있습니다.

## 4. GitHub에서 필요 파일 다운로드

다음 명령어를 사용하여 GitHub 저장소에서 필요한 JSON 파일들, 스키마 파일 및 Python 스크립트를 로컬 환경으로 다운로드합니다.
`[YOUR_GITHUB_USERNAME]`, `[YOUR_REPOSITORY_NAME]`, `[BRANCH_NAME]`을 실제 GitHub 사용자 이름, 저장소 이름, 브랜치 이름으로 변경하세요. 일반적으로 브랜치 이름은 `main` 또는 `master`입니다.

```bash
# 제품 카탈로그 데이터 다운로드
curl -L -o retail_products.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/retail_products.json

# 제품 카탈로그 스키마 파일 다운로드
curl -L -o retail_products_schema.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/retail_products_schema.json

# 사용자 이벤트 데이터 원본 파일 다운로드
curl -L -o recent_retail_events.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/recent_retail_events.json

# Python 스크립트 다운로드
curl -L -o update_event_time.py https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/update_event_time.py
```
(또는 `wget` 사용: `wget -O retail_products.json [URL]` )

## 5. 이벤트 데이터 처리 및 GCS 업로드 (`update_event_time.py` 사용)

다운로드한 `recent_retail_events.json` 파일의 `eventTime`을 수정하고 Google Cloud Storage(GCS)에 업로드하기 위해 `update_event_time.py` Python 스크립트를 사용합니다. 이 스크립트는 각 JSON 라인의 `eventTime` 필드를 최근 33일 동안 선형적으로 분포하도록 수정하고, 시간 부분을 무작위화합니다. 이렇게 처리된 이벤트 데이터는 GCS에 저장되어 Vertex AI Search for Retail 서비스에서 직접 사용됩니다.

**스크립트 실행을 위한 사전 준비 사항:**
*   Python 3
*   Google Cloud SDK 설치 및 인증 완료 (`gsutil`이 업로드에 사용됨).

**설정:**
1.  **스크립트 입력 파일:** `update_event_time.py` 스크립트는 기본적으로 `recent_retail_events.json`을 입력으로 사용하도록 하드코딩되어 있습니다. 이 파일이 스크립트와 동일한 디렉터리에 있는지 확인하십시오. (필요시 스크립트 내 `INPUT_JSON_FILE` 변수 수정)
2.  **GCS 버킷 경로 환경 변수 설정:** 스크립트는 `GCS_BUCKET_PATH` 환경 변수 설정을 필요로 합니다. 이 변수는 처리된 파일이 업로드될 GCS 버킷 및 경로를 정의해야 합니다. GCS에 저장되는 파일명은 입력 파일명과 동일합니다 (즉, `recent_retail_events.json`).

    **Linux / macOS:**
    ```bash
    export GCS_BUCKET_PATH="your-gcs-bucket-name/optional-path"
    # 예시: export GCS_BUCKET_PATH="my-retail-data-bucket/processed_events"
    # 예시 (버킷만 지정 시): export GCS_BUCKET_PATH="my-retail-data-bucket"
    ```

    **Windows (Command Prompt):**
    ```bash
    set GCS_BUCKET_PATH=your-gcs-bucket-name\optional-path
    ```

    **Windows (PowerShell):**
    ```bash
    $env:GCS_BUCKET_PATH="your-gcs-bucket-name/optional-path"
    ```
    *참고: 스크립트는 `gs://` 접두사가 없는 경우 이를 추가하며, 경로 끝의 슬래시를 적절히 처리합니다.*

**스크립트 실행:**
```bash
python3 update_event_time.py
```
스크립트 실행 후, 처리된 `recent_retail_events.json` 파일은 지정된 GCS 위치에 업로드됩니다. 이 파일은 Vertex AI Search for Retail에서 사용될 준비가 된 것입니다.

## 6. GCS에 업로드된 이벤트 데이터 검증

`update_event_time.py` 스크립트로 GCS에 업로드된 이벤트 데이터가 올바르게 처리되었는지 확인합니다.

1.  **GCS에서 파일 내용 일부 확인 (처음 몇 줄):**
    `GCS_BUCKET_PATH` 환경 변수와 `recent_retail_events.json` 파일명을 조합하여 GCS URI를 구성합니다.
    ```bash
    # GCS_BUCKET_PATH 환경 변수가 설정되어 있다고 가정
    PROCESSED_EVENT_FILE_GCS_URI=""
    TEMP_GCS_PATH="$GCS_BUCKET_PATH" # 환경 변수 값 임시 저장

    # gs:// 접두사 추가 (없는 경우)
    if [[ ! "$TEMP_GCS_PATH" == gs://* ]]; then
        TEMP_GCS_PATH="gs://$TEMP_GCS_PATH"
    fi

    # 경로 마지막에 슬래시 추가 (없는 경우) 및 파일명 결합
    if [[ "${TEMP_GCS_PATH: -1}" != "/" ]]; then
        PROCESSED_EVENT_FILE_GCS_URI="${TEMP_GCS_PATH}/recent_retail_events.json"
    else
        PROCESSED_EVENT_FILE_GCS_URI="${TEMP_GCS_PATH}recent_retail_events.json"
    fi

    echo "Verifying file: $PROCESSED_EVENT_FILE_GCS_URI"
    gsutil cat "$PROCESSED_EVENT_FILE_GCS_URI" | head -n 5
    ```
    출력된 JSON 객체들의 `eventTime` 필드가 최근 33일 이내의 날짜와 무작위 시간으로 변경되었는지 확인합니다.

2.  **(선택 사항) GCS에서 파일 크기 및 메타데이터 확인:**
    ```bash
    gsutil ls -lh "$PROCESSED_EVENT_FILE_GCS_URI"
    ```
    파일 크기가 예상과 비슷한지 확인합니다.

3.  **(선택 사항) 로컬로 다운로드하여 전체 파일 검토:**
    ```bash
    gsutil cp "$PROCESSED_EVENT_FILE_GCS_URI" ./downloaded_events_check.json
    # less ./downloaded_events_check.json 또는 다른 텍스트 편집기로 확인
    rm ./downloaded_events_check.json # 확인 후 삭제
    ```

데이터가 예상대로 수정되었다면 Vertex AI Search for Retail에서 사용할 준비가 된 것입니다.

## 7. BigQuery `retail.products` 테이블로 제품 카탈로그 데이터 로드

다운로드한 제품 카탈로그 데이터 (`retail_products.json`)와 해당 스키마 파일 (`retail_products_schema.json`)을 사용하여 BigQuery `retail.products` 테이블에 로드합니다. 명령어는 설정된 `$GCP_PROJECT_ID` 환경 변수를 사용합니다.

*   **제품 카탈로그 데이터 로드 (`retail_products.json`):**
    ```bash
    bq load --project_id=$GCP_PROJECT_ID \
      --source_format=NEWLINE_DELIMITED_JSON \
      --schema=./retail_products_schema.json \
      retail.products \
      ./retail_products.json
    ```    (Windows Command Prompt에서는 `$GCP_PROJECT_ID` 대신 `%GCP_PROJECT_ID%`를 사용하고, 파일 경로를 적절히 수정하세요.)

**참고:**
*   `--source_format=NEWLINE_DELIMITED_JSON`: JSON 파일이 줄 바꿈으로 구분된 JSON (NDJSON) 형식임을 나타냅니다.
*   `--schema=./retail_products_schema.json`: BigQuery가 테이블 스키마를 정의하기 위해 사용할 로컬 스키마 파일을 지정합니다.

## 8. BigQuery 제품 데이터 업로드 확인

`retail.products` 테이블에 데이터가 성공적으로 로드되었는지 확인하려면 다음 `bq` 명령어를 사용하거나 Google Cloud Console의 BigQuery UI에서 직접 쿼리할 수 있습니다.

*   `retail.products` 테이블의 처음 5개 행 확인:
    ```bash
    bq head -n 5 $GCP_PROJECT_ID:retail.products
    ```

*   `retail.products` 테이블의 총 행 수 확인 (SQL 쿼리):
    ```bash
    bq query --project_id=$GCP_PROJECT_ID --use_legacy_sql=false \
    "SELECT COUNT(*) FROM \`$GCP_PROJECT_ID.retail.products\`"
    ```
    (Windows Command Prompt에서는 `$GCP_PROJECT_ID` 대신 `%GCP_PROJECT_ID%`를 사용하세요. SQL 쿼리 문자열 내의 `$GCP_PROJECT_ID`는 셸에 의해 확장되어야 하므로, 쿼리 문자열을 작은따옴표(') 대신 큰따옴표(")로 감싸는 것이 중요합니다.)

## 9. 중요 참고 사항

*   **JSON 파일 형식**: BigQuery는 **줄 바꿈으로 구분된 JSON (NDJSON)** 형식의 파일을 예상합니다. 각 JSON 객체는 파일의 개별 줄에 있어야 합니다.
*   **오류 처리**: `bq load` 작업 중 오류 발생 시 명령어 출력 메시지를 확인하고, `bq ls -j --project_id=$GCP_PROJECT_ID` 및 `bq show -j --project_id=$GCP_PROJECT_ID [JOB_ID]`로 세부 정보를 확인하세요.
*   **스키마 관리**: 이 가이드에서는 제품 카탈로그에 대해 명시적 스키마 파일(`retail_products_schema.json`) 사용을 권장합니다. 이는 데이터 타입 및 필드 모드(NULLABLE, REQUIRED 등)를 정확하게 제어하여 데이터 일관성을 보장하는 데 도움이 됩니다.

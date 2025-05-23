# GitHub JSON 데이터를 BigQuery로 업로딩 가이드

이 문서는 GitHub에 저장된 `merchant_center_products.json` 및 `retail_products.json` 파일을 Google BigQuery의 지정된 데이터 세트 및 테이블로 로드하는 방법을 안내합니다. 모든 로드 작업은 `bq` 명령줄 도구를 사용합니다.

## 목차

1.  [사전 준비 사항](#1-사전-준비-사항)
2.  [Google Cloud 프로젝트 ID 환경 변수 설정](#2-google-cloud-프로젝트-id-환경-변수-설정)
3.  [(선택 사항) BigQuery 데이터 세트 생성](#3-선택-사항-bigquery-데이터-세트-생성)
4.  [GitHub에서 데이터 파일 다운로드](#4-github에서-데이터-파일-다운로드)
5.  [선택 사항: `update_event_time.py`를 사용한 이벤트 데이터 처리](#5-선택-사항-update_event_timepy를-사용한-이벤트-데이터-처리)
6.  [BigQuery 테이블로 데이터 로드](#6-bigquery-테이블로-데이터-로드)
7.  [데이터 업로드 확인](#7-데이터-업로드-확인)
8.  [중요 참고 사항](#8-중요-참고-사항)

## 1. 사전 준비 사항

`bq` 및 `gcloud` 명령줄 도구를 사용하기 전에 로컬 개발 환경에서 다음 설정을 완료해야 합니다.

*   **Google Cloud SDK 설치**: 아직 설치하지 않았다면, [Google Cloud SDK 설치 가이드](https://cloud.google.com/sdk/docs/install)를 참조하여 설치합니다.
*   **gcloud 초기화 및 인증**:
    ```bash
    gcloud init
    gcloud auth application-default login
    ```
*   **기본 프로젝트 설정 (gcloud용)**: 다음 명령어로 `gcloud` 명령어의 기본 프로젝트를 설정합니다. 이 README의 `bq` 명령어는 아래 단계에서 설정할 환경 변수를 사용합니다. `[YOUR_PROJECT_ID]`를 실제 Google Cloud 프로젝트 ID로 변경합니다.
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

## 3. (선택 사항) BigQuery 데이터 세트 생성

데이터를 로드할 `merchant_center` 및 `retail` 데이터 세트가 아직 없다면 다음 명령어를 사용하여 생성합니다. 명령어는 위에서 설정한 `$GCP_PROJECT_ID` (또는 Windows의 경우 `%GCP_PROJECT_ID%`) 환경 변수를 사용합니다.

*   `merchant_center` 데이터 세트 생성:
    ```bash
    bq --project_id=$GCP_PROJECT_ID mk --dataset merchant_center
    ```
*   `retail` 데이터 세트 생성:
    ```bash
    bq --project_id=$GCP_PROJECT_ID mk --dataset retail
    ```
    (Windows Command Prompt에서는 `$GCP_PROJECT_ID` 대신 `%GCP_PROJECT_ID%`를 사용하세요.)

    데이터 세트가 이미 존재한다면 이 단계는 건너뛰어도 됩니다. `bq --project_id=$GCP_PROJECT_ID ls` 명령어로 데이터 세트 목록을 확인할 수 있습니다.

## 4. GitHub에서 데이터 파일 다운로드

다음 명령어를 사용하여 GitHub 저장소에서 JSON 파일을 로컬 환경으로 다운로드합니다.
`[YOUR_GITHUB_USERNAME]`, `[YOUR_REPOSITORY_NAME]`, `[BRANCH_NAME]`을 실제 GitHub 사용자 이름, 저장소 이름, 브랜치 이름으로 변경하세요. 일반적으로 브랜치 이름은 `main` 또는 `master`입니다.

```bash
# merchant_center_products.json 다운로드
curl -L -o merchant_center_products.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/merchant_center_products.json

# retail_products.json 다운로드
curl -L -o retail_products.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/retail_products.json

# 예시: 이벤트 데이터 파일 다운로드 (리포지토리에 있는 경우)
# curl -L -o recent_retail_events.json https://raw.githubusercontent.com/[YOUR_GITHUB_USERNAME]/[YOUR_REPOSITORY_NAME]/[BRANCH_NAME]/recent_retail_events.json
```
(또는 `wget` 사용: `wget -O merchant_center_products.json [URL]` )

## 5. 선택 사항: `update_event_time.py`를 사용한 이벤트 데이터 처리

만약 리포지토리에 `update_event_time.py` 스크립트와 관련 이벤트 데이터 파일(예: `recent_retail_events.json`)이 포함되어 있다면, 이 Python 스크립트를 사용하여 데이터를 처리할 수 있습니다. 스크립트는 각 JSON 라인의 `eventTime` 필드를 최근 33일 동안 선형적으로 분포하도록 수정하고, 시간 부분을 무작위화합니다. 처리된 파일은 `GCS_BUCKET_PATH` 환경 변수로 지정된 Google Cloud Storage(GCS) 버킷에 업로드됩니다.

**스크립트 실행을 위한 사전 준비 사항:**
*   Python 3
*   Google Cloud SDK 설치 및 인증 완료 (`gsutil`이 업로드에 사용됨).

**설정:**
1.  **스크립트 입력 파일:** 스크립트는 기본적으로 `recent_retail_events.json`을 입력으로 사용하도록 하드코딩되어 있습니다. 이 파일이 스크립트와 동일한 디렉터리에 있는지 확인하거나, 스크립트 내의 `INPUT_JSON_FILE` 변수를 수정하십시오.
2.  **GCS 버킷 경로 환경 변수 설정:** 스크립트는 `GCS_BUCKET_PATH` 환경 변수 설정을 필요로 합니다. 이 변수는 처리된 파일이 업로드될 GCS 버킷 및 경로를 정의해야 합니다. GCS에 저장되는 파일명은 입력 파일명과 동일합니다 (예: `recent_retail_events.json`).

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

스크립트 실행 후, 처리된 파일은 지정된 GCS 위치에서 사용할 수 있게 됩니다. 이 GCS URI를 후속 데이터 수집 작업(예: BigQuery 또는 Retail API로 로드)에 사용할 수 있으며, 필요에 따라 로드 명령을 조정해야 합니다.

## 6. BigQuery 테이블로 데이터 로드

다운로드한 (또는 해당하는 경우 처리된) JSON 파일을 사용하여 BigQuery 테이블에 데이터를 로드합니다. 명령어는 설정된 `$GCP_PROJECT_ID` 환경 변수를 사용합니다.

*   `merchant_center_products.json`을 `merchant_center.products` 테이블로 로드:
    ```bash
    bq load --project_id=$GCP_PROJECT_ID \
      --source_format=NEWLINE_DELIMITED_JSON \
      --autodetect \
      merchant_center.products \
      ./merchant_center_products.json
    ```

*   `retail_products.json`을 `retail.products` 테이블로 로드:
    ```bash
    bq load --project_id=$GCP_PROJECT_ID \
      --source_format=NEWLINE_DELIMITED_JSON \
      --autodetect \
      retail.products \
      ./retail_products.json
    ```
    (Windows Command Prompt에서는 `$GCP_PROJECT_ID` 대신 `%GCP_PROJECT_ID%`를 사용하세요.)

    *만약 Python 스크립트를 사용하여 `retail_products.json`(또는 유사한 파일)을 처리하고 GCS에 업로드했다면, `./retail_products.json` 부분을 해당 GCS URI(예: `gs://your-gcs-bucket-name/optional-path/retail_products.json`)로 대체해야 합니다. 스키마(자동 감지 또는 `--schema`로 지정)가 처리된 데이터와 일치하는지 확인하십시오.*

**참고:**
*   `--source_format=NEWLINE_DELIMITED_JSON`: JSON 파일이 줄 바꿈으로 구분된 JSON (NDJSON) 형식임을 나타냅니다.
*   `--autodetect`: BigQuery가 데이터의 스키마를 자동으로 추론하도록 합니다. 프로덕션 환경에서는 명시적 스키마 정의(`--schema`와 스키마 파일 사용)가 더 안정적입니다.
*   `./merchant_center_products.json` 및 `./retail_products.json`: 현재 디렉터리에 있는 로컬 파일 경로입니다.

## 7. 데이터 업로드 확인

데이터가 테이블에 성공적으로 로드되었는지 확인하려면 다음 `bq` 명령어를 사용하거나 Google Cloud Console의 BigQuery UI에서 직접 쿼리할 수 있습니다.

*   `merchant_center.products` 테이블의 처음 5개 행 확인:
    ```bash
    bq head -n 5 $GCP_PROJECT_ID:merchant_center.products
    ```
*   `retail.products` 테이블의 처음 5개 행 확인:
    ```bash
    bq head -n 5 $GCP_PROJECT_ID:retail.products
    ```

*   각 테이블의 총 행 수 확인 (SQL 쿼리):
    ```bash
    bq query --project_id=$GCP_PROJECT_ID --use_legacy_sql=false \
    "SELECT COUNT(*) FROM \`$GCP_PROJECT_ID.merchant_center.products\`"

    bq query --project_id=$GCP_PROJECT_ID --use_legacy_sql=false \
    "SELECT COUNT(*) FROM \`$GCP_PROJECT_ID.retail.products\`"
    ```
    (Windows Command Prompt에서는 `$GCP_PROJECT_ID` 대신 `%GCP_PROJECT_ID%`를 사용하세요. SQL 쿼리 문자열 내의 `$GCP_PROJECT_ID`는 셸에 의해 확장되어야 하므로, 쿼리 문자열을 작은따옴표(') 대신 큰따옴표(")로 감싸는 것이 중요합니다. 만약 작은따옴표를 사용해야 한다면, 변수를 문자열 밖에 두거나 문자열 연결을 사용해야 합니다.)

## 8. 중요 참고 사항

*   **JSON 파일 형식**: BigQuery는 **줄 바꿈으로 구분된 JSON (NDJSON)** 형식의 파일을 예상합니다.
*   **오류 처리**: 로드 중 오류 발생 시 `bq` 명령어 출력 메시지를 확인하고, `bq ls -j --project_id=$GCP_PROJECT_ID` 및 `bq show -j --project_id=$GCP_PROJECT_ID [JOB_ID]`로 세부 정보를 확인하세요.
*   **스키마 자동 감지 제한**: `--autodetect`는 편리하지만, 프로덕션 환경이나 특정 데이터 타입 및 모드(예: REQUIRED, NULLABLE)가 중요한 경우에는 명시적 스키마 정의 (JSON 스키마 파일과 함께 `--schema` 플래그 사용)가 더 안정적입니다.

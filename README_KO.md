# GitHub JSON 데이터를 BigQuery로 업로딩 가이드

이 문서는 GitHub에 저장된 `merchant_center_products.json` 및 `retail_products.json` 파일을 Google BigQuery의 지정된 데이터 세트 및 테이블로 로드하는 방법을 안내합니다. 모든 로드 작업은 `bq` 명령줄 도구와 스키마 자동 감지(`--autodetect`) 기능을 사용합니다.

## 목차

1.  [사전 준비 사항](#1-사전-준비-사항)
2.  [Google Cloud 프로젝트 ID 환경 변수 설정](#2-google-cloud-프로젝트-id-환경-변수-설정)
3.  [(선택 사항) BigQuery 데이터 세트 생성](#3-선택-사항-bigquery-데이터-세트-생성)
4.  [GitHub에서 데이터 파일 다운로드](#4-github에서-데이터-파일-다운로드)
5.  [BigQuery 테이블로 데이터 로드](#5-bigquery-테이블로-데이터-로드)
6.  [데이터 업로드 확인](#6-데이터-업로드-확인)
7.  [중요 참고 사항](#7-중요-참고-사항)

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
```
(또는 `wget` 사용: `wget -O merchant_center_products.json [URL]` )

## 5. BigQuery 테이블로 데이터 로드

다운로드한 JSON 파일을 사용하여 BigQuery 테이블에 데이터를 로드합니다. 각 파일은 해당 데이터 세트의 `products`라는 이름의 테이블로 로드됩니다. `--autodetect` 플래그를 사용하여 스키마를 자동으로 감지합니다. 명령어는 설정된 `$GCP_PROJECT_ID` 환경 변수를 사용합니다.

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

**참고:**
*   `--source_format=NEWLINE_DELIMITED_JSON`: JSON 파일이 줄 바꿈으로 구분된 JSON (NDJSON) 형식임을 나타냅니다.
*   `--autodetect`: BigQuery가 데이터의 스키마를 자동으로 추론하도록 합니다.
*   `./merchant_center_products.json` 및 `./retail_products.json`: 현재 디렉터리에 있는 로컬 파일 경로입니다.

## 6. 데이터 업로드 확인

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

## 7. 중요 참고 사항

*   **JSON 파일 형식**: BigQuery는 **줄 바꿈으로 구분된 JSON (NDJSON)** 형식의 파일을 예상합니다.
*   **오류 처리**: 로드 중 오류 발생 시 `bq` 명령어 출력 메시지를 확인하고, `bq ls -j --project_id=$GCP_PROJECT_ID` 및 `bq show -j --project_id=$GCP_PROJECT_ID [JOB_ID]`로 세부 정보를 확인하세요.
*   **스키마 자동 감지 제한**: `--autodetect`는 편리하지만, 프로덕션 환경에서는 명시적 스키마 정의가 더 안정적일 수 있습니다.

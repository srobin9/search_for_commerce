## Product Discovery - 소매 카탈로그 및 사용자 이벤트 데이터 수집

## 개요

Cloud Retail 서비스와 Retail API를 사용하면 고객이 머신러닝, 추천 시스템 또는 Google Cloud에 대한 높은 수준의 전문 지식 없이도 엔드투엔드 개인 맞춤형 추천 시스템을 구축할 수 있습니다. Retail API 제품 추천 및 제품 검색 서비스를 사용하려면 제품 카탈로그 데이터와 해당 카탈로그와 관련된 사용자 이벤트 데이터를 생성하거나 가져와야 합니다.

이 실습에서는 다양한 기술을 사용하여 제품 카탈로그 및 사용자 이벤트 데이터를 업로드하여 Retail Recommendations AI 및 Product Search 서비스를 위한 환경을 준비합니다. 몇 가지 일반적인 데이터 수집 오류를 살펴보고 Cloud Console 및 Retail API를 사용하여 소매 카탈로그 및 이벤트 데이터를 검토합니다.

이 실습에서는 제품 카탈로그에 Google Merchant Center 데이터 세트의 하위 집합을 사용합니다. Google Merchant Center에서 직접 내보낸 데이터는 Retail 데이터 수집 API와 호환되지 않는 스키마를 사용하므로 실습에 사용된 데이터 세트는 Retail 스키마를 따르도록 수정되었습니다.

## 목표

이 실습에서는 다음 작업을 완료하는 방법을 배웁니다.

*   Retail API를 활성화합니다.
*   BigQuery 및 Cloud Storage에서 제품 카탈로그 및 사용자 이벤트 데이터를 가져옵니다.
*   데이터 가져오기 이벤트 및 오류를 검토합니다.
*   제품 카탈로그 및 사용자 이벤트 데이터를 검토합니다.
*   Retail API를 사용하여 사용자 이벤트 데이터를 업로드합니다.

## 설정 및 요구사항

### Qwiklabs 설정

각 실습에서는 정해진 시간 동안 무료로 새 Google Cloud 프로젝트와 리소스 집합을 제공합니다.

1.  **시크릿 창**을 사용하여 Qwiklabs에 로그인합니다.

2.  실습 액세스 시간(예: `1:15:00`)을 확인하고 해당 시간 내에 완료할 수 있는지 확인합니다.
    일시 중지 기능은 없습니다. 필요한 경우 다시 시작할 수 있지만 처음부터 시작해야 합니다.

3.  준비가 되면 **실습 시작**을 클릭합니다.

4.  실습 사용자 인증 정보(사용자 이름 및 비밀번호)를 확인합니다. 이 정보를 사용하여 Google Cloud Console에 로그인합니다.

5.  **Google Console 열기**를 클릭합니다.

6.  **다른 계정 사용**을 클릭하고 이 실습의 사용자 인증 정보를 복사하여 프롬프트에 붙여넣습니다.
    다른 사용자 인증 정보를 사용하면 오류가 발생하거나 요금이 부과될 수 있습니다.

7.  약관에 동의하고 복구 리소스 페이지를 건너뜁니다.

**참고:** 실습을 완료했거나 다시 시작하려는 경우가 아니면 **실습 종료**를 클릭하지 마십시오. 이렇게 하면 작업이 지워지고 프로젝트가 제거됩니다.

### Cloud Shell 시작

Google Cloud에 있는 동안 자체 컴퓨터에서 Google Cloud를 원격으로 운영할 수 있습니다. 이 실습에서는 Google Cloud Console과 Google Cloud에서 실행되는 명령줄 환경인 Cloud Shell을 모두 사용합니다.

1.  Cloud Console에서 **Cloud Shell 활성화**를 클릭합니다.
    (Cloud Shell 활성화 아이콘 강조 표시됨)

    **참고:** Cloud Shell을 처음 시작하는 경우 Cloud Shell이 무엇인지 설명하는 중간 화면이 표시됩니다. 이 경우 **계속**을 클릭하면 다시 표시되지 않습니다.
    다음은 일회성 화면의 모습입니다.

    (Cloud Shell 대화 상자)

    Cloud Shell을 프로비저닝하고 연결하는 데 몇 분 정도 걸립니다.

    Cloud Shell은 클라우드에 호스팅된 가상 머신에 대한 터미널 액세스를 제공합니다. 가상 머신에는 필요한 모든 개발 도구가 포함되어 있습니다. 영구적인 5GB 홈 디렉터리를 제공하며 Google Cloud에서 실행되므로 네트워크 성능과 인증이 크게 향상됩니다. 이 실습의 대부분 또는 모든 작업은 브라우저만 사용하여 Cloud Console과 Cloud Shell을 통해 수행할 수 있습니다.

2.  Cloud Shell에 연결되면 이미 인증되었고 프로젝트가 프로젝트 ID로 이미 설정되어 있음을 확인할 수 있습니다.

3.  Cloud Shell에서 다음 명령을 실행하여 인증되었는지 확인합니다.
    ```bash
    gcloud auth list
    ```
    복사됨!

    **출력:**
    ```
    Credentialed Accounts
    ACTIVE  ACCOUNT
    *      [my_account]@[my_domain.com]
    To set the active account, run:
        gcloud config set account `ACCOUNT`
    ```
    복사됨!

    **참고:** `gcloud` 명령줄 도구는 Google Cloud의 강력하고 통합된 명령줄 도구입니다. Cloud Shell에 사전 설치되어 제공됩니다. `gcloud`의 기능 중에는 셸에서의 탭 완성이 있습니다. 자세한 내용은 [gcloud CLI 개요 가이드](https://cloud.google.com/sdk/gcloud/overview?hl=ko)를 참조하십시오.

4.  다음 명령을 실행하여 이 실습에 올바른 프로젝트를 사용하고 있는지 확인합니다.
    ```bash
    gcloud config list project
    ```
    복사됨!

    **출력:**
    ```
      [core]
      project = [PROJECT_ID]
    ```
    올바른 프로젝트가 표시되지 않으면 다음 명령으로 설정할 수 있습니다.
    ```bash
    gcloud config set project [PROJECT_ID]
    ```    복사됨!

    **출력:**
    ```
    Updated property [core/project].
    ```

## 작업 1. Retail API 활성화

Retail Recommendations AI 또는 Retail Search API를 사용하기 전에 Retail API를 활성화해야 합니다.

1.  **탐색 메뉴**(탐색 메뉴 아이콘)에서 인공 지능 섹션 아래 **모든 제품 보기**를 클릭하고 **Search for Retail**을 선택합니다.

2.  **API 켜기**를 클릭합니다.

3.  **계속**을 클릭하고 **동의**를 클릭하여 데이터 약관에 동의합니다.

4.  **시작하기**를 클릭합니다.

## 작업 2. 제품 카탈로그 및 사용자 이벤트 데이터 가져오기

이 작업에서는 BigQuery에서 제품 카탈로그 데이터를, Cloud Storage에서 사용자 이벤트 데이터를 가져옵니다.

### BigQuery에서 Merchant Center 제품 테이블 스키마 데이터 가져오기

`merchant_center.products` 테이블에는 Google Merchant Center 제품 테이블 스키마를 사용하여 Google Merchant Center의 테스트 계정에서 내보낸 카탈로그 데이터가 포함되어 있습니다. 이 데이터 세트는 이전 Recommendations AI Console 또는 API를 사용하여 카탈로그 데이터로 가져올 수 있습니다. Recommendations AI API를 대체하는 Retail API는 현재 Merchant Center 제품 테이블 스키마를 사용하는 데이터 가져오기를 지원하지 않으며 모든 데이터 가져오기는 Retail 스키마를 사용해야 합니다. 데이터 가져오기 오류를 검사하는 방법을 확인하기 위해 Retail API를 사용하여 이 데이터를 계속 가져오려고 시도합니다.

1.  GCP Console에서 **Search for Retail > 데이터**를 클릭하여 Retail 데이터 관리 페이지를 엽니다.

2.  **카탈로그** 탭이 선택되어 있는지 확인하고 **가져오기**를 클릭합니다.

3.  제품 카탈로그를 가져오도록 가져오기 매개변수를 다음과 같이 구성합니다.

    *   **가져오기 유형**으로 **제품 카탈로그**를 선택합니다.
    *   **데이터 소스**로 **BigQuery**를 선택합니다.
    *   **가져오기 브랜치**로 **브랜치 0**을 선택합니다.
    *   **BigQuery 테이블**에서 **찾아보기**를 클릭합니다.

4.  검색창에 `products`를 입력하고 **검색**을 클릭합니다.

5.  `products - Dataset: merchant_center table`의 라디오 버튼을 선택합니다.

    소스 테이블에 `id` 필드가 없어 진행할 수 없습니다.

    스키마 때문에 데이터에 훨씬 더 많은 문제가 있습니다.

### BigQuery에서 Retail 제품 스키마 데이터 가져오기

이 작업에서는 Retail 제품 스키마를 사용하는 BigQuery 테이블에서 카탈로그로 제품 데이터를 가져옵니다.

1.  GCP Console의 **탐색 메뉴**(탐색 메뉴 아이콘)에서 인공 지능 섹션 아래 **모든 제품 보기**를 클릭하고 **Search for Retail > 데이터**를 선택하여 Retail 데이터 관리 페이지를 엽니다.

2.  **카탈로그** 탭이 선택되어 있는지 확인하고 **가져오기**를 클릭합니다.

3.  제품 카탈로그를 가져오도록 가져오기 매개변수를 다음과 같이 구성합니다.

    *   **가져오기 유형**으로 **제품 카탈로그**를 선택합니다.
    *   **가져오기 브랜치**로 **브랜치 0**을 선택합니다.
    *   **데이터 소스**로 **BigQuery**를 선택합니다.
    *   **BigQuery 테이블**에서 **찾아보기**를 클릭합니다.

4.  검색창에 `products`를 입력하고 **검색**을 클릭합니다.

5.  `products - Dataset: retail table`의 라디오 버튼을 선택합니다.

6.  **선택**을 클릭합니다.

    **참고:** 테이블 이름을 클릭하면 데이터 카탈로그 페이지가 열리므로 Retail 제품 가져오기 페이지로 돌아가야 합니다.
7.  **가져오기**를 클릭합니다.
    다음과 유사한 메시지가 포함된 팝업 메시지가 나타날 때까지 기다려야 합니다.

    (샘플 소매 카탈로그 가져오기 팝업)
    `Successfully scheduled import operation import-products-6583047802807380211. It may take up to 5 minutes to see your new long running operation in the Integration Activity panel.` (가져오기 작업 import-products-6583047802807380211이 성공적으로 예약되었습니다. 통합 활동 패널에 새 장기 실행 작업이 표시되는 데 최대 5분이 걸릴 수 있습니다.)

    가져오기 작업이 예약되면 정기적인 데이터 가져오기 작업을 예약하는 데 사용할 수 있는 `gcloud scheduler` 명령의 세부 정보도 표시됩니다.

8.  **취소**를 클릭하여 가져오기 페이지를 닫고 Retail 데이터 페이지로 돌아가 카탈로그 데이터 가져오기 작업의 상태를 확인합니다.

9.  가져오기 작업이 성공적으로 예약되었다는 팝업을 닫으려면 **X**를 클릭합니다.

10. Search for Retail 탐색 메뉴에서 **데이터**를 클릭한 다음 **활동 상태**를 클릭하여 가져오기 작업의 진행 상황을 모니터링합니다.

    가져오기 작업은 제품 카탈로그 가져오기 활동 섹션의 가져오기 작업 상태가 **성공**으로 변경되는 데 1~2분이 걸립니다. 총 1268개의 항목이 가져옵니다.

### Cloud Storage에서 사용자 이벤트 데이터 가져오기

이 작업에서는 BigQuery 테이블에서 사용자 이벤트 데이터를 가져옵니다.

1.  GCP Console의 **탐색 메뉴**(탐색 메뉴 아이콘)에서 인공 지능 섹션 아래 **모든 제품 보기**를 클릭하고 **Search for Retail > 데이터**를 선택하여 Retail 데이터 관리 페이지를 엽니다.

2.  **이벤트** 탭이 선택되어 있는지 확인하고 **가져오기**를 클릭합니다.

3.  제품 카탈로그를 가져오도록 가져오기 매개변수를 다음과 같이 구성합니다.

    *   **가져오기 유형**으로 **사용자 이벤트**를 선택합니다.
    *   **데이터 소스**로 **Google Cloud Storage**를 선택합니다.
    *   **Google Cloud Storage 위치**에서 **찾아보기** 버튼을 클릭합니다.

4.  `bucket_name`이라는 스토리지 버킷으로 이동하여 `recent_retail_events.json` 파일을 선택합니다.

5.  파일 이름이 선택되었는지 확인하려면 **파일 이름**을 클릭합니다.

6.  **선택**을 클릭합니다.

7.  **가져오기**를 클릭합니다.

    다음과 유사한 메시지가 포함된 팝업 메시지가 나타날 때까지 기다려야 합니다.

    (샘플 소매 이벤트 가져오기 팝업)
    `Successfully scheduled import operation import-products-6583047802807380211. It may take up to 5 minutes to see your new long running operation in the Integration Activity panel.` (가져오기 작업 import-products-6583047802807380211이 성공적으로 예약되었습니다. 통합 활동 패널에 새 장기 실행 작업이 표시되는 데 최대 5분이 걸릴 수 있습니다.)

    가져오기 작업이 예약되면 정기적인 이벤트 가져오기 작업을 예약하는 데 사용할 수 있는 `gcloud scheduler` 명령의 세부 정보도 표시됩니다.

8.  `gcloud scheduler` 명령이 표시된 상태로 가져오기 작업이 예약될 때까지 기다립니다.

9.  **취소**를 클릭하여 가져오기 페이지를 닫고 Retail 데이터 페이지로 돌아가 이벤트 데이터 가져오기 작업의 상태를 확인합니다.

10. 가져오기 작업이 성공적으로 예약되었다는 팝업을 닫으려면 **X**를 클릭합니다.

11. Search for Retail 탐색 메뉴에서 **데이터**를 클릭한 다음 **활동 상태**를 클릭하여 가져오기 작업의 진행 상황을 모니터링합니다.

    가져오기 작업은 사용자 이벤트 가져오기 활동 섹션의 가져오기 작업 상태가 **성공**으로 변경되는 데 1~2분이 걸립니다. 약 32,000개의 항목이 가져오고 5개의 항목이 실패합니다.

## 작업 3. 데이터 가져오기 이벤트 및 오류 검토

이 작업에서는 데이터 가져오기 작업을 검토하고 잘못된 데이터가 발견되었을 때 가져오기 작업에서 기록한 일부 오류를 살펴봅니다.

1.  Search for Retail 탐색 메뉴에서 **데이터**를 클릭한 다음 **활동 상태**를 클릭하여 가져오기 작업의 진행 상황을 모니터링합니다.

2.  **사용자 이벤트** 탭을 클릭한 다음 세부 정보 열에서 **전체 오류 로그 보기**를 클릭하여 오류를 검토합니다.

    이렇게 하면 소스 데이터가 있던 Cloud Storage 버킷의 `/error` 폴더가 열립니다.

3.  가져온 이벤트 데이터 파일에 해당하는 파일 이름을 클릭합니다. 크기는 약 1킬로바이트입니다.

4.  **다운로드**를 클릭하여 파일을 다운로드한 다음 컴퓨터에서 열어 오류 세부 정보를 검토합니다. 해당 이벤트의 데이터 스키마와 관련된 다양한 문제로 인해 가져오기에 실패한 5개의 이벤트가 표시됩니다.

    **오류 로그 내용**
    ```json
    {
      "code": 3,
      "message": "'userEvent.productDetails' is required for eventType add-to-cart.",
      "details": [{
        "@type": "type.googleapis.com/google.protobuf.Struct",
        "value": {
          "line_number": 475
        }
      }]
    }
    {
      "code": 3,
      "message": "link: Cannot find field.",
      "details": [{
        "@type": "type.googleapis.com/google.protobuf.Struct",
        "value": {
          "line_number": 478
        }
      }]
    }
    ```

5.  Cloud Console로 돌아가 Cloud Storage 탭을 닫습니다.

6.  Retail 활동 상태 탭을 열고 **닫기**를 클릭하여 활동 상태 팝아웃을 닫습니다.

## 작업 4. 제품 카탈로그 및 사용자 이벤트 데이터 검토

이 작업에서는 가져온 제품 및 이벤트 데이터를 검토합니다.

1.  Search for Retail 탐색 메뉴에서 **데이터**를 클릭한 다음 **카탈로그** 탭이 선택되어 있는지 확인합니다.

2.  **브랜치 이름**은 **브랜치 0 (기본값)**으로 설정된 상태로 둡니다.

    카탈로그 제품 목록에는 카탈로그에 업로드된 1268개의 제품 레코드가 표시되며 그중 746개가 재고 있음 상태입니다.

3.  **필터**에 `GGOEGCBT136699`를 입력합니다.

    Google Yellow YoYo의 제품 레코드가 표시됩니다. 제품이 품절 상태임을 확인합니다.

4.  **링크 아이콘**을 클릭하여 링크를 열어봅니다. 열린 페이지에는 "죄송합니다. 이 페이지를 사용할 수 없습니다."라고 표시됩니다.

5.  열린 제품 탭을 닫고 Search for Retail 데이터 페이지로 돌아갑니다.

6.  **필터**에 `GGOECAEB163612`를 입력합니다.

    Google Black Cloud Tee의 제품 레코드가 표시됩니다. 이 제품은 재고 있음 상태임을 확인합니다.

7.  **링크 아이콘**을 클릭하여 링크를 엽니다. Google Merchandise 스토어의 제품 페이지가 열립니다.

8.  열린 제품 탭을 닫고 Search for Retail 데이터 페이지로 돌아갑니다.

## 작업 5. Retail API를 사용하여 사용자 이벤트 데이터 업로드

이제 `curl` 및 기타 명령줄 유틸리티를 사용하여 Retail Recommendations AI API를 호출하여 요청을 만들고, 추천을 받고, 결과를 필터링하고 구체화하는 방법을 살펴봅니다.

### 요청 인증을 위한 IAM 서비스 계정 만들기

1.  프로젝트 ID를 저장할 환경 변수를 만듭니다.
    ```bash
    export PROJECT_ID=$(gcloud config get-value core/project)
    ```
    복사됨!

2.  Retail API에 대한 제어된 액세스를 위해 IAM 서비스 계정을 만듭니다.
    ```bash
    export SA_NAME="retail-service-account"
    gcloud iam service-accounts create $SA_NAME --display-name $SA_NAME
    ```
    복사됨!

3.  서비스 계정을 Retail 편집자 IAM 역할에 바인딩합니다.
    ```bash
    gcloud projects add-iam-policy-binding ${PROJECT_ID} \
     --member="serviceAccount:$SA_NAME@${PROJECT_ID}.iam.gserviceaccount.com" \
     --role="roles/retail.editor"
    ```
    복사됨!

### 실습 사용자 계정이 새 서비스 계정으로 가장을 사용하도록 허용

서비스 계정 토큰 생성자 역할을 사용하여 실습 사용자에 대한 서비스 계정의 역할 바인딩을 만들면 실습 사용자가 서비스 계정 가장을 사용하여 서비스 계정에 대한 제한된 기간의 인증 토큰을 안전하게 생성할 수 있습니다. 그런 다음 이러한 토큰을 사용하여 API 및 서비스에 대한 액세스를 대화형으로 테스트할 수 있습니다.

1.  가장을 허용하도록 사용자 계정에 대한 Retail API 서비스 계정의 역할 바인딩을 만듭니다.
    ```bash
    export USER_ACCOUNT=$(gcloud config list --format 'value(core.account)')
    gcloud iam service-accounts add-iam-policy-binding $SA_NAME@$PROJECT_ID.iam.gserviceaccount.com --member "user:$USER_ACCOUNT" --role roles/iam.serviceAccountTokenCreator
    ```
    복사됨!

2.  Retail API에 대한 임시 액세스 토큰을 생성합니다.
    ```bash
    export ACCESS_TOKEN=$(gcloud auth print-access-token --impersonate-service-account $SA_NAME@$PROJECT_ID.iam.gserviceaccount.com )
    ```
    복사됨!

    이 명령은 서비스 계정 토큰 생성자 역할이 전파되는 데 최대 10분이 걸릴 수 있으므로 실패할 수 있습니다. 실패하면 1분 후에 이 명령을 다시 시도하고 성공할 때까지 다시 시도하십시오. 또한 명령이 가장을 사용하고 있음을 알리는 경고가 표시됩니다. 이는 예상된 동작입니다.

### Retail API에 사용자 이벤트 제출

JSON 형식의 사용자 이벤트 데이터를 Retail API `userEvents:write` 메서드에 전달하여 샘플 사용자 이벤트를 Retail API에 업로드합니다.

1.  샘플 사용자 이벤트 JSON 데이터를 환경 변수에 저장합니다.
    ```bash
    DATA='{
      "eventType": "detail-page-view",
      "visitorId": "GA1.3.1260529204.1622654859",
      "productDetails": [{
       "product": {
         "id": "GGOEGDHB163199"
       }
     }, {
       "product": {
         "id": "GGOEAAKQ137410"
       }
     }
      ]
    }'
    ```
    복사됨!

2.  Retail API `userEvents.write` 메서드를 사용하여 카탈로그에 사용자 이벤트 데이터를 쓰는 REST API URL을 환경 변수에 저장합니다.
    ```bash
    URL="https://retail.googleapis.com/v2/projects/${PROJECT_ID}/locations/global/catalogs/default_catalog/userEvents:write?access_token=${ACCESS_TOKEN}"
    ```
    복사됨!

    이는 Retail API에 사용자 이벤트 데이터를 쓰는 REST API URL입니다. URL에는 프로젝트 ID에 대한 bash 환경 변수 대체와 `access_token`이라는 인라인 매개변수인 액세스 토큰이 포함되어 있습니다. 이 토큰은 이전에 가장을 사용하여 생성한 서비스 계정을 사용하여 요청을 인증합니다.

3.  `curl`을 사용하여 REST API를 통해 사용자 이벤트를 업로드합니다.
    ```bash
    curl -H 'Content-Type: application/json' -X POST -d "${DATA}"  $URL
    ```
    복사됨!

    `curl`을 사용하여 `userEvents:write` 메서드를 호출하고 이벤트 데이터를 POST 요청의 JSON 데이터 페이로드로 전달했습니다.

    **출력**
    ```json
      {
     "eventType": "detail-page-view",
     "visitorId": "GA1.3.1260529204.1622654859",
     "eventTime": "2021-06-28T18:39:26.691324Z",
     "productDetails": [
       {
         "product": {
           "name": "projects/610724409905/locations/global/catalogs/default_catalog/branches/0/products/GGOEGDHB163199",
           "id": "GGOEGDHB163199",
           "type": "PRIMARY",
           "primaryProductId": "GGOEGDHB163199",
           "categories": [
             "Drinkware"
           ],
           "title": "Google Chrome Dino Light Up Water Bottle",
           "priceInfo": {
             "currencyCode": "USD",
             "price": 24
           },
           "availability": "IN_STOCK",
           "uri": "https://shop.googlemerchandisestore.com/Google+Redesign/Google+Chrome+Dino+Light+Up+Water+Bottle",
           "images": [
             {
               "uri": "https://shop.googlemerchandisestore.com/store/20160512512/assets/items/images/GGOEGDHB163199.jpg"
             }
           ]
         }
       },
       {
         "product": {
           "name": "projects/610724409905/locations/global/catalogs/default_catalog/branches/0/products/GGOEAAKQ137410",
           "id": "GGOEAAKQ137410",
           "type": "PRIMARY",
           "primaryProductId": "GGOEAAKQ137410",
           "categories": [
             "Apparel"
           ],
           "title": "Android Iconic Sock",
           "priceInfo": {
             "currencyCode": "USD",
             "price": 17
           },
           "availability": "IN_STOCK",
           "uri": "https://shop.googlemerchandisestore.com/Google+Redesign/Apparel/Android+Iconic+Sock",
           "images": [
             {
               "uri": "https://shop.googlemerchandisestore.com/store/20160512512/assets/items/images/GGOEAAKQ137410.jpg"
             }
           ]
         }
       }
     ]
      }
    ```
    업로드가 성공하면 제품 URL 및 이미지와 같은 관련 제품 데이터와 이벤트 타임스탬프를 포함하여 응답에 형식이 지정된 데이터가 표시됩니다. 그렇지 않으면 오류가 표시됩니다. 가장 일반적인 오류는 잘못된 형식의 페이로드, URL 또는 잘못된 토큰과 관련됩니다.

## 축하합니다!

축하합니다. 다양한 기술을 사용하여 Retail 제품 카탈로그 및 사용자 이벤트 데이터를 성공적으로 가져왔고, 몇 가지 일반적인 데이터 수집 오류를 살펴보고, Cloud Console 및 Retail API를 사용하여 Retail 카탈로그 및 이벤트 데이터를 검토했습니다.

## 실습 종료

실습을 완료하면 **실습 종료**를 클릭합니다. Google Cloud Skills Boost는 사용한 리소스를 제거하고 계정을 정리합니다.

실습 경험을 평가할 기회가 주어집니다. 해당 별표 수를 선택하고 의견을 입력한 다음 **제출**을 클릭합니다.

별표 수는 다음을 나타냅니다.

*   별 1개 = 매우 불만족
*   별 2개 = 불만족
*   별 3개 = 보통
*   별 4개 = 만족
*   별 5개 = 매우 만족

피드백을 제공하고 싶지 않으면 대화 상자를 닫을 수 있습니다.

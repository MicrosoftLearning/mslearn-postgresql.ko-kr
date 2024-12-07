---
lab:
  title: 추출 및 추상 요약 수행
  module: Summarize data using Azure AI Services and Azure Database for PostgreSQL
---

# 추출 및 추상 요약 수행

Margie's Travel에서 관리하는 임대 부동산 앱은 부동산 관리자에게 임대 목록을 설명할 방법을 제공합니다. 시스템에 있는 설명은 대부분 임대 부동산, 이웃, 지역 관광 명소, 상점, 기타 편의 시설에 대해 자세히 설명하기 때문에 길이가 깁니다. 앱에 사용할 새로운 AI 기반 기능을 구현할 때 한 가지 기능에 대한 요청이 들어왔습니다. 생성형 AI로 긴 설명에 대해 간결한 요약을 만들어 사용자가 부동산을 보다 쉽게 검토할 수 있도록 하는 것입니다. 이 연습에서는 Azure Database For PostgreSQL 유연한 서버에서 `azure_ai` 확장을 사용하여 임대 부동산 설명에 대한 추상 및 추출 요약을 수행하고 그 결과로 나오는 요약을 비교합니다.

## 시작하기 전에

관리 권한이 있는 [Azure 구독](https://azure.microsoft.com/free)이 필요합니다.

### Azure 구독에 리소스 배포

이 단계에서는 Azure Cloud Shell에서 Azure CLI 명령을 사용하여 리소스 그룹을 만들고 Bicep 스크립트를 실행하여 이 연습을 완료하는 데 필요한 Azure 서비스를 Azure 구독에 배포하는 방법을 안내합니다.

1. 웹 브라우저를 열고 [Azure Portal](https://portal.azure.com/)로 이동합니다.

2. Azure Portal 도구 모음의 **Cloud Shell** 아이콘을 선택하여 브라우저 창 아래쪽에 새 [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 창을 엽니다.

    ![Cloud Shell 아이콘을 빨간색 상자로 강조 표시한 Azure 도구 모음의 스크린샷.](media/15-portal-toolbar-cloud-shell.png)

    메시지가 표시되는 경우 *Bash* 셸을 여는 데 필요한 옵션을 선택합니다. 이전에 *PowerShell* 콘솔을 사용한 적이 있다면 이것을 *Bash* 셸로 전환합니다.

3. Cloud Shell 프롬프트에서 다음을 입력하여 연습 리소스가 포함된 GitHub 리포지토리를 복제합니다.

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. 그 다음 Azure CLI 명령을 사용하여 Azure 리소스를 만들 때 중복 입력을 줄이기 위해 변수를 정의하는 세 가지 명령을 실행합니다. 이 변수는 리소스 그룹에 할당할 이름(`RG_NAME`), 리소스를 배포할 Azure 지역(`REGION`), PostgreSQL 관리자 로그인용 임의 생성 암호(`ADMIN_PASSWORD`)를 나타냅니다.

    첫 번째 명령에서는 해당 변수에 `eastus` 지역을 할당하지만, 원하는 위치로 바꿀 수도 있습니다. 그러나 기본값을 바꾸는 경우 다른 [추상 요약을 지원하는 Azure 지역](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support)을 선택해야 이 학습 경로의 모듈에 있는 모든 작업을 완료할 수 있습니다.

    ```bash
    REGION=eastus
    ```

    다음 명령에서는 이 연습에서 사용할 모든 리소스를 저장하는 리소스 그룹에 사용할 이름을 할당합니다. 해당 변수에는 `rg-learn-postgresql-ai-$REGION` 리소스 그룹 이름을 할당하며, 여기서 `$REGION` 부분은 앞에서 지정한 위치입니다. 그러나 원하는 다른 리소스 그룹 이름으로 변경할 수 있습니다.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    마지막 명령에서는 PostgreSQL 관리자 로그인에 사용할 암호를 임의로 생성합니다. 나중에 PostgreSQL 유연한 서버에 연결하는 데 사용하기 위해 이 암호를 안전한 장소에 **복사해 두어야 합니다**.

    ```bash
    a=()
    for i in {a..z} {A..Z} {0..9}; 
        do
        a[$RANDOM]=$i
        done
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]::18}")
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo $ADMIN_PASSWORD
    ```

5. 두 개 이상의 Azure 구독에 액세스할 수 있고 기본 구독이 이 연습에 대한 리소스 그룹 및 기타 리소스를 만드는 데 사용할 구독이 아닌 경우, 이 명령을 실행할 때 적절한 구독을 설정하고 `<subscriptionName|subscriptionId>` 토큰을 사용할 구독의 이름 또는 ID로 바꿉니다.

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. 다음 Azure CLI 명령을 실행하여 리소스 그룹을 만듭니다.

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. 마지막으로 Azure CLI를 사용하여 Bicep 배포 스크립트를 실행하는 방법으로 리소스 그룹에 Azure 리소스를 프로비전합니다.

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep 배포 스크립트는 이 연습을 완료하는 데 필요한 Azure 서비스를 리소스 그룹으로 프로비전합니다. 배포 리소스에는 Azure Database for PostgreSQL 유연한 서버, Azure OpenAI, Azure AI 언어 서비스가 포함됩니다. 또한 Bicep 스크립트는 azure.extensions 서버 매개 변수를 통해 PostgreSQL 서버의 _허용 목록_에 `azure_ai` 및 `vector` 확장을 추가하고, 서버에 `rentals` 데이터베이스를 만들고, Azure OpenAI Service에 `text-embedding-ada-002` 모델을 사용한 `embedding` 배포를 추가하는 등 몇 가지 구성 단계를 수행합니다. 이 Bicep 파일은 이 학습 경로의 모든 모듈에서 공유되므로 일부 연습에서는 배포된 리소스 중 일부만 사용할 수 있다는 점에 유의해야 합니다.

    배포를 완료하려면 보통 몇 분 정도 걸립니다. Cloud Shell에서 모니터링하거나 위에서 만든 리소스 그룹에 대한 **배포** 페이지로 이동하여 배포 진행 상황을 관찰할 수 있습니다.

8. 리소스 배포가 완료되면 Cloud Shell 창을 닫습니다.

### 배포 오류 문제 해결

Bicep 배포 스크립트를 실행할 때 몇 가지 오류가 발생할 수 있습니다.

- 이전에 이 학습 경로에 대한 Bicep 배포 스크립트를 실행한 후 리소스를 삭제한 경우, 리소스를 삭제하고 48시간 이내에 스크립트를 다시 실행하려고 하면 다음과 같은 오류 메시지가 표시될 수 있습니다.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    이 메시지가 표시되면 위의 `azure deployment group create` 명령을 수정하여 `restore` 매개 변수를 `true`에 해당하도록 설정하고 다시 실행합니다.

- 선택한 지역이 특정 리소스를 프로비전하지 못하도록 제한된 경우 `REGION` 변수를 다른 위치로 설정하고 명령을 다시 실행하여 리소스 그룹을 만들고 Bicep 배포 스크립트를 실행해야 합니다.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 책임 있는 AI 계약을 수락해야 하는 요구 사항으로 인해 스크립트가 AI 리소스를 만들 수 없는 경우 다음과 같은 오류가 발생할 수 있습니다. 이 경우 Azure Portal 사용자 인터페이스를 사용하여 Azure AI 서비스 리소스를 만든 다음 배포 스크립트를 다시 실행합니다.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```


## Azure Cloud Shell에서 psql을 사용하여 데이터베이스에 연결

이 작업에서는 [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)의 [psql 명령줄 유틸리티](https://www.postgresql.org/docs/current/app-psql.html)를 사용하여 Azure Database for PostgreSQL 유연한 서버의 `rentals` 데이터베이스에 연결합니다.

1. [Azure Portal](https://portal.azure.com/)에서 새로 만든 Azure Database for PostgreSQL 유연한 서버 인스턴스로 이동합니다.

2. 리소스 메뉴의 **설정**에서 **데이터베이스**를 선택하고 `rentals` 데이터베이스에 대한 **연결**을 선택합니다.

    ![Azure Database for PostgreSQL 데이터베이스 페이지의 스크린샷. rentals 데이터베이스에 대한 데이터베이스 및 연결이 빨간색 상자로 강조 표시됩니다.](media/15-postgresql-rentals-database-connect.png)

3. Cloud Shell의 "사용자 pgAdmin에 대한 암호" 프롬프트에서 **pgAdmin** 로그인에 대해 임의로 생성된 암호를 입력합니다.

    로그인하면 `rentals` 데이터베이스에 대한 `psql` 프롬프트가 표시됩니다.

4. 이 연습의 이후 부분에서는 계속 Cloud Shell에서 작업하므로 창의 오른쪽 위에 있는 **최대화** 버튼을 선택하여 브라우저 창 내에서 해당 창을 확장하면 도움이 될 수 있습니다.

    ![빨간색 상자로 강조 표시된 최대화 버튼이 있는 Azure Cloud Shell 창의 스크린샷.](media/15-azure-cloud-shell-pane-maximize.png)

## 샘플 데이터를 이용해 데이터베이스 작성

`azure_ai` 확장을 탐색하기 전에 `rentals` 데이터베이스에 테이블을 몇 개 추가하고 샘플 데이터로 채워 확장의 기능을 검토할 때 사용할 정보를 준비합니다.

1. 다음 명령을 실행하여 임대 부동산 목록 및 고객 리뷰 데이터를 저장하기 위한 `listings` 및 `reviews` 테이블을 만듭니다.

    ```sql
    DROP TABLE IF EXISTS listings;

    CREATE TABLE listings (
        id int,
        name varchar(100),
        description text,
        property_type varchar(25),
        room_type varchar(30),
        price numeric,
        weekly_price numeric
    );
    ```

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. 다음으로, `COPY` 명령을 사용하여 위에서 만든 각 테이블에 CSV 파일의 데이터를 로드합니다. 먼저 다음 명령을 사용하여 `listings` 테이블을 채웁니다.

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    명령의 출력은 `COPY 50`이어야 합니다. 이는 CSV 파일의 테이블에 50개의 행이 기록되었음을 나타냅니다.

3. 마지막으로 아래 명령을 실행하여 고객 리뷰를 `reviews` 테이블에 로드합니다.

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    명령의 출력은 `COPY 354`여야 합니다. 이는 CSV 파일의 테이블에 354개의 행이 기록되었음을 나타냅니다.

## `azure_ai` 확장 설치 및 구성

`azure_ai` 확장을 사용하기 전에는 확장을 데이터베이스에 설치하고 Azure AI Services 리소스에 연결하도록 구성해야 합니다. `azure_ai` 확장을 사용하면 Azure OpenAI 및 Azure AI 언어 서비스를 데이터베이스에 통합할 수 있습니다. 데이터베이스에서 확장을 사용하도록 설정하려면 다음 단계를 수행합니다.

1. `psql` 프롬프트에서 다음 명령을 실행하여, 환경을 설정할 때 실행한 Bicep 배포 스크립트를 통해 `azure_ai` 및 `vector`확장이 서버의 _허용 목록_에 성공적으로 추가되었는지 확인합니다.

    ```sql
    SHOW azure.extensions;
    ```

    이 명령은 서버의 _허용 목록_에 확장 목록을 표시합니다. 모든 항목이 올바르게 설치된 경우 다음과 같이 출력에 `azure_ai` 및 `vector`가 포함되어 있어야 합니다.

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Azure Database for PostgreSQL 유연한 서버 데이터베이스에서 확장을 설치 및 사용하려면 [PostgreSQL 확장 사용 방법](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions)에 설명된 대로 서버의 _허용 목록_에 추가해야 합니다.

2. 이제 [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html) 명령을 사용하여 `azure_ai` 확장을 설치할 수 있습니다.

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` 명령은 스크립트 파일을 실행하여 데이터베이스에 새 확장을 로드합니다. 이 스크립트는 일반적으로 함수, 데이터 형식, 스키마와 같은 새 SQL 개체를 만듭니다. 동일한 이름의 확장이 이미 있는 경우 오류가 표시됩니다. `IF NOT EXISTS` 문을 추가하면 확장이 이미 설치된 경우 오류를 표시하지 않고 명령을 실행할 수 있습니다.

## Azure AI 서비스 계정 연결

`azure_ai` 확장의 `azure_cognitive` 스키마에 포함된 Azure AI 서비스 통합은 데이터베이스에서 직접 액세스할 수 있는 다양한 AI 언어 기능 세트를 제공합니다. 텍스트 요약 기능은 [Azure AI 언어 서비스](https://learn.microsoft.com/azure/ai-services/language-service/overview)를 통해 지원됩니다.

1. `azure_ai` 확장을 사용하여 Azure AI 언어 서비스 호출을 성공적으로 수행하려면 확장에 해당 서비스의 엔드포인트와 키를 제공해야 합니다. Cloud Shell이 열려 있는 것과 같은 브라우저 탭을 사용하여 [Azure Portal](https://portal.azure.com/)의 언어 서비스 리소스로 이동하고 왼쪽 탐색 메뉴의 **리소스 관리**에서 **키 및 엔드포인트** 항목을 선택합니다.

    ![Azure 언어 서비스의 키 및 엔드포인트 페이지가 표시되어 있고 키 1 및 엔드포인트 복사 버튼이 빨간색 상자로 강조 표시된 스크린샷.](media/15-azure-language-service-keys-endpoints.png)

    > [!Note]
    >
    > 위의 `azure_ai` 확장을 설치할 때 `NOTICE: extension "azure_ai" already exists, skipping CREATE EXTENSION` 메시지가 표시되었고 이전에 언어 서비스 엔드포인트 및 키를 사용하여 확장을 구성했다면 `azure_ai.get_setting()` 함수를 사용하여 해당 설정이 올바른지 확인한 다음 올바른 경우 2단계를 건너뛸 수 있습니다.

2. 엔드포인트 및 액세스 키 값을 복사한 다음, 아래 명령에서 `{endpoint}` 및 `{api-key}` 토큰을 Azure Portal에서 복사한 값으로 바꿉니다. Cloud Shell의 `psql` 명령 프롬프트에서 명령을 실행하여 `azure_ai.settings` 테이블에 값을 추가합니다.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## 확장의 요약 기능 검토

이 작업에서는 `azure_cognitive` 스키마의 두 요약 함수를 검토합니다.

1. 이 연습의 이후 부분에서는 Cloud Shell에서만 작업하므로 창의 오른쪽 위에 있는 **최대화** 버튼을 선택하여 브라우저 창 내에서 해당 창을 확장하면 도움이 될 수 있습니다.

    ![빨간색 상자로 강조 표시된 최대화 버튼이 있는 Azure Cloud Shell 창의 스크린샷.](media/15-azure-cloud-shell-pane-maximize.png)

2. Cloud Shell에서 `psql` 작업 시 쿼리 결과에 확장된 디스플레이를 사용하도록 설정하면 후속 명령에 대한 출력의 가독성이 향상되어 유용할 수 있습니다. 확장된 디스플레이를 자동으로 적용할 수 있도록 허용하려면 다음 명령을 실행합니다.

    ```sql
    \x auto
    ```

3. `azure_ai` 확장의 텍스트 요약 함수는 `azure_cognitive` 스키마 내에서 찾을 수 있습니다. 추출 요약의 경우 `summarize_extractive()` 함수를 사용합니다. [`\df` 메타 명령](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) 을 사용하면 다음을 실행하여 함수를 검사합니다.

    ```sql
    \df azure_cognitive.summarize_extractive
    ```

    메타 명령 출력에는 함수의 스키마, 이름, 결과 데이터 형식, 인수가 표시됩니다. 이 정보는 쿼리에서 함수와 상호 작용하는 방법을 이해하는 데 도움이 됩니다.

    출력에 `summarize_extractive()` 함수의 세 가지 오버로드가 표시되어 차이점을 검토할 수 있습니다. 출력의 `Argument data types` 속성은 세 가지 함수 오버로드에 필요한 인수 목록을 표시합니다.

    | 인수 | Type | 기본값 | 설명 |
    | -------- | ---- | ------- | ----------- |
    | text | `text` 또는 `text[]` || 요약을 생성해야 하는 텍스트입니다. |
    | language_text | `text` 또는 `text[]` || 요약할 텍스트의 언어를 나타내는 언어 코드(또는 언어 코드 배열)입니다. 필요한 언어 코드를 검색하려면 [지원되는 언어 목록](https://learn.microsoft.com/azure/ai-services/language-service/summarization/language-support)을 검토합니다. |
    | sentence_count | `integer` | 3 | 생성할 요약 문장의 수입니다. |
    | sort_by | `text` | 'offset' | 생성된 요약 문장의 정렬 순서입니다. 허용되는 값은 "오프셋"과 "순위"입니다. 오프셋은 원본 콘텐츠 내에서 추출된 각 문장의 시작 위치를 나타내며 순위는 문장이 콘텐츠의 주요 아이디어와 얼마나 관련성이 있는지에 대한 AI 생성 표시기입니다. |
    | batch_size | `integer` | 25 | `text[]` 입력이 예상되는 두 가지 오버로드에만 해당됩니다. 한 번에 처리할 레코드 수를 지정합니다. |
    | disable_service_logs | `boolean` | false | 서비스 로그를 비활성화 여부를 나타내는 플래그입니다. |
    | timeout_ms | `integer` | NULL | 작업이 중지된 후의 시간 제한(밀리초)입니다. |
    | throw_on_error | `boolean` | true | 함수가 오류 발생 시 예외를 throw하여 래핑 트랜잭션을 롤백해야 하는지 여부를 나타내는 플래그입니다. |
    | max_attempts | `integer` | 1 | 오류가 발생한 경우 Azure AI 서비스에 대한 호출을 다시 시도하는 횟수입니다. |
    | retry_delay_ms | `integer` | 1000 | Azure AI 서비스 엔드포인트 호출을 다시 시도하기 전에 대기하는 시간(밀리초)입니다. |

4. 위의 단계를 반복하되 이번에는 `azure_cognitive.summarize_abstractive()` 함수에 대해 [`\df` 메타 명령어](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC)를 실행하고 출력을 검토합니다.

    두 함수는 유사한 시그니처를 가지고 있지만, `summarize_abstractive()`에는 `sort_by` 매개변수가 없으며, `summarize_extractive()` 함수가 반환하는 `azure_cognitive.sentence` 복합 형식 배열과는 달리 `text` 배열을 반환합니다. 이러한 차이는 서로 다른 두 메서드가 요약을 생성하는 방식과 관련이 있습니다. 추출 요약은 요약하는 텍스트 내에서 가장 중요한 문장을 식별하고 순위를 지정한 다음 해당 문장을 요약으로 반환합니다. 반면 추상 요약은 생성형 AI를 사용하여 텍스트의 핵심 요소를 요약하는 새로운 오리지널 문장을 만듭니다.

5. 또한 쿼리의 출력을 올바르게 처리할 수 있도록 함수가 반환하는 데이터 형식의 구조를 이해해야 합니다. `summarize_extractive()` 함수가 반환하는 `azure_cognitive.sentence` 유형을 검사하려면 다음을 실행합니다.

    ```sql
    \dT+ azure_cognitive.sentence
    ```

6. 위 명령의 출력은 `sentence` 유형이 `tuple`임을 나타냅니다. 해당 `tuple`의 구조를 살펴보고 `sentence` 복합 형식에 포함된 열을 검토하려면 다음을 실행합니다.

    ```sql
    \d+ azure_cognitive.sentence
    ```

    이 명령의 출력은 다음과 유사합니다.

    ```sql
                            Composite type "azure_cognitive.sentence"
        Column  |     Type         | Collation | Nullable | Default | Storage  | Description 
    ------------+------------------+-----------+----------+---------+----------+-------------
     text       | text             |           |           |        | extended | 
     rank_score | double precision |           |           |        | plain    |
    ```

    `azure_cognitive.sentence`은(는) 추출 문장의 텍스트와 각 문장에 대한 순위 점수를 포함하는 복합 형식으로, 해당 문장이 텍스트의 주요 주제와 얼마나 관련이 있는지를 나타냅니다. 문서 요약은 추출된 문장의 순위를 매기며, 문장이 나타나는 순서대로 반환할지 또는 순위에 따라 반환할지 결정할 수 있습니다.

## 속성 설명에 대한 요약 만들기

이 작업에서는 `summarize_extractive()` 및 `summarize_abstractive()` 함수를 사용하여 속성 설명에 대한 간결한 두 문장 요약을 작성합니다.

1. 이제 `summarize_extractive()` 함수와 이 함수가 반환하는 `sentiment_analysis_result`에 대해 살펴봤으니 함수를 사용해 보겠습니다. 다음의 간단한 쿼리를 실행하여 `reviews` 테이블에 있는 몇 개의 댓글에 대해 감정 분석을 수행합니다.

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    출력의 `extractive_summary` 필드에 있는 두 문장을 원본 `description`와(과) 비교하고, 이 문장이 원본이 아니라 `description`에서 추출한 것임을 확인합니다. 각 문장 이후에 나열된 숫자 값은 언어 서비스에서 할당한 순위 점수입니다.

2. 그런 다음 동일한 레코드에 대해 추상 요약을 수행합니다.

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    확장의 추상 요약 기능은 원래 텍스트의 전체 의도를 캡슐화하는 고유한 자연어 요약을 제공합니다.

    다음과 유사한 오류가 표시되는 경우 Azure 환경을 만들 때 추상 요약을 지원하지 않는 지역을 선택했습니다.

    ```bash
    ERROR: azure_cognitive.summarize_abstractive: InvalidRequest: Invalid Request.

    InvalidParameterValue: Job task: 'AbstractiveSummarization-task' failed with validation errors: ['Invalid Request.']

    InvalidRequest: Job task: 'AbstractiveSummarization-task' failed with validation error: Document abstractive summarization is not supported in the region Central US. The supported regions are North Europe, East US, West US, UK South, Southeast Asia.
    ```

    추상 요약을 사용하여 이 단계를 수행하고 나머지 작업을 완료하려면 오류 메시지에 지정된 지원되는 지역 중 하나에 새 Azure AI 언어 서비스를 만들어야 합니다. 이 서비스는 다른 랩 리소스에 사용한 것과 동일한 리소스 그룹에서 프로비전할 수 있습니다. 또는 나머지 작업에 대해 추출 요약을 대체할 수 있지만 두 가지 요약 기술의 출력을 비교할 수 있다는 이점이 없습니다.

3. 마지막 쿼리를 실행하여 두 요약 기술을 나란히 비교합니다.

    ```sql
    SELECT
        id,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    생성된 요약을 나란히 배치하면 각 메서드에서 생성된 요약의 품질을 쉽게 비교할 수 있습니다. Margie's Travel 애플리케이션의 경우 추상 요약이 더 나은 옵션이며, 자연스럽고 읽기 쉬운 방식으로 고품질 정보를 제공하는 간결한 요약을 제공합니다. 추출 요약은 일부 세부 정보를 제공하지만 추상 요약으로 만든 원본 콘텐츠보다 단절되어 있고 가치가 떨어집니다.

## 데이터베이스에 설명 요약 저장

1. 다음 쿼리를 실행하여 `listings` 테이블을 변경하고 새 `summary` 열을 추가합니다.

    ```sql
    ALTER TABLE listings
    ADD COLUMN summary text;
    ```

2. 생성형 AI를 사용하여 데이터베이스의 모든 기존 속성에 대한 요약을 만들려면 언어 서비스가 여러 레코드를 동시에 처리할 수 있도록 설명을 일괄 처리로 보내는 것이 가장 효율적입니다.

    ```sql
    WITH batch_cte AS (
        SELECT azure_cognitive.summarize_abstractive(ARRAY(SELECT description FROM listings ORDER BY id), 'en', batch_size => 25) AS summary
    ),
    summary_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            ARRAY_TO_STRING(summary, ',') AS summary
        FROM batch_cte
    )
    UPDATE listings AS l
    SET summary = s.summary
    FROM summary_cte AS s
    WHERE l.id = s.id;
    ```

    업데이트 문은 요약이 포함된 `listings` 테이블을 업데이트하기 전에 두 개의 CTE(공통 테이블 식)를 사용하여 데이터에 대한 작업을 수행합니다. 첫 번째 CTE(`batch_cte`)는 `listings` 테이블의 모든 `description` 값을 언어 서비스로 전송하여 추상 요약을 생성합니다. 이 작업은 한 번에 25개의 레코드를 일괄 처리합니다. 두 번째 CTE(`summary_cte`)는 `summarize_abstractive()` 함수가 반환한 요약의 서수 위치를 사용하여 각 요약에 `listings` 테이블에서 `description`이(가) 속한 레코드에 해당하는 `id`을 할당합니다. 또한 `ARRAY_TO_STRING` 함수를 사용하여 텍스트 배열(`text[]`) 반환값에서 생성된 요약을 끌어와 간단한 문자열로 변환합니다. 마지막으로 `UPDATE` 문은 관련 목록에 대한 요약을 `listings` 테이블에 씁니다.

3. 마지막 단계로 쿼리를 실행하여 `listings` 테이블에 기록된 요약을 봅니다.

    ```sql
    SELECT
        id,
        name,
        description,
        summary
    FROM listings
    LIMIT 5;
    ```

## 목록에 대한 리뷰의 AI 요약 생성

Margie's Travel 앱에서는 숙소에 대한 모든 리뷰의 요약을 표시함으로써 사용자가 리뷰의 전반적인 내용을 빠르게 파악할 수 있습니다.

1. 다음 쿼리를 실행하면 목록에 대한 모든 리뷰를 단일 문자열로 결합한 다음 해당 문자열에 대한 추상 요약을 생성합니다.

    ```sql
    SELECT unnest(azure_cognitive.summarize_abstractive(reviews_combined, 'en')) AS review_summary
    FROM (
        -- Combine all reviews for a listing
        SELECT string_agg(comments, ' ') AS reviews_combined
        FROM reviews
        WHERE listing_id = 1
    );
    ```

## 정리

이 연습을 완료한 후에는 만든 Azure 리소스를 삭제합니다. 요금은 데이터베이스 사용량이 아니라 구성된 용량에 대해 부과됩니다. 다음 지침에 따라 리소스 그룹과 이 랩을 위해 만든 모든 리소스를 삭제합니다.

1. 웹 브라우저를 열어 [Azure Portal](https://portal.azure.com/)로 이동한 뒤 홈페이지의 Azure 서비스에서 **리소스 그룹**을 선택합니다.

    ![Azure Portal의 Azure 서비스 아래 리소스 그룹이 빨간색 상자로 강조 표시된 스크린샷.](media/15-azure-portal-home-azure-services-resource-groups.png)

2. 원하는 필드 검색 상자의 필터에 이 랩을 위해 만든 리소스 그룹의 이름을 입력한 다음 목록에서 이 리소스 그룹을 선택합니다.

3. 리소스 그룹의 **개요** 페이지에서 **리소스 그룹 삭제**를 선택합니다.

    ![리소스 그룹 삭제 버튼이 빨간색 상자로 강조 표시된 리소스 그룹의 개요 블레이드 스크린샷.](media/15-resource-group-delete.png)

4. 확인 대화 상자에서 삭제할 리소스 그룹의 이름을 입력하여 확인하고 **삭제**를 선택합니다.

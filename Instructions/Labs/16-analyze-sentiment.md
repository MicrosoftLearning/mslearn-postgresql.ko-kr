---
lab:
  title: 감정 분석
  module: Perform Sentiment Analysis and Opinion Mining using Azure Database for PostgreSQL
---

# 감정 분석

Margie's Travel을 위해 빌드하는 AI 기반 앱의 일환으로 사용자에게 개별 리뷰의 감정과 지정된 임대 부동산에 대한 모든 리뷰의 전반적인 감정에 대한 정보를 제공하고자 합니다. 이를 위해 Azure Database for PostgreSQL 유연한 서버에서 `azure_ai` 확장을 사용하여 감정 분석 기능을 데이터베이스에 통합합니다.

## 시작하기 전에

관리 권한이 있는 [Azure 구독](https://azure.microsoft.com/free)이 필요합니다.

### Azure 구독에서 리소스 배포

이 단계에서는 Azure Cloud Shell에서 Azure CLI 명령을 사용하여 리소스 그룹을 만들고 Bicep 스크립트를 실행하여 이 연습을 완료하는 데 필요한 Azure 서비스를 Azure 구독에 배포하는 방법을 안내합니다.

1. 웹 브라우저를 열고 [Azure Portal](https://portal.azure.com/)로 이동합니다.

2. Azure Portal 도구 모음의 **Cloud Shell** 아이콘을 선택하여 브라우저 창 아래쪽에 새 [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 창을 엽니다.

    ![Cloud Shell 아이콘을 빨간색 상자로 강조 표시한 Azure 도구 모음을 보여주는 스크린샷.](media/11-portal-toolbar-cloud-shell.png)

    메시지가 표시되면 *Bash* 셸을 여는 데 필요한 옵션을 선택합니다. 이전에 *PowerShell* 콘솔을 사용한 경우 *Bash* 셸로 전환합니다.

3. Cloud Shell 프롬프트에서 다음을 입력하여 연습 리소스가 포함된 GitHub 리포지토리를 복제합니다.

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. 다음으로, Azure CLI 명령을 사용하여 Azure 리소스를 만들 때 중복 입력을 줄이기 위해 변수를 정의하는 세 가지 명령을 실행합니다. 이 변수는 리소스 그룹에 할당할 이름(`RG_NAME`), 리소스를 배포할 Azure 지역(`REGION`), PostgreSQL 관리자 로그인용 임의 생성 암호(`ADMIN_PASSWORD`)를 나타냅니다.

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

5. 두 개 이상의 Azure 구독에 액세스할 수 있고 기본 구독이 이 연습에 대한 리소스 그룹 및 기타 리소스를 만드는 데 사용할 구독이 아닌 경우, 이 명령을 실행하여 적절한 구독을 설정하고 `<subscriptionName|subscriptionId>` 토큰을 사용할 구독의 이름 또는 ID로 바꿉니다.

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

    Bicep 배포 스크립트는 이 연습을 완료하는 데 필요한 Azure 서비스를 리소스 그룹으로 프로비전합니다. 배포된 리소스에는 Azure Database for PostgreSQL 유연한 서버, Azure OpenAI, Azure AI 언어 서비스가 포함됩니다. 또한 Bicep 스크립트는 azure.extensions 서버 매개 변수를 통해 PostgreSQL 서버의 _허용 목록_에 `azure_ai` 및 `vector` 확장을 추가하고, 서버에 `rentals` 데이터베이스를 만들고, Azure OpenAI Service에 `text-embedding-ada-002` 모델을 사용한 `embedding` 배포를 추가하는 등 몇 가지 구성 단계를 수행합니다. 이 Bicep 파일은 이 학습 경로의 모든 모듈에서 공유되므로 일부 연습에서는 배포된 리소스 중 일부만 사용할 수 있다는 점에 유의해야 합니다.

    배포를 완료하려면 보통 몇 분 정도 걸립니다. Cloud Shell에서 모니터링하거나 위에서 만든 리소스 그룹에 대한 **배포** 페이지로 이동하여 배포 진행 상황을 관찰할 수 있습니다.

8. 리소스 배포가 완료되면 Cloud Shell 창을 닫습니다.

### 배포 오류 문제 해결

Bicep 배포 스크립트를 실행할 때 몇 가지 오류가 발생할 수 있습니다. 가장 일반적인 메시지와 해결 단계는 다음과 같습니다.

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

이 작업에서는 [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)의 [psql 명령줄 유틸리티](https://www.postgresql.org/docs/current/app-psql.html)를 사용하여 Azure Database for PostgreSQL 서버의 `rentals` 데이터베이스에 연결합니다.

1. [Azure Portal](https://portal.azure.com/)에서 새로 만든 Azure Database for PostgreSQL 유연한 서버 인스턴스로 이동합니다.

2. 리소스 메뉴의 **설정**에서 **데이터베이스**를 선택하고 `rentals` 데이터베이스에 대한 **연결**을 선택합니다. **연결**을 선택해도 실제로 데이터베이스에 연결되지는 않습니다. 다양한 방법을 사용하여 데이터베이스에 연결하기 위한 지침만 제공합니다. **브라우저에서 또는 로컬로 연결** 지침을 검토하고 해당 지침을 사용하여 Azure Cloud Shell을 사용하여 연결합니다.

    ![Azure Database for PostgreSQL 데이터베이스 페이지의 스크린샷. 임대 데이터베이스에 대한 데이터베이스 및 연결이 빨간색 상자로 강조 표시됩니다.](media/17-postgresql-rentals-database-connect.png)

3. Cloud Shell의 "사용자 pgAdmin에 대한 암호" 프롬프트에서 **pgAdmin** 로그인에 대해 임의로 생성된 암호를 입력합니다.

    로그인하면 `rentals` 데이터베이스에 대한 `psql` 프롬프트가 표시됩니다.

## 샘플 데이터를 이용해 데이터베이스 작성

`azure_ai` 확장을 사용하여 임대 부동산 리뷰의 감정을 분석하려면 먼저 데이터베이스에 샘플 데이터를 추가해야 합니다. `rentals` 데이터베이스에 테이블을 추가하고 고객 리뷰를 채워 감정 분석을 수행할 데이터를 준비합니다.

1. 다음 명령을 실행하여 고객이 제출한 부동산 리뷰를 저장할 `reviews` 테이블을 만듭니다.

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. 다음으로, `COPY` 명령을 사용하여 테이블에 CSV 파일의 데이터를 채웁니다. 아래 명령을 실행하여 고객 리뷰를 `reviews` 테이블에 로드합니다.

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    명령의 출력은 `COPY 354`여야 합니다. 이는 CSV 파일의 테이블에 354개의 행이 기록되었음을 나타냅니다.

## `azure_ai` 확장 설치 및 구성

`azure_ai` 확장을 사용하기 전에는 확장을 데이터베이스에 설치하고 Azure AI Services 리소스에 연결하도록 구성해야 합니다. `azure_ai` 확장을 사용하면 Azure OpenAI 및 Azure AI 언어 서비스를 데이터베이스에 통합할 수 있습니다. 데이터베이스에서 확장을 사용하도록 설정하려면 다음 단계를 수행합니다.

1. `psql` 프롬프트에서 다음 명령을 실행하여, 환경을 설정할 때 실행한 Bicep 배포 스크립트를 통해 `azure_ai` 및 `vector` 확장이 서버의 _허용 목록_에 성공적으로 추가되었는지 확인합니다.

    ```sql
    SHOW azure.extensions;
    ```

    이 명령은 서버의 _허용 목록_에 확장 목록을 표시합니다. 모든 항목이 올바르게 설치된 경우 다음과 같이 출력에 `azure_ai` 및 `vector`가 포함되어 있어야 합니다.

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Azure Database for PostgreSQL 유연한 서버 데이터베이스에서 확장을 설치 및 사용하기 위해서는 [PostgreSQL 확장 사용 방법](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions)에 설명된 대로 서버의 _허용 목록_에 확장을 반드시 추가해야 합니다.

2. 이제 [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html) 명령을 사용하여 `azure_ai` 확장을 설치할 수 있습니다.

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` 명령은 스크립트 파일을 실행하여 데이터베이스에 새 확장을 로드합니다. 이 스크립트는 일반적으로 함수, 데이터 형식 및 스키마와 같은 새 SQL 개체를 만듭니다. 동일한 이름의 확장이 이미 있는 경우 오류가 표시됩니다. `IF NOT EXISTS` 문을 추가하면 확장이 이미 설치된 경우 오류를 표시하지 않고 명령을 실행할 수 있습니다.

## Azure AI 서비스 계정 연결

`azure_ai` 확장의 `azure_cognitive` 스키마에 포함된 Azure AI 서비스 통합은 데이터베이스에서 직접 액세스할 수 있는 다양한 AI 언어 기능 세트를 제공합니다. 감정 분석 기능은 [Azure AI 언어 서비스](https://learn.microsoft.com/azure/ai-services/language-service/overview)를 통해 지원됩니다.

1. `azure_ai` 확장을 사용하여 Azure AI 언어 서비스 호출을 성공적으로 수행하려면 확장에 해당 서비스의 엔드포인트와 키를 제공해야 합니다. Cloud Shell이 열려 있는 것과 같은 브라우저 탭을 사용하여 [Azure Portal](https://portal.azure.com/)의 언어 서비스 리소스로 이동하고 왼쪽 탐색 메뉴의 **리소스 관리**에서 **키 및 엔드포인트** 항목을 선택합니다.

    ![Azure 언어 서비스의 키 및 엔드포인트 페이지가 표시되어 있고 키 1 및 엔드포인트 복사 버튼이 빨간색 상자로 강조 표시된 스크린샷.](media/16-azure-language-service-keys-endpoints.png)

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

## 확장의 감정 분석 기능 검토

이 작업에서는 `azure_cognitive.analyze_sentiment()` 함수를 사용하여 임대 부동산 목록의 리뷰를 평가합니다.

1. 이 연습의 이후 부분에서는 Cloud Shell에서만 작업하므로 창의 오른쪽 위에 있는 **최대화** 버튼을 선택하여 브라우저 창 내에서 해당 창을 확장하면 도움이 될 수 있습니다.

    ![빨간색 상자로 강조 표시된 최대화 버튼이 있는 Azure Cloud Shell 창의 스크린샷.](media/16-azure-cloud-shell-pane-maximize.png)

2. Cloud Shell에서 `psql` 작업 시 쿼리 결과에 확장된 디스플레이를 사용하도록 설정하면 후속 명령에 대한 출력의 가독성이 향상되어 유용할 수 있습니다. 확장된 디스플레이를 자동으로 적용할 수 있도록 허용하려면 다음 명령을 실행합니다.

    ```sql
    \x auto
    ```

3. `azure_ai` 확장의 감정 분석 기능은 `azure_cognitive` 스키마 내에서 찾을 수 있습니다. `analyze_sentiment()` 함수를 사용합니다. [`\df` 메타 명령](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC)을 사용하면 다음을 실행하여 함수를 검사합니다.

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    메타 명령 출력에는 함수의 스키마, 이름, 결과 데이터 형식, 인수가 표시됩니다. 이 정보는 쿼리에서 함수와 상호 작용하는 방법을 이해하는 데 도움이 됩니다.

    출력에 `analyze_sentiment()` 함수의 세 가지 오버로드가 표시되어 차이점을 검토할 수 있습니다. 출력의 `Argument data types` 속성은 세 가지 함수 오버로드에 필요한 인수 목록을 표시합니다.

    | 인수 | Type | 기본값 | 설명 |
    | -------- | ---- | ------- | ----------- |
    | text | `text` 또는 `text[]` || 감정을 분석할 텍스트입니다. |
    | language_text | `text` 또는 `text[]` || 감정을 분석할 텍스트의 언어를 나타내는 언어 코드(또는 언어 코드 배열)입니다. 필요한 언어 코드를 검색하려면 [지원되는 언어 목록](https://learn.microsoft.com/azure/ai-services/language-service/sentiment-opinion-mining/language-support)을 검토합니다. |
    | batch_size | `integer` | 10 | `text[]` 입력이 예상되는 두 가지 오버로드에만 해당됩니다. 한 번에 처리할 레코드 수를 지정합니다. |
    | disable_service_logs | `boolean` | false | 서비스 로그를 비활성화 여부를 나타내는 플래그입니다. |
    | timeout_ms | `integer` | NULL | 작업이 중지된 후의 시간 제한(밀리초)입니다. |
    | throw_on_error | `boolean` | true | 함수가 오류 발생 시 예외를 throw하여 래핑 트랜잭션을 롤백해야 하는지 여부를 나타내는 플래그입니다. |
    | max_attempts | `integer` | 1 | 오류가 발생한 경우 Azure AI 서비스에 대한 호출을 다시 시도하는 횟수입니다. |
    | retry_delay_ms | `integer` | 1000 | Azure AI 서비스 엔드포인트 호출을 다시 시도하기 전에 대기하는 시간(밀리초)입니다. |

4. 또한 쿼리의 출력을 올바르게 처리할 수 있도록 함수가 반환하는 데이터 형식의 구조를 이해하는 것이 중요합니다. `sentiment_analysis_result` 형식을 검사하기 위해 다음 명령을 실행합니다.

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. 위 명령의 출력은 `sentiment_analysis_result` 형식이 `tuple`임을 나타냅니다. 다음 명령을 실행하여 `sentiment_analysis_result` 형식 내에 포함된 열을 확인하는 방법으로 해당 `tuple`의 구조를 자세히 살펴볼 수 있습니다.

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    이 명령의 출력은 다음과 유사합니다.

    ```sql
                     Composite type "azure_cognitive.sentiment_analysis_result"
         Column     |     Type         | Collation | Nullable | Default | Storage  | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment      | text             |           |          |         | extended | 
     positive_score | double precision |           |          |         | plain    | 
     neutral_score  | double precision |           |          |         | plain    | 
     negative_score | double precision |           |          |         | plain    |
    ```

    `azure_cognitive.sentiment_analysis_result`의 경우 복합 형식으로, 입력 텍스트의 감정 예측이 포함되어 있습니다. 여기에는 긍정적, 부정적, 중립적, 혼합적일 수 있는 감정과 텍스트에 있는 긍정적, 중립적, 부정적 양상에 대한 점수가 포함됩니다. 점수는 0과 1 사이의 실수로 표시됩니다. 예를 들어,(중립적, 0.26, 0.64, 0.09)에서는 감정이 중립적이며 긍정적인 점수는 0.26, 중립적인 점수는 0.64, 부정적인 점수는 0.09입니다.

## 리뷰의 감정 분석

1. 이제 `analyze_sentiment()` 함수와 여기에서 반환되는 `sentiment_analysis_result` 결과를 검토했으니 사용할 함수를 배치해 보겠습니다. 다음의 간단한 쿼리를 실행하여 `reviews` 테이블에 있는 몇 개의 댓글에 대해 감정 분석을 수행합니다.

    ```sql
    SELECT
        id,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id <= 10
    ORDER BY id;
    ```

    분석된 두 레코드에서 출력의 `sentiment` 값인 `(mixed,0.71,0.09,0.2)` 및 `(positive,0.99,0.01,0)` 값을 확인합니다. 이는 위 쿼리의 `analyze_sentiment()` 함수에서 반환된 `sentiment_analysis_result` 값을 나타냅니다. 이 분석은 `reviews` 테이블의 `comments` 필드에 대해 수행되었습니다.

    > [!Note]
    >
    > `analyze_sentiment()` 함수 인라인을 사용하면 쿼리 내에서 텍스트의 감정을 빠르게 분석할 수 있습니다. 이는 적은 수의 레코드에서는 잘 작동하지만, 대량의 레코드에서 감정을 분석하거나 수만 개 이상의 리뷰가 포함될 수 있는 테이블의 모든 레코드를 업데이트하는 데에는 적합하지 않을 수 있습니다.

2. 보다 긴 리뷰에 유용할 수 있는 또 다른 방법은 그 안에 있는 각 문장별로 감정을 분석하는 것입니다. 이렇게 하려면 텍스트 배열을 지원하는 `analyze_sentiment()` 함수의 오버로드를 사용합니다.

    ```sql
    SELECT
        azure_cognitive.analyze_sentiment(ARRAY_REMOVE(STRING_TO_ARRAY(comments, '.'), ''), 'en') AS sentence_sentiments
    FROM reviews
    WHERE id = 1;
    ```

    위의 쿼리에서는 PostgreSQL의 `STRING_TO_ARRAY` 함수를 사용했습니다. 또한 `ARRAY_REMOVE` 함수를 사용하여 빈 문자열인 배열 요소를 제거했습니다. 제거하지 않을 경우 `analyze_sentiment()` 함수에 오류가 발생합니다.

    쿼리의 출력을 통해 전반적인 검토에 할당된 `mixed` 감정을 보다 잘 이해할 수 있습니다. 문장은 긍정적, 중립적, 부정적인 감정의 혼합물입니다.

3. 이전 두 쿼리의 경우 쿼리에서 바로 `sentiment_analysis_result` 결과를 반환했습니다. 그러나 사용자에게는 `sentiment_analysis_result``tuple` 내에서 기본 값을 검색하는 작업이 필요할 가능성이 높습니다. 압도적으로 긍정적인 리뷰를 검색하고 감정 구성 요소를 개별 필드로 추출하는 다음 쿼리를 실행합니다.

    ```sql
    WITH cte AS (
        SELECT id, comments, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    SELECT
        id,
        (sentiment).sentiment,
        (sentiment).positive_score,
        (sentiment).neutral_score,
        (sentiment).negative_score,
        comments
    FROM cte
    WHERE (sentiment).positive_score > 0.98
    LIMIT 5;
    ```

    위의 쿼리는 공통 테이블 식 또는 CTE를 사용하여 `reviews` 테이블의 모든 레코드에 대한 감정 점수를 가져옵니다. 그런 다음 CTE에서 반환된 `sentiment_analysis_result`의 `sentiment` 복합 형식 열을 선택하여 `tuple.`의 개별 값을 추출합니다.

## Reviews 테이블에 감정 저장

이번에는 Margie's Travel을 위해 빌드하는 임대 부동산 추천 시스템에서 감정 평가가 필요할 때마다 호출을 보내고 비용이 발생하는 상황을 방지하기 위해 데이터베이스에 감정 등급을 저장하려고 합니다. 즉시 감정 분석을 수행하는 것은 적은 수의 레코드를 사용하거나 거의 실시간으로 데이터를 분석하는 데 적합할 수 있습니다. 하지만 애플리케이션에서 사용하기 위해 감정 데이터를 데이터베이스에 추가하는 것은 저장된 리뷰에 유용합니다. 이렇게 하려면 `reviews` 테이블을 변경하여 감정 평가 및 긍정, 중립, 부정 점수를 저장하기 위한 열을 추가하면 됩니다.

1. 다음 쿼리를 실행하여 `reviews` 테이블이 감정 세부 정보를 저장할 수 있도록 업데이트합니다.

    ```sql
    ALTER TABLE reviews
    ADD COLUMN sentiment varchar(10),
    ADD COLUMN positive_score numeric,
    ADD COLUMN neutral_score numeric,
    ADD COLUMN negative_score numeric;
    ```

2. 다음으로, `reviews` 테이블의 기존 레코드를 감정값 및 관련 점수로 업데이트해 보겠습니다.

    ```sql
    WITH cte AS (
        SELECT id, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    UPDATE reviews AS r
    SET
        sentiment = (cte.sentiment).sentiment,
        positive_score = (cte.sentiment).positive_score,
        neutral_score = (cte.sentiment).neutral_score,
        negative_score = (cte.sentiment).negative_score
    FROM cte
    WHERE r.id = cte.id;
    ```

    테이블의 모든 리뷰에 있는 댓글이 분석을 위해 언어 서비스의 엔드포인트로 개별적으로 전송되므로 이 쿼리를 실행하는 데에는 시간이 오래 걸립니다. 많은 레코드를 처리할 때는 레코드를 일괄 처리 단위로 보내는 것이 더 효율적입니다.

3. 아래 쿼리를 실행하여 동일한 업데이트 작업을 수행해 보겠습니다. 하지만 이번에는 `reviews` 테이블의 댓글을 10개씩 일괄 처리(허용되는 최대 일괄 처리 크기)하도록 보내고 성능 차이를 평가합니다.

    ```sql
    WITH cte AS (
        SELECT azure_cognitive.analyze_sentiment(ARRAY(SELECT comments FROM reviews ORDER BY id), 'en', batch_size => 10) as sentiments
    ),
    sentiment_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            sentiments AS sentiment
        FROM cte
    )
    UPDATE reviews AS r
    SET
        sentiment = (sentiment_cte.sentiment).sentiment,
        positive_score = (sentiment_cte.sentiment).positive_score,
        neutral_score = (sentiment_cte.sentiment).neutral_score,
        negative_score = (sentiment_cte.sentiment).negative_score
    FROM sentiment_cte
    WHERE r.id = sentiment_cte.id;
    ```

    이 쿼리는 좀 더 복잡하지만 두 개의 CTE를 사용하므로 훨씬 성능이 향상됩니다. 이 쿼리에서 첫 번째 CTE는 일괄 처리하는 리뷰 댓글의 감정을 분석하고, 두 번째 CTE는 각 행별로 서수 위치 및 ``sentiment_analysis_result` 기반의 `id`을(를) 포함하는 새 테이블로 `sentiment_analysis_results`의 결과 테이블을 추출합니다. 그런 다음 update 문에서 두 번째 CTE를 사용하여 데이터베이스에 값을 쓸 수 있습니다.

4. 다음으로 쿼리를 실행하여 업데이트 결과를 관찰하고,가장 부정적인 감정부터 시작하여 **부정적인** 감정을 가진 리뷰를 검색합니다.

    ```sql
    SELECT
        id,
        negative_score,
        comments
    FROM reviews
    WHERE sentiment = 'negative'
    ORDER BY negative_score DESC;
    ```

## 정리

이 연습을 완료한 후에는 만든 Azure 리소스를 삭제합니다. 요금은 데이터베이스 사용량이 아니라 구성된 용량에 대해 부과됩니다. 다음 지침에 따라 리소스 그룹과 이 랩을 위해 만든 모든 리소스를 삭제합니다.

1. 웹 브라우저를 열어 [Azure Portal](https://portal.azure.com/)로 이동한 뒤 홈페이지의 Azure 서비스에서 **리소스 그룹**을 선택합니다.

    ![Azure Portal의 Azure 서비스 아래 리소스 그룹이 빨간색 상자로 강조 표시된 스크린샷.](media/16-azure-portal-home-azure-services-resource-groups.png)

2. 원하는 필드 검색 상자의 필터에 이 랩을 위해 만든 리소스 그룹의 이름을 입력한 다음 목록에서 이 리소스 그룹을 선택합니다.

3. 리소스 그룹의 **개요** 페이지에서 **리소스 그룹 삭제**를 선택합니다.

    ![리소스 그룹 삭제 버튼이 빨간색 상자로 강조 표시된 리소스 그룹의 개요 블레이드 스크린샷.](media/16-resource-group-delete.png)

4. 확인 대화 상자에서 삭제할 리소스 그룹의 이름을 입력하여 확인하고 **삭제**를 선택합니다.

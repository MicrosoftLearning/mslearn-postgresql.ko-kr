---
lab:
  title: Azure AI 언어를 사용한 인사이트 추출
  module: Extract insights using the Azure AI Language service with Azure Database for PostgreSQL
---

# Azure Database for PostgreSQL과 함께 Azure AI 언어 서비스를 사용하여 인사이트 추출

목록 회사는 가장 인기 있는 문구 또는 장소와 같은 시장 추세를 분석하고자 합니다. 또한 팀은 PII(개인 식별 정보)에 대한 보호를 강화하고자 합니다. 현재 데이터는 Azure Database for PostgreSQL 유연한 서버에 저장됩니다. 프로젝트 예산은 작기 때문에 키워드 및 태그를 유지 관리하는 데 초기 비용과 지속적인 비용을 최소화하는 것이 필수적입니다. 개발자들은 PII가 여러 형태로 나타날 수 있다는 점을 우려하며, 자체적인 정규식 검사기보다는 비용 효율적이고 검증된 솔루션을 선호합니다.

`azure_ai` 확장을 사용하여 Azure AI 언어 서비스와 데이터베이스를 통합합니다. 이 확장은 다음을 비롯한 여러 Azure Cognitive Service API에 사용자 정의 SQL 함수 API를 제공합니다.

- 핵심 구 추출
- 엔터티 인식
- PII 인식

이 방법을 사용하면 데이터 과학 팀이 목록 인기 데이터를 빠르게 조인하여 시장 추세를 결정할 수 있습니다. 이는 또한 응용 프로그램 개발자들에게 접근이 필요하지 않은 상황에서 표시할 수 있는 PII로부터 안전한 텍스트를 제공합니다. 식별된 엔터티를 저장하면 조회나 가양성 PII 인식(PII가 아닌 데이터를 PII로 인식하는 경우) 발생 시 사람이 검토할 수 있습니다.

결국 `listings` 테이블에 추출된 인사이트가 포함된 4개의 새 열이 생깁니다.

- `key_phrases`
- `recognized_entities`
- `pii_safe_description`
- `pii_entities`

## 시작하기 전에

관리 권한이 있는 [Azure 구독](https://azure.microsoft.com/free)이 필요합니다.

### Azure 구독에서 리소스 배포

이 단계에서는 Azure Cloud Shell에서 Azure CLI 명령을 사용하여 리소스 그룹을 만들고 Bicep 스크립트를 실행하여 이 연습을 완료하는 데 필요한 Azure 서비스를 Azure 구독에 배포하는 방법을 안내합니다.

1. 웹 브라우저를 열고 [Azure Portal](https://portal.azure.com/)로 이동합니다.

2. Azure Portal 도구 모음의 **Cloud Shell** 아이콘을 선택하여 브라우저 창 아래쪽에 새 [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 창을 엽니다.

    ![Cloud Shell 아이콘을 빨간색 상자로 강조 표시한 Azure 도구 모음을 보여주는 스크린샷.](media/17-portal-toolbar-cloud-shell.png)

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

    Bicep 배포 스크립트는 이 연습을 완료하는 데 필요한 Azure 서비스를 리소스 그룹으로 프로비전합니다. 배포 리소스에는 Azure Database for PostgreSQL - 유연한 서버, Azure OpenAI, Azure AI 언어 서비스가 포함됩니다. 또한 Bicep 스크립트는 `azure.extensions` 서버 매개 변수를 통해 PostgreSQL 서버의 _허용 목록_에 `azure_ai` 및 `vector` 확장을 추가하고, 서버에 `rentals` 데이터베이스를 만들고, Azure OpenAI Service에 `text-embedding-ada-002` 모델을 사용한 `embedding` 배포를 추가하는 등 몇 가지 구성 단계를 수행합니다. 이 Bicep 파일은 이 학습 경로의 모든 모듈에서 공유되므로 일부 연습에서는 배포된 리소스 중 일부만 사용할 수 있다는 점에 유의해야 합니다.

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

이 작업에서는 [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)의 [psql 명령줄 유틸리티](https://www.postgresql.org/docs/current/app-psql.html)를 사용하여 Azure Database for PostgreSQL 서버의 `rentals` 데이터베이스에 연결합니다.

1. [Azure Portal](https://portal.azure.com/)에서 새로 만든 Azure Database for PostgreSQL - 유연한 서버로 이동합니다.

2. 리소스 메뉴의 **설정**에서 **데이터베이스**를 선택하고 `rentals` 데이터베이스에 대한 **연결**을 선택합니다. **연결**을 선택해도 실제로 데이터베이스에 연결되지는 않습니다. 다양한 방법을 사용하여 데이터베이스에 연결하기 위한 지침만 제공합니다. **브라우저에서 또는 로컬로 연결** 지침을 검토하고 해당 지침을 사용하여 Azure Cloud Shell을 사용하여 연결합니다.

    ![Azure Database for PostgreSQL 데이터베이스 페이지의 스크린샷. 임대 데이터베이스에 대한 데이터베이스 및 연결이 빨간색 상자로 강조 표시됩니다.](media/17-postgresql-rentals-database-connect.png)

3. Cloud Shell의 "사용자 pgAdmin에 대한 암호" 프롬프트에서 **pgAdmin** 로그인에 대해 임의로 생성된 암호를 입력합니다.

    로그인하면 `rentals` 데이터베이스에 대한 `psql` 프롬프트가 표시됩니다.

4. 이 연습의 이후 부분에서는 계속 Cloud Shell에서 작업하므로 창의 오른쪽 위에 있는 **최대화** 버튼을 선택하여 브라우저 창 내에서 해당 창을 확장하면 도움이 될 수 있습니다.

    ![빨간색 상자로 강조 표시된 최대화 버튼이 있는 Azure Cloud Shell 창의 스크린샷.](media/17-azure-cloud-shell-pane-maximize.png)

## 설정: 확장 구성

벡터를 저장 및 쿼리하고 포함을 생성하려면 Azure Database for PostgreSQL 유연한 서버에서 `vector` 및 `azure_ai` 두 확장을 허용 목록에 추가하고 사용하도록 설정해야 합니다.

1. 두 확장을 모두 허용 목록에 추가하려면 [PostgreSQL 확장을 사용하는 방법](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions)의 지침에 따라 서버 매개 변수 `azure.extensions`에 `vector` 및 `azure_ai` 확장을 추가합니다.

2. `vector` 확장을 사용하도록 설정하기 위해 다음 SQL 명령을 실행합니다. 자세한 지침은 [Azure Database for PostgreSQL - 유연한 서버에서 `pgvector`를 사용하도록 설정하고 사용하는 방법](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension)을 참조하세요.

    ```sql
    CREATE EXTENSION vector;
    ```

3. `azure_ai` 확장을 사용하도록 설정하기 위해 다음 SQL 명령을 실행합니다. ***Azure OpenAI*** 리소스에 대한 엔드포인트 및 API 키가 필요합니다. 자세한 지침은 [`azure_ai` 확장을 사용하도록 설정](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension)을 참조하세요.

    ```sql
    CREATE EXTENSION azure_ai;
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

4. `azure_ai` 확장을 사용하여 ***Azure AI 언어*** 서비스 호출을 성공적으로 수행하려면 확장에 해당 서비스의 엔드포인트와 키를 제공해야 합니다. Cloud Shell이 열려 있는 것과 같은 브라우저 탭을 사용하여 [Azure Portal](https://portal.azure.com/)의 언어 서비스 리소스로 이동하고 왼쪽 탐색 메뉴의 **리소스 관리**에서 **키 및 엔드포인트** 항목을 선택합니다.

5. 엔드포인트 및 액세스 키 값을 복사한 다음, 아래 명령에서 `{endpoint}` 및 `{api-key}` 토큰을 Azure Portal에서 복사한 값으로 바꿉니다. Cloud Shell의 `psql` 명령 프롬프트에서 명령을 실행하여 `azure_ai.settings` 테이블에 값을 추가합니다.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## 샘플 데이터를 이용해 데이터베이스 작성.

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

샘플 데이터를 다시 설정하려면 `DROP TABLE listings` 실행 후 이 단계를 반복합니다.

## 핵심 구 추출

1. 핵심 구는 `pg_typeof` 함수에 의해 `text[]`(으)로 추출됩니다.

    ```sql
    SELECT pg_typeof(azure_cognitive.extract_key_phrases('The food was delicious and the staff were wonderful.', 'en-us'));
    ```

    키 결과를 포함하는 열을 만듭니다.

    ```sql
    ALTER TABLE listings ADD COLUMN key_phrases text[];
    ```

1. 열을 일괄 처리로 채웁니다. 할당량에 따라 `LIMIT` 값을 조정할 수 있습니다. *이 명령은 원하는 만큼 실행해도 괜찮습니다*. 이 연습에서 모든 행을 채울 필요는 없습니다.

    ```sql
    UPDATE listings
    SET key_phrases = azure_cognitive.extract_key_phrases(description)
    FROM (SELECT id FROM listings WHERE key_phrases IS NULL ORDER BY id LIMIT 100) subset
    WHERE listings.id = subset.id;
    ```

1. 핵심 구별 목록 쿼리:

    ```sql
    SELECT id, name FROM listings WHERE 'closet' = ANY(key_phrases);
    ```

    키 구문이 채워진 목록에 따라 아래와 같은 결과를 얻게 됩니다.

    ```sql
       id    |                name                
    ---------+-------------------------------------
      931154 | Met Tower in Belltown! MT2
      931758 | Hottest Downtown Address, Pool! MT2
     1084046 | Near Pike Place & Space Needle! MT2
     1084084 | The Best of the Best, Seattle! MT2
    ```

## 명명된 엔터티 인식

1. 엔터티는 `pg_typeof` 함수에 의해 표시된 대로 `azure_cognitive.entity[]`(으)로 추출됩니다.

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    키 결과를 포함하는 열을 만듭니다.

    ```sql
    ALTER TABLE listings ADD COLUMN entities azure_cognitive.entity[];
    ```

2. 열을 일괄 처리로 채웁니다. 이 과정에는 몇 분이 걸릴 수 있습니다. 할당량에 따라 `LIMIT` 값을 조정하거나 부분 결과를 더 빨리 반환할 수 있습니다. *이 명령은 원하는 만큼 실행해도 괜찮습니다*. 이 연습에서 모든 행을 채울 필요는 없습니다.

    ```sql
    UPDATE listings
    SET entities = azure_cognitive.recognize_entities(description, 'en-us')
    FROM (SELECT id FROM listings WHERE entities IS NULL ORDER BY id LIMIT 500) subset
    WHERE listings.id = subset.id;
    ```

3. 이제 모든 목록의 엔터티를 쿼리하여 지하실이 있는 숙소를 찾을 수 있습니다.

    ```sql
    SELECT id, name
    FROM listings, unnest(listings.entities) AS e
    WHERE e.text LIKE '%basements%'
    LIMIT 10;
    ```

    이때 반환되는 내용은 다음과 같습니다.

    ```sql
       id    |                name                
    ---------+-------------------------------------
      430610 | 3br/3ba. modern, roof deck, garage
      430610 | 3br/3ba. modern, roof deck, garage
     1214306 | Private Bed/bath in Home: green (A)
       74328 | Spacious Designer Condo
      938785 | Best Ocean Views By Pike Place! PA1
       23430 | 1 Bedroom Modern Water View Condo
      828298 | 2 Bedroom Sparkling City Oasis
      338043 | large modern unit & fab location
      872152 | Luxurious Local Lifestyle 2Bd/2+Bth
      116221 | Modern, Light-Filled Fremont Flat
    ```

## PII 인식

1. 엔터티는 `pg_typeof` 함수에 의해 표시된 대로 `azure_cognitive.pii_entity_recognition_result`(으)로 추출됩니다.

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_pii_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    이 값은 수정된 텍스트와 PII 엔터티 배열을 포함하는 복합 형식으로, 다음과 같이 확인됩니다.

    ```sql
    \d azure_cognitive.pii_entity_recognition_result
    ```

    출력되는 내용:

    ```sql
         Composite type "azure_cognitive.pii_entity_recognition_result"
         Column    |           Type           | Collation | Nullable | Default 
    ---------------+--------------------------+-----------+----------+---------
     redacted_text | text                     |           |          | 
     entities      | azure_cognitive.entity[] |           |          | 
    ```

    수정된 텍스트를 담을 열과 인식된 엔터티를 담을 또 다른 열을 생성합니다.

    ```sql
    ALTER TABLE listings ADD COLUMN description_pii_safe text;
    ALTER TABLE listings ADD COLUMN pii_entities azure_cognitive.entity[];
    ```

2. 열을 일괄 처리로 채웁니다. 이 과정에는 몇 분이 걸릴 수 있습니다. 할당량에 따라 `LIMIT` 값을 조정하거나 부분 결과를 더 빨리 반환할 수 있습니다. *이 명령은 원하는 만큼 실행해도 괜찮습니다*. 이 연습에서 모든 행을 채울 필요는 없습니다.

    ```sql
    UPDATE listings
    SET
        description_pii_safe = pii.redacted_text,
        pii_entities = pii.entities
    FROM (SELECT id, description FROM listings WHERE description_pii_safe IS NULL OR pii_entities IS NULL ORDER BY id LIMIT 100) subset,
    LATERAL azure_cognitive.recognize_pii_entities(subset.description, 'en-us') as pii
    WHERE listings.id = subset.id;
    ```

3. 이제 잠재적 PII가 수정된 목록 설명을 표시할 수 있습니다.

    ```sql
    SELECT description_pii_safe
    FROM listings
    WHERE description_pii_safe IS NOT NULL
    LIMIT 1;
    ```

    다음이 표시됩니다.

    ```sql
    A lovely stone-tiled room with kitchenette. New full mattress futon bed. Fridge, microwave, kettle for coffee and tea. Separate entrance into book-lined mudroom. Large bathroom with Jacuzzi (shared occasionally with ***** to do laundry). Stone-tiled, radiant heated floor, 300 sq ft room with 3 large windows. The bed is queen-sized futon and has a full-sized mattress with topper. Bedside tables and reading lights on both sides. Also large leather couch with cushions. Kitchenette is off the side wing of the main room and has a microwave, and fridge, and an electric kettle for making coffee or tea. Kitchen table with two chairs to use for meals or as desk. Extra high-speed WiFi is also provided. Access to English Garden. The Ballard Neighborhood is a great place to visit: *10 minute walk to downtown Ballard with fabulous bars and restaurants, great ****** farmers market, nice three-screen cinema, and much more. *5 minute walk to the Ballard Locks, where ships enter and exit Puget Sound
    ```

4. 위와 동일한 목록을 사용하여 PII에서 인식되는 엔터티를 식별할 수도 있습니다.

    ```sql
    SELECT pii_entities
    FROM listings
    WHERE entities IS NOT NULL
    LIMIT 1;
    ```

    다음이 표시됩니다.

    ```sql
                            pii_entities                        
    -------------------------------------------------------------
    {"(hosts,PersonType,\"\",0.93)","(Sunday,DateTime,Date,1)"}
    ```

## 작업 확인

추출된 핵심 구, 인식된 엔터티 및 PII가 채워졌는지 확인합니다.

1. 핵심 구 확인:

    ```sql
    SELECT COUNT(*) FROM listings WHERE key_phrases IS NOT NULL;
    ```

    실행한 일괄 처리 수에 따라 다음과 같이 표시됩니다.

    ```sql
    count 
    -------
     100
    ```

2. 인식된 엔터티 확인:

    ```sql
    SELECT COUNT(*) FROM listings WHERE entities IS NOT NULL;
    ```

    다음과 유사한 결과가 표시됩니다.

    ```sql
    count 
    -------
     500
    ```

3. 수정된 PII 확인:

    ```sql
    SELECT COUNT(*) FROM listings WHERE description_pii_safe IS NOT NULL;
    ```

    100개의 단일 일괄 처리를 로드했다면, 다음의 결과를 확인할 수 있습니다.

    ```sql
    count 
    -------
     100
    ```

    PII가 검색된 목록 수를 확인할 수 있습니다.

    ```sql
    SELECT COUNT(*) FROM listings WHERE description != description_pii_safe;
    ```

    다음과 유사한 결과가 표시됩니다.

    ```sql
    count 
    -------
        87
    ```

4. 검색된 PII 엔터티 확인: 위의 경우 빈 PII 배열 없이 13개가 있어야 합니다.

    ```sql
    SELECT COUNT(*) FROM listings WHERE pii_entities IS NULL AND description_pii_safe IS NOT NULL;
    ```

    결과:

    ```sql
    count 
    -------
        13
    ```

## 정리

이 연습을 완료한 후에는 만든 Azure 리소스를 삭제합니다. 요금은 데이터베이스 사용량이 아니라 구성된 용량에 대해 부과됩니다. 다음 지침에 따라 리소스 그룹과 이 랩을 위해 만든 모든 리소스를 삭제합니다.

1. 웹 브라우저를 열어 [Azure Portal](https://portal.azure.com/)로 이동한 뒤 홈페이지의 Azure 서비스에서 **리소스 그룹**을 선택합니다.

    ![Azure Portal의 Azure 서비스 아래 리소스 그룹이 빨간색 상자로 강조 표시된 스크린샷.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. 원하는 필드 검색 상자의 필터에 이 랩을 위해 만든 리소스 그룹의 이름을 입력한 다음 목록에서 이 리소스 그룹을 선택합니다.

3. 리소스 그룹의 **개요** 페이지에서 **리소스 그룹 삭제**를 선택합니다.

    ![리소스 그룹 삭제 버튼이 빨간색 상자로 강조 표시된 리소스 그룹의 개요 블레이드 스크린샷.](media/17-resource-group-delete.png)

4. 확인 대화 상자에서 삭제할 리소스 그룹의 이름을 입력하여 확인하고 **삭제**를 선택합니다.

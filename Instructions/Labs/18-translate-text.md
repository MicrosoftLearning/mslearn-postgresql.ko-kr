---
lab:
  title: Azure AI 번역기를 사용하여 텍스트 번역
  module: Translate Text using the Azure AI Translator and Azure Database for PostgreSQL
---

# Azure AI 번역기를 사용하여 텍스트 번역

Margie's Travel의 수석 개발자로서 국제화 노력을 지원하라는 요청을 받았습니다. 현재 회사의 단기 임대 서비스에 대한 모든 임대 목록은 영어로 제공됩니다. 이러한 목록을 많은 개발 노력 없이 다양한 언어로 번역하고 싶습니다. 모든 데이터는 Azure Database for PostgreSQL 유연한 서버에 호스팅되며, 번역을 수행하기 위해 Azure AI 서비스를 사용하려고 합니다.

이 연습에서는 Azure Database for PostgreSQL 유연한 서버 데이터베이스를 통해 Azure AI 번역기 서비스를 사용하여 영어 텍스트를 다양한 언어로 번역합니다.

## 시작하기 전에

관리 권한이 있는 [Azure 구독](https://azure.microsoft.com/free)이 필요합니다.

### Azure 구독에 리소스 배포

이 단계에서는 Azure Cloud Shell에서 Azure CLI 명령을 사용하여 리소스 그룹을 만들고 Bicep 스크립트를 실행하여 이 연습을 완료하는 데 필요한 Azure 서비스를 Azure 구독에 배포하는 방법을 안내합니다.

1. 웹 브라우저를 열고 [Azure Portal](https://portal.azure.com/)로 이동합니다.

2. Azure Portal 도구 모음의 **Cloud Shell** 아이콘을 선택하여 브라우저 창 아래쪽에 새 [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 창을 엽니다.

    ![Cloud Shell 아이콘을 빨간색 상자로 강조 표시한 Azure 도구 모음의 스크린샷.](media/11-portal-toolbar-cloud-shell.png)

    메시지가 표시되는 경우 *Bash* 셸을 여는 데 필요한 옵션을 선택합니다. 이전에 *PowerShell* 콘솔을 사용한 적이 있다면 이것을 *Bash* 셸로 전환합니다.

3. Cloud Shell 프롬프트에서 다음을 입력하여 연습 리소스가 포함된 GitHub 리포지토리를 복제합니다.

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

    이전 모듈에서 이 GitHub 리포지토리를 이미 복제한 경우 계속 사용할 수 있으며 다음과 같은 오류 메시지가 표시될 수 있습니다.

    ```bash
    fatal: destination path 'mslearn-postgresql' already exists and is not an empty directory.
    ```

    이 메시지가 표시되면 다음 단계를 안전하게 진행할 수 있습니다.

4. 그 다음 Azure CLI 명령을 사용하여 Azure 리소스를 만들 때 중복 입력을 줄이기 위해 변수를 정의하는 세 가지 명령을 실행합니다. 이 변수는 리소스 그룹에 할당할 이름(`RG_NAME`), 리소스를 배포할 Azure 지역(`REGION`), PostgreSQL 관리자 로그인용 임의 생성 암호(`ADMIN_PASSWORD`)를 나타냅니다.

    첫 번째 명령에서는 해당 변수에 `eastus` 지역을 할당하지만, 원하는 위치로 바꿀 수도 있습니다. 그러나 기본값을 바꾸는 경우 다른 [추상 요약을 지원하는 Azure 지역](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support)을 선택해야 이 학습 경로의 모듈에 있는 모든 작업을 완료할 수 있습니다.

    ```bash
    REGION=eastus
    ```

    다음 명령에서는 이 연습에서 사용할 모든 리소스를 저장하는 리소스 그룹에 사용할 이름을 할당합니다. 해당 변수에는 `rg-learn-postgresql-ai-$REGION` 리소스 그룹 이름을 할당하며, 여기서 `$REGION` 부분은 앞에서 지정한 위치입니다. 그러나 원하는 다른 리소스 그룹 이름으로 변경할 수 있습니다.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    마지막 명령에서는 PostgreSQL 관리자 로그인에 사용할 암호를 임의로 생성합니다. 나중에 PostgreSQL 유연한 서버에 연결하는 데 사용할 안전한 장소에 복사해야 합니다.

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-translate.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep 배포 스크립트는 이 연습을 완료하는 데 필요한 Azure 서비스를 리소스 그룹으로 프로비전합니다. 배포된 리소스에는 Azure Database for PostgreSQL 유연한 서버 및 Azure AI 번역기 서비스가 포함됩니다. 또한 Bicep 스크립트는 PostgreSQL 서버의 _allowlist_에 (azure.extensions 서버 매개 변수를 통해) `azure_ai` 및 `vector` 확장을 추가하고 서버에 `rentals`이라는 데이터베이스를 생성하는 등 몇 가지 구성 단계를 수행합니다. **Bicep 파일은 이 학습 경로의 다른 모듈과 다릅니다.**

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

1. [Azure Portal](https://portal.azure.com/)에서 새로 만든 Azure Database for PostgreSQL 유연한 서버로 이동합니다.

2. 리소스 메뉴의 **설정**에서 **데이터베이스**를 선택하고 `rentals` 데이터베이스에 대한 **연결**을 선택합니다.

    ![Azure Database for PostgreSQL 데이터베이스 페이지의 스크린샷. rentals 데이터베이스에 대한 데이터베이스 및 연결이 빨간색 상자로 강조 표시됩니다.](media/17-postgresql-rentals-database-connect.png)

3. Cloud Shell의 "사용자 pgAdmin에 대한 암호" 프롬프트에서 **pgAdmin** 로그인에 대해 임의로 생성된 암호를 입력합니다.

    로그인하면 `rentals` 데이터베이스에 대한 `psql` 프롬프트가 표시됩니다.

4. 이 연습의 이후 부분에서는 계속 Cloud Shell에서 작업하므로 창의 오른쪽 위에 있는 **최대화** 버튼을 선택하여 브라우저 창 내에서 해당 창을 확장하면 도움이 될 수 있습니다.

    ![빨간색 상자로 강조 표시된 최대화 버튼이 있는 Azure Cloud Shell 창의 스크린샷.](media/17-azure-cloud-shell-pane-maximize.png)

## 목록 데이터를 이용해 데이터베이스 작성

번역하려면 영어 목록 데이터를 사용할 수 있어야 합니다. 이전 모듈에서 `rentals` 데이터베이스에 `listings` 테이블을 만들지 않은 경우 다음 지침에 따라 만듭니다.

1. 다음 명령을 실행하여 임대 부동산 목록 데이터를 저장하기 위한 `listings` 테이블을 만듭니다.

    ```sql
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

2. 다음으로, `COPY` 명령을 사용하여 위에서 만든 각 테이블에 CSV 파일의 데이터를 로드합니다. 먼저 다음 명령을 사용하여 `listings` 테이블을 채웁니다.

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    명령의 출력은 `COPY 50`이어야 합니다. 이는 CSV 파일의 테이블에 50개의 행이 기록되었음을 나타냅니다.

## 번역을 위한 추가 테이블 만들기

`listings` 데이터가 있지만 번역을 수행하려면 두 개의 추가 테이블이 필요합니다.

1. 다음 명령을 실행하여 `languages` 및 `listing_translations` 테이블을 만듭니다.

    ```sql
    CREATE TABLE languages (
        code VARCHAR(7) NOT NULL PRIMARY KEY
    );
    ```

    ```sql
    CREATE TABLE listing_translations(
        id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        listing_id INT,
        language_code VARCHAR(7),
        description TEXT
    );
    ```

2. 그런 다음 번역을 위해 언어당 하나의 행을 삽입합니다. 이 경우 독일어, 중국어 간체, 힌디어, 헝가리어, 스와힐리어 등 5개 언어에 대한 행을 만듭니다.

    ```sql
    INSERT INTO languages(code)
    VALUES
        ('de'),
        ('zh-Hans'),
        ('hi'),
        ('hu'),
        ('sw');
    ```

    명령 출력은 `INSERT 0 5`이(가) 되어야 하며, 이는 테이블에 5개의 새 행을 삽입했음을 나타냅니다.

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

    Azure Database for PostgreSQL 유연한 서버에 확장을 설치 및 사용하려면 [PostgreSQL 확장 사용 방법](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions)에 설명된 대로 서버의 _허용 목록_에 추가해야 합니다.

2. 이제 [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html) 명령을 사용하여 `azure_ai` 확장을 설치할 수 있습니다.

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` 명령은 스크립트 파일을 실행하여 데이터베이스에 새 확장을 로드합니다. 이 스크립트는 일반적으로 함수, 데이터 형식, 스키마와 같은 새 SQL 개체를 만듭니다. 동일한 이름의 확장이 이미 있는 경우 오류가 표시됩니다. `IF NOT EXISTS` 문을 추가하면 확장이 이미 설치된 경우 오류를 표시하지 않고 명령을 실행할 수 있습니다.

3. 그런 다음, `azure_ai.set_setting()` 함수를 사용하여 Azure AI 번역기 서비스에 대한 연결을 구성해야 합니다. Cloud Shell이 열려 있는 동일한 브라우저 탭을 사용하여 Cloud Shell 창을 최소화하거나 복원한 다음, [Azure Portal](https://portal.azure.com/)에서 Azure AI 번역기 리소스로 이동합니다. Azure AI 번역기 리소스 페이지에서 리소스 메뉴의 **리소스 관리** 섹션에서 **키 및 엔드포인트**를 선택한 다음 사용 가능한 키, 지역 및 문서 번역 엔드포인트 중 하나를 복사합니다.

    ![Azure AI 번역기 서비스의 키 및 엔드포인트 페이지가 표시되며, 키 1, 지역 및 문서 번역 엔드포인트 복사 버튼이 빨간색 상자로 강조 표시된 스크린샷.](media/18-azure-ai-translator-keys-and-endpoint.png)

    `KEY 1` 또는 `KEY 2`를 사용할 수 있습니다. 항상 두 개의 키를 사용하면 서비스 중단 없이 키를 안전하게 회전하고 다시 생성할 수 있습니다.

4. AI 번역기 엔드포인트, 구독 키 및 지역을 가리키도록 `azure_cognitive` 설정을 구성합니다. `azure_cognitive.endpoint` 값은 서비스의 문서 번역 URL입니다. `azure_cognitive.subscription_key` 값은 키 1 또는 키 2입니다. `azure_cognitive.region` 값은 Azure AI 번역기 인스턴스의 지역이 됩니다.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint','https://<YOUR_ENDPOINT>.cognitiveservices.azure.com/');
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '<YOUR_KEY>');
    SELECT azure_ai.set_setting('azure_cognitive.region', '<YOUR_REGION>');
    ```

## 목록 데이터를 번역하는 저장 프로시저 만들기

언어 번역 테이블을 채우기 위해 데이터를 일괄적으로 로드하는 저장 프로시저를 만듭니다.

1. `psql` 프롬프트에서 다음 명령을 실행하여 `translate_listing_descriptions`이라는 이름의 새 저장 프로시저를 만듭니다.

    ```sql
    CREATE OR REPLACE PROCEDURE translate_listing_descriptions(max_num_listings INT DEFAULT 10)
    LANGUAGE plpgsql
    AS $$
    BEGIN
        WITH batch_to_load(id, description) AS
        (
            SELECT id, description
            FROM listings l
            WHERE NOT EXISTS (SELECT * FROM listing_translations ll WHERE ll.listing_id = l.id)
            LIMIT max_num_listings
        )
        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT b.id, l.code, (unnest(tr.translations)).TEXT
        FROM batch_to_load b
            CROSS JOIN languages l
            CROSS JOIN LATERAL azure_cognitive.translate(b.description, l.code) tr;
    END;
    $$;
    ```

    이 저장 프로시저는 5개의 레코드를 일괄 처리로 로드하고 선택한 각 언어로 설명을 번역한 다음 번역된 설명을 `listing_translations` 테이블에 삽입합니다.

2. 다음 SQL 명령을 사용하여 저장 프로시저를 실행합니다.

    ```sql
    CALL translate_listing_descriptions(10);
    ```

    이 호출은 5개 언어로 번역하는 데 임대 목록당 약 1초가 걸리므로 각 실행에는 약 10초가 소요됩니다. 명령 출력은 `CALL`이(가) 되어야 하며 이는 저장 프로시저 호출이 성공했음을 나타냅니다.

3. 저장 프로시저를 네 번 더 호출하여 이 프로시저를 총 다섯 번 호출합니다. 그러면 테이블의 모든 목록에 대한 번역이 생성됩니다.

4. 다음 스크립트를 실행하여 목록 번역 수를 가져옵니다.

    ```sql
    SELECT COUNT(*) FROM listing_translations;
    ```

    호출 결과는 250의 값을 반환해야 하며, 이는 각 목록이 5개 언어로 번역되었음을 나타냅니다. `listing_translations` 테이블을 쿼리하여 데이터를 추가로 분석할 수 있습니다.

## 번역을 사용하여 새 목록을 추가하는 프로시저 만들기

기존 목록을 번역하는 저장 프로시저가 있지만 국제화 계획에는 새 목록을 입력할 때 번역해야 합니다. 이렇게 하려면 다른 저장 프로시저를 만듭니다.

1. `psql` 프롬프트에서 다음 명령을 실행하여 `add_listing`이라는 이름의 새 저장 프로시저를 만듭니다.

    ```sql
    CREATE OR REPLACE PROCEDURE add_listing(id INT, name VARCHAR(255), description TEXT)
    LANGUAGE plpgsql
    AS $$
    DECLARE
    listing_id INT;
    BEGIN
        INSERT INTO listings(id, name, description)
        VALUES(id, name, description);

        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT id, l.code, (unnest(tr.translations)).TEXT
        FROM languages l
            CROSS JOIN LATERAL azure_cognitive.translate(description, l.code) tr;
    END;
    $$;
    ```

    이 저장 프로시저는 `listings` 테이블에 행을 삽입합니다. 그런 다음 `languages` 테이블의 각 언어에 대한 설명을 번역하고 이 번역을 `listing_translations` 테이블에 삽입합니다.

2. 다음 SQL 명령을 사용하여 저장 프로시저를 실행합니다.

    ```sql
    CALL add_listing(51, 'A Beautiful Home', 'This is a beautiful home in a great location.');
    ```

    명령 출력은 `CALL`이(가) 되어야 하며 이는 저장 프로시저 호출이 성공했음을 나타냅니다.

3. 다음 스크립트를 실행하여 새 목록에 대한 번역을 가져옵니다.

    ```sql
    SELECT l.id, l.name, l.description, lt.language_code, lt.description AS translated_description
    FROM listing_translations lt
        INNER JOIN listings l ON lt.listing_id = l.id
    WHERE l.name = 'A Beautiful Home';
    ```

    호출은 다음 테이블과 비슷한 값을 가진 5개의 행을 반환해야 합니다.

    ```sql
     id  | listing_id | language_code |                    description                     
    -----+------------+---------------+------------------------------------------------------
     126 |          2 | de            | Dies ist ein schönes Haus in einer großartigen Lage.
     127 |          2 | zh-Hans       | 这是一个美丽的家，地理位置优越。
     128 |          2 | hi            | यह एक महान स्थान में एक सुंदर घर है।
     129 |          2 | hu            | Ez egy gyönyörű otthon egy nagyszerű helyen.
     130 |          2 | sw            | Hii ni nyumba nzuri katika eneo kubwa.
    ```

## 정리

이 연습을 완료한 후에는 만든 Azure 리소스를 삭제합니다. 요금은 데이터베이스 사용량이 아니라 구성된 용량에 대해 부과됩니다. 다음 지침에 따라 리소스 그룹과 이 랩을 위해 만든 모든 리소스를 삭제합니다.

1. 웹 브라우저를 열어 [Azure Portal](https://portal.azure.com/)로 이동한 뒤 홈페이지의 Azure 서비스에서 **리소스 그룹**을 선택합니다.

    ![Azure Portal의 Azure 서비스 아래 리소스 그룹이 빨간색 상자로 강조 표시된 스크린샷.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. 필드 검색 상자의 필터에 이 랩을 위해 만든 리소스 그룹의 이름을 입력한 다음 목록에서 이 리소스 그룹을 선택합니다.

3. 리소스 그룹의 **개요** 페이지에서 **리소스 그룹 삭제**를 선택합니다.

    ![리소스 그룹 삭제 버튼이 빨간색 상자로 강조 표시된 리소스 그룹의 개요 블레이드 스크린샷.](media/17-resource-group-delete.png)

4. 확인 대화 상자에서 삭제할 리소스 그룹의 이름을 입력하여 확인하고 **삭제**를 선택합니다.

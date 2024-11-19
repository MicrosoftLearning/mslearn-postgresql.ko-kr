---
lab:
  title: 잠금 이해
  module: Understand concurrency in PostgreSQL
---

# 잠금 이해

이 연습에서는 PostgreSQL에서 시스템 매개 변수와 메타데이터를 살펴봅니다.

## 시작하기 전에

이 모듈의 연습을 완료하려면 자체 Azure 구독이 필요합니다. Azure 구독이 없는 경우 [Azure 무료 계정으로 클라우드에서 빌드](https://azure.microsoft.com/free/)에서 무료 평가판 계정을 설정할 수 있습니다.

## 연습 환경 만들기

### Azure 구독에 리소스 배포

이 단계에서는 Azure Cloud Shell에서 Azure CLI 명령을 사용하여 리소스 그룹을 만들고 Bicep 스크립트를 실행하여 이 연습을 완료하는 데 필요한 Azure 서비스를 Azure 구독에 배포하는 방법을 안내합니다.

1. 웹 브라우저를 열고 [Azure Portal](https://portal.azure.com/)로 이동합니다.

2. Azure Portal 도구 모음의 **Cloud Shell** 아이콘을 선택하여 브라우저 창 아래쪽에 새 [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 창을 엽니다.

    ![Cloud Shell 아이콘을 빨간색 상자로 강조 표시한 Azure 도구 모음의 스크린샷.](media/08-portal-toolbar-cloud-shell.png)

    메시지가 표시되는 경우 *Bash* 셸을 여는 데 필요한 옵션을 선택합니다. 이전에 *PowerShell* 콘솔을 사용한 적이 있다면 이것을 *Bash* 셸로 전환합니다.

3. Cloud Shell 프롬프트에서 다음을 입력하여 연습 리소스가 포함된 GitHub 리포지토리를 복제합니다.

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. 그 다음 Azure CLI 명령을 사용하여 Azure 리소스를 만들 때 중복 입력을 줄이기 위해 변수를 정의하는 세 가지 명령을 실행합니다. 이 변수는 리소스 그룹에 할당할 이름(`RG_NAME`), 리소스를 배포할 Azure 지역(`REGION`), PostgreSQL 관리자 로그인용 임의 생성 암호(`ADMIN_PASSWORD`)를 나타냅니다.

    첫 번째 명령에서는 해당 변수에 `eastus` 지역을 할당하지만, 원하는 위치로 바꿀 수도 있습니다.

    ```bash
    REGION=eastus
    ```

    다음 명령에서는 이 연습에서 사용할 모든 리소스를 저장하는 리소스 그룹에 사용할 이름을 할당합니다. 해당 변수에는 `rg-learn-work-with-postgresql-$REGION` 리소스 그룹 이름을 할당하며, 여기서 `$REGION` 부분은 앞에서 지정한 위치입니다. 그러나 원하는 다른 리소스 그룹 이름으로 변경할 수 있습니다.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    마지막 명령에서는 PostgreSQL 관리자 로그인에 사용할 암호를 임의로 생성합니다. 나중에 PostgreSQL 유연한 서버에 연결하는 데 사용하기 위해 이 암호를 안전한 장소에 복사해 두어야 합니다.

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
    ```

    Bicep 배포 스크립트는 이 연습을 완료하는 데 필요한 Azure 서비스를 리소스 그룹으로 프로비전합니다. 배포되는 리소스는 Azure Database for PostgreSQL- 유연한 서버입니다. 이 bicep 스크립트는 데이터베이스도 만듭니다. 이는 명령줄에서 매개 변수로 구성할 수 있습니다.

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
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 책임 있는 AI 계약을 수락해야 하는 요구 사항으로 인해 스크립트가 AI 리소스를 만들 수 없는 경우 다음과 같은 오류가 발생할 수 있습니다. 이 경우 Azure Portal 사용자 인터페이스를 사용하여 Azure AI 서비스 리소스를 만든 다음 배포 스크립트를 다시 실행합니다.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Azure Cloud Shell에서 psql을 사용하여 데이터베이스에 연결

이 작업에서는 [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview)의 [psql 명령줄 유틸리티](https://www.postgresql.org/docs/current/app-psql.html)를 사용하여 Azure Database for PostgreSQL 서버의 `adventureworks` 데이터베이스에 연결합니다.

1. [Azure Portal](https://portal.azure.com/)에서 새로 만든 Azure Database for PostgreSQL 유연한 서버로 이동합니다.

2. 리소스 메뉴의 **설정**에서 **데이터베이스**를 선택하고 `adventureworks` 데이터베이스에 대한 **연결**을 선택합니다.

    ![Azure Database for PostgreSQL 데이터베이스 페이지의 스크린샷. adventureworks 데이터베이스에 대한 데이터베이스 및 연결이 빨간색 상자로 강조 표시됩니다.](media/08-postgresql-adventureworks-database-connect.png)

3. Cloud Shell의 "사용자 pgAdmin에 대한 암호" 프롬프트에서 **pgAdmin** 로그인에 대해 임의로 생성된 암호를 입력합니다.

    로그인하면 `adventureworks` 데이터베이스에 대한 `psql` 프롬프트가 표시됩니다.

4. 이 연습의 이후 부분에서는 계속 Cloud Shell에서 작업하므로 창의 오른쪽 위에 있는 **최대화** 버튼을 선택하여 브라우저 창 내에서 해당 창을 확장하면 도움이 될 수 있습니다.

    ![빨간색 상자로 강조 표시된 최대화 버튼이 있는 Azure Cloud Shell 창의 스크린샷.](media/08-azure-cloud-shell-pane-maximize.png)

### 데이터를 이용해 데이터베이스 작성

1. 데이터베이스에 테이블을 만들고 샘플 데이터로 채워 이 연습에서 잠금을 검토할 때 사용할 정보를 준비합니다.
1. 다음 명령을 실행하여 데이터를 로드할 `production.workorder` 테이블을 만듭니다.

    ```sql
    DROP SCHEMA IF EXISTS production CASCADE;
    CREATE SCHEMA production;
    
    DROP TABLE IF EXISTS production.workorder;
    CREATE TABLE production.workorder
    (
        workorderid integer NOT NULL,
        productid integer NOT NULL,
        orderqty integer NOT NULL,
        scrappedqty smallint NOT NULL,
        startdate timestamp without time zone NOT NULL,
        enddate timestamp without time zone,
        duedate timestamp without time zone NOT NULL,
        scrapreasonid smallint,
        modifieddate timestamp without time zone NOT NULL DEFAULT now()
    )
    WITH (
        OIDS = FALSE
    )
    TABLESPACE pg_default;
    ```

1. 다음으로, `COPY` 명령을 사용하여 위에서 만든 테이블에 CSV 파일의 데이터를 로드합니다. 먼저 다음 명령을 사용하여 `production.workorder` 테이블을 채웁니다.

    ```sql
    \COPY production.workorder FROM 'mslearn-postgresql/Allfiles/Labs/08/Lab8_workorder.csv' CSV HEADER
    ```

    명령의 출력은 `COPY 72591`이어야 합니다. 이는 CSV 파일의 테이블에 72591개의 행이 기록되었음을 나타냅니다.

1. 데이터가 로드되면 Cloud Shell 창을 닫습니다.

### Azure Data Studio로 데이터베이스에 연결

1. Azure Data Studio를 아직 설치하지 않은 경우 [***Azure Data Studio***를 다운로드하여 설치](https://go.microsoft.com/fwlink/?linkid=2282284)합니다.
1. Azure Data Studio를 시작합니다.
1. Azure Data Studio에서 **PostgreSQL** 확장을 설치하지 않은 경우 지금 설치합니다.
1. **서버**를 선택하고 **새 연결**을 선택합니다.
1. **연결 형식**에서 **PostgreSQL**을 선택합니다.
1. **서버 이름**에 서버를 배포할 때 지정한 값을 입력합니다.
1. **사용자 이름**에 **pgAdmin**을 입력합니다.
1. **암호**에서 앞서 생성한 **pgAdmin** 로그인에 대해 임의로 생성된 암호를 입력합니다.
1. **암호 저장**을 선택합니다.
1. **연결**을 클릭합니다.

## 작업 1: 기본 잠금 동작 조사

1. Azure Data Studio를 엽니다.
1. **데이터베이스**를 확장하고 **adventureworks**를 마우스 오른쪽 단추로 클릭한 다음, **새 쿼리**를 선택합니다.
   
    ![새 쿼리 컨텍스트 메뉴 항목을 강조 표시한 adventureworks 데이터베이스 스크린샷](media/08-new-query.png)

1. **파일** 및 **새 쿼리**로 이동합니다. 이제 이름이 **SQL_Query_1**로 시작하는 쿼리 탭과 이름이 **SQL_Query_2**로 시작하는 다른 쿼리 탭이 있어야 합니다.
1. **SQLQuery_1** 탭을 선택하고, 다음 쿼리를 입력하고, **실행**을 선택합니다.

    ```sql
    SELECT * FROM production.workorder
    ORDER BY scrappedqty DESC;
    ```

1. 첫 번째 행의 **scrappedqty** 값은 **673**입니다.
1. **SQLQuery_2** 탭을 선택하고, 다음 쿼리를 입력하고, **실행**을 선택합니다.

    ```sql
    BEGIN TRANSACTION;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. 두 번째 쿼리는 트랜잭션을 시작하지만 트랜잭션을 커밋하지는 않습니다.
1. **SQLQuery_1**로 돌아가서 쿼리를 다시 실행합니다.
1. 첫 번째 행의 **stockedqty** 값은 여전히 **673**입니다. 쿼리가 데이터의 스냅샷을 사용하고 있으며 다른 트랜잭션의 업데이트가 표시되지 않습니다.
1. **SQLQuery_2** 탭을 선택하고, 기존 쿼리를 삭제하고, 다음 쿼리를 입력하고, **실행**을 선택합니다.

    ```sql
    ROLLBACK TRANSACTION;
    ```

## 작업 2: 트랜잭션에 테이블 잠금 적용

1. **SQLQuery_2** 탭을 선택하고, 다음 쿼리를 입력하고, **실행**을 선택합니다.

    ```sql
    BEGIN TRANSACTION;
    LOCK TABLE production.workorder IN ACCESS EXCLUSIVE MODE;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. 두 번째 쿼리는 트랜잭션을 시작하지만 트랜잭션을 커밋하지는 않습니다.
1. **SQLQuery_1**로 돌아가서 쿼리를 다시 실행합니다.
1. 트랜잭션이 차단되어 아무리 오래 기다리더라도 완료되지 않습니다.
1. **SQLQuery_2** 탭을 선택하고, 기존 쿼리를 삭제하고, 다음 쿼리를 입력하고, **실행**을 선택합니다.

    ```sql
    ROLLBACK TRANSACTION;
    ```

1. **SQLQuery_1**로 돌아가서 몇 초 동안 기다리면 블록이 제거되어 쿼리가 완료되었음을 알 수 있습니다.

이 연습에서는 기본 잠금 동작을 살펴보았습니다. 그런 다음, 잠금을 명시적으로 적용하고 일부 잠금이 매우 높은 수준의 보호를 제공하지만 이러한 잠금은 성능에도 영향을 줄 수 있음을 보았습니다.

## 연습 정리

이 연습에서 배포한 Azure Database for PostgreSQL에는 요금이 발생하며, 이 연습 후에 서버를 삭제할 수 있습니다. 또는 **rg-learn-work-with-postgresql-eastus** 리소스 그룹을 삭제하여 이 연습의 일부로 배포한 모든 리소스를 제거할 수 있습니다.

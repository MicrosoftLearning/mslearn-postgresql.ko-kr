---
lab:
  title: 오프라인 PostgreSQL 데이터베이스 마이그레이션
  module: Migrate to Azure Database for PostgreSQL Flexible Server
---

# 오프라인 PostgreSQL 데이터베이스 마이그레이션

이 연습에서는 Azure Database for PostgreSQL 유연한 서버를 만들고 Azure Database for PostgreSQL 유연한 서버 내의 마이그레이션 기능을 사용하여 온-프레미스 PostgreSQL 서버 또는 Azure Database for PostgreSQL 서버에서 오프라인 데이터베이스 마이그레이션을 수행합니다.

## 시작하기 전에

이 연습을 완료하려면 자체 Azure 구독이 필요합니다. Azure 구독이 아직 없는 경우 [Azure 평가판](https://azure.microsoft.com/free)을 만들 수 있습니다.

### Azure에서 연결을 허용하도록 pg_hba.conf 파일을 편집합니다(외부 PostgreSQL 서버에서 마이그레이션하지 않는 경우 건너뛰기)

> [!NOTE]
> 이 랩에서는 마이그레이션의 원본 및 대상으로 사용할 두 개의 Azure Database for PostgreSQL을 만듭니다. 그러나 사용자 고유의 환경을 사용하는 경우 이 연습을 완료하려면 데이터베이스, 적절한 권한 및 네트워크 액세스 권한이 있는 기존 PostgreSQL 서버에 액세스해야 합니다.
> 
> 사용자 고유의 환경을 사용하는 경우 이 연습에서는 마이그레이션의 원본으로 사용하는 서버가 데이터베이스를 연결하고 마이그레이션할 수 있도록 Azure Database for PostgreSQL 유연한 서버에 액세스할 수 있어야 합니다. 이렇게 하려면 공용 IP 주소 및 포트를 통해 원본 서버에 액세스할 수 있어야 합니다. Azure 지역 IP 주소 목록은 [Azure IP 범위 및 서비스 태그 - 공용 클라우드](https://www.microsoft.com/en-gb/download/details.aspx?id=56519)에서 다운로드하여 사용하는 Azure 지역에 따라 방화벽 규칙에서 허용되는 IP 주소의 범위를 최소화하는 데 도움을 받을 수 있습니다. 서버의 방화벽을 열어 Azure Database for PostgreSQL 유연한 서버의 마이그레이션 기능이 기본적으로 TCP 포트 **5432**인 원본 PostgreSQL 서버에 액세스할 수 있도록 합니다.
>
> 원본 데이터베이스 앞에 방화벽 어플라이언스를 사용하는 경우, Azure Database for PostgreSQL 유연한 서버의 마이그레이션 기능에서 마이그레이션을 위해 원본 데이터베이스에 액세스할 수 있도록 방화벽 규칙을 추가해야 할 수 있습니다.
>
> 마이그레이션에 지원되는 PostgreSQL의 최대 버전은 버전 16입니다.

원본 PostgreSQL 서버는 인스턴스가 Azure Database for PostgreSQL 유연한 서버의 연결을 허용하도록 pg_hba.conf 파일을 업데이트해야 합니다.

1. pg_hba.conf에 항목을 추가하여 Azure IP 범위에서 연결을 허용합니다. pg_hba.conf의 항목은 연결할 수 있는 호스트, 데이터베이스, 사용자 및 사용할 수 있는 인증 방법을 지정합니다.
1. 예를 들어 Azure 서비스가 IP 범위 104.45.0.0/16 내에 있는 경우입니다. 모든 사용자가 암호 인증을 사용하여 이 범위의 모든 데이터베이스에 연결할 수 있도록 하려면 다음을 추가합니다.

``` bash
host    all    all    104.45.0.0/16    md5
```

1. Azure를 포함하여 인터넷을 통한 연결을 허용하는 경우 강력한 인증 메커니즘이 있는지 확인합니다.

- 강력한 암호를 사용합니다.
- 액세스를 최대한 적은 수의 IP 주소로 제한합니다.
- VPN 또는 VNet 사용: 가능하면 VPN(가상 사설망) 또는 Azure VNet(Virtual Network)을 구성하여 Azure와 PostgreSQL 서버 간에 보안 터널을 제공합니다.

1. pg_hba.conf에 대한 변경 내용을 저장한 후 psql 세션 내에서 SQL 명령을 사용하여 변경 내용을 적용하려면 PostgreSQL 구성을 다시 로드해야 합니다.

```sql
SELECT pg_reload_conf();
```

1. Azure에서 로컬 PostgreSQL 서버로의 연결을 테스트하여 구성이 예상대로 작동하는지 확인합니다. Azure VM 또는 아웃바운드 데이터베이스 연결 지원하는 서비스에서 이 작업을 수행할 수 있습니다.

### Azure 구독에 리소스 배포

이 단계에서는 Azure Cloud Shell에서 Azure CLI 명령을 사용하여 리소스 그룹을 만들고 Bicep 스크립트를 실행하여 이 연습을 완료하는 데 필요한 Azure 서비스를 Azure 구독에 배포하는 방법을 안내합니다.

1. 웹 브라우저를 열고 [Azure Portal](https://portal.azure.com/)로 이동합니다.

1. Azure Portal 도구 모음의 **Cloud Shell** 아이콘을 선택하여 브라우저 창 아래쪽에 새 [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 창을 엽니다.

    ![Cloud Shell 아이콘을 빨간색 상자로 강조 표시한 Azure 도구 모음을 보여주는 스크린샷.](media/11-portal-toolbar-cloud-shell.png)

    메시지가 표시되면 *Bash* 셸을 여는 데 필요한 옵션을 선택합니다. 이전에 *PowerShell* 콘솔을 사용한 경우 *Bash* 셸로 전환합니다.

1. Cloud Shell 프롬프트에서 다음을 입력하여 연습 리소스가 포함된 GitHub 리포지토리를 복제합니다.

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. 다음으로, Azure CLI 명령을 사용하여 Azure 리소스를 만들 때 중복 입력을 줄이기 위해 변수를 정의하는 세 가지 명령을 실행합니다. 이 변수는 리소스 그룹에 할당할 이름(`RG_NAME`), 리소스를 배포할 Azure 지역(`REGION`), PostgreSQL 관리자 로그인용 임의 생성 암호(`ADMIN_PASSWORD`)를 나타냅니다.

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

1. 두 개 이상의 Azure 구독에 액세스할 수 있고 기본 구독이 이 연습에 대한 리소스 그룹 및 기타 리소스를 만드는 데 사용할 구독이 아닌 경우, 이 명령을 실행하여 적절한 구독을 설정하고 `<subscriptionName|subscriptionId>` 토큰을 사용할 구독의 이름 또는 ID로 바꿉니다.

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

1. 다음 Azure CLI 명령을 실행하여 리소스 그룹을 만듭니다.

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. 마지막으로 Azure CLI를 사용하여 Bicep 배포 스크립트를 실행하는 방법으로 리소스 그룹에 Azure 리소스를 프로비전합니다.

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server-migration.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
    ```

    Bicep 배포 스크립트는 이 연습을 완료하는 데 필요한 Azure 서비스를 리소스 그룹으로 프로비전합니다. 배포된 리소스는 두 개의 Azure Database for PostgreSQL - 유연한 서버입니다. 마이그레이션을 위한 원본 및 대상 서버입니다.

    배포를 완료하는 데 일반적으로 몇 분 정도 걸립니다(5-10분 이상). Cloud Shell에서 모니터링하거나 위에서 만든 리소스 그룹에 대한 **배포** 페이지로 이동하여 배포 진행 상황을 관찰할 수 있습니다.

1. 리소스 배포가 완료되면 Cloud Shell 창을 닫습니다.

1. Azure Portal에서 두 개의 새 Azure Database for PostgreSQL 서버의 이름을 검토합니다. 원본 서버의 데이터베이스를 나열하면 **adventureworks** 데이터베이스가 포함되지만 대상 데이터베이스는 포함되지 않습니다.

1. **네트워킹** 섹션의 *두* 서버 아래에서
    1. **+ 현재 IP 주소 추가(xxx.xxx.xxx)** 및 **저장**을 선택합니다.
    1. **Azure 내의 모든 Azure 서비스에서 이 서버로의 퍼블릭 액세스 허용** 확인란을 선택합니다.
    1. **공용 IP 주소를 사용하여 인터넷을 통해 이 리소스에 대한 공용 액세스 허용** 확인란을 선택합니다.

> [!NOTE]
> 프로덕션 환경에서는 Azure Database for PostgreSQL 서버에 액세스하려는 옵션, 네트워크 및 IP만 선택해야 합니다. 

> [!NOTE]
> 앞에서 설명한 것처럼 이 Bicep 스크립트는 두 개의 Azure Database for PostgreSQL 서버인 원본 및 대상 서버를 만듭니다.  ***환경에서 온-프레미스 PostgreSQL 서버를 이 랩의 원본 서버로 사용하는 경우 다음 지침의 원본 서버 연결 정보를 사용자 환경에서 온-프레미스 서버의 연결 정보로 바꿉니다.***  환경과 Azure 모두에서 필요한 방화벽 규칙을 사용하도록 설정해야 합니다.
    
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

## 마이그레이션을 위한 데이터베이스, 테이블 및 데이터 만들기

이 랩에서는 온-프레미스 PostreSQL 서버 또는 Azure Database for PostgreSQL 서버에서 마이그레이션하는 옵션을 제공합니다. 마이그레이션하는 서버 유형에 대한 지침을 따릅니다.

### 온-프레미스 PostgreSQL 서버에 데이터베이스 만들기(Azure Database for PostgreSQL 서버에서 마이그레이션하는 경우 건너뛰기)

이제 Azure Database for PostgreSQL 유연한 서버로 마이그레이션할 데이터베이스를 설정해야 합니다. 이 단계를 원본 PostgreSQL 서버 인스턴스에서 완료해야 하며, 이 랩을 완료하기 위해 Azure Database for PostgreSQL 유연한 서버에 액세스할 수 있어야 합니다.

먼저 빈 데이터베이스를 만들어 테이블을 만든 다음 데이터로 로드해야 합니다. 먼저 ***Lab10_setupTable.sql*** 및 ***Lab10_workorder.csv*** 파일을 [리포지토리](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/10)에서 로컬 드라이브(예: **C:\\**)로 다운로드해야 합니다.
이러한 파일이 있으면 다음 명령을 사용하여 데이터베이스를 만들고 **PostgreSQL 서버에 필요한 대로 호스트, 포트 및 사용자 이름 값을 바꿀 수 있습니다.**

```bash
psql --host=localhost --port=5432 --username=pgadmin --command="CREATE DATABASE adventureworks;"
```

다음 명령을 실행하여 데이터를 로드할 `production.workorder` 테이블을 만듭니다.

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
    ALTER TABLE production.workorder OWNER to pgAdmin;
```

```sql
psql --host=localhost --port=5432 --username=postgres --dbname=adventureworks --command="\COPY production.workorder FROM 'C:\Lab10_workorder.csv' CSV HEADER"
```

명령의 출력은 `COPY 72101`이(가) 되어야 하며 이는 CSV 파일의 테이블에 72101개의 행이 기록되었음을 나타냅니다.

## 마이그레이션 전(Azure Database for PostgreSQL 서버에서 마이그레이션하는 경우 건너뛰기)

원본 서버에서 데이터베이스의 오프라인 마이그레이션을 시작하기 전에 대상 서버가 구성되고 준비되었는지 확인해야 합니다.

1. 원본 서버에서 새 유연한 서버로 사용자 및 역할을 마이그레이션합니다. 이 작업은 다음 코드와 함께 pg_dumpall 도구를 사용하여 수행할 수 있습니다.
    1. 슈퍼 사용자 역할은 Azure Database for PostgreSQL에서 지원되지 않으므로 이러한 권한을 가진 사용자는 마이그레이션 전에 해당 역할을 제거해야 합니다.

```bash
pg_dumpall --globals-only -U <<username>> -f <<filename>>.sql
```

1. 원본 서버의 서버 매개 변수 값을 대상 서버에 일치시킵니다.
1. 대상에서 고가용성 및 읽기 복제본을 사용하지 않도록 설정합니다.

### Azure Database for PostgreSQL 서버에 데이터베이스 만들기(온-프레미스 PostgreSQL 서버에서 마이그레이션하는 경우 건너뛰기)

이제 Azure Database for PostgreSQL 유연한 서버로 마이그레이션할 데이터베이스를 설정해야 합니다. 이 단계를 원본 PostgreSQL 서버 인스턴스에서 완료해야 하며, 이 랩을 완료하기 위해 Azure Database for PostgreSQL 유연한 서버에 액세스할 수 있어야 합니다.

먼저 빈 데이터베이스를 만들어 테이블을 만든 다음 데이터로 로드해야 합니다. 

1. [Azure Portal](https://portal.azure.com/)에서 새로 만든 원본 Azure Database for PostgreSQL 서버로 이동합니다(_**psql-learn-source**_-location-uniquevalue).

1. 리소스 메뉴의 **설정**에서 **데이터베이스**를 선택하고 `adventureworks` 데이터베이스에 대한 **연결**을 선택합니다.

    ![Azure Database for PostgreSQL 데이터베이스 페이지의 스크린샷. adventureworks 데이터베이스에 대한 데이터베이스 및 연결이 빨간색 상자로 강조 표시됩니다.](media/08-postgresql-adventureworks-database-connect.png)

1. Cloud Shell의 "사용자 pgAdmin에 대한 암호" 프롬프트에서 **pgAdmin** 로그인에 대해 임의로 생성된 암호를 입력합니다.

    로그인하면 `adventureworks` 데이터베이스에 대한 `psql` 프롬프트가 표시됩니다.

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

    ```sql
    \COPY production.workorder FROM 'mslearn-postgresql/Allfiles/Labs/10/Lab10_workorder.csv' CSV HEADER
    ```

    명령의 출력은 `COPY 72101`이(가) 되어야 하며 이는 CSV 파일의 테이블에 72101개의 행이 기록되었음을 나타냅니다.

1. **Cloud Shell**을 닫습니다.

## Azure Database for PostgreSQL 유연한 서버에서 데이터베이스 마이그레이션 프로젝트 만들기

1. 대상 서버의 유연한 서버 블레이드의 왼쪽 메뉴에서 **마이그레이션**을 선택합니다.

   ![Azure Database for PostgreSQL 유연한 서버 마이그레이션 옵션.](./media/10-pgflex-migation.png)

1. **마이그레이션** 블레이드 상단의 **+ 만들기** 옵션을 클릭합니다.
   > **참고**: **+ 만들기** 옵션을 사용할 수 없는 경우 **컴퓨팅 + 스토리지**를 선택하고 컴퓨팅 계층을 **범용** 또는 **메모리 최적화**로 변경하고 마이그레이션 프로세스 다시 만들기를 시도합니다. 마이그레이션에 성공하면 컴퓨팅 계층을 **버스트 가능**으로 다시 변경할 수 있습니다.
1. **설정** 탭에서 각 필드에 다음 내용을 입력합니다.
    1. 마이그레이션 이름 - **`Migration-AdventureWorks`**
    1. 원본 서버 유형 - 이 랩의 경우 온-프레미스에서든 또는 Azure Database for PostgreSQL에서든 마이그레이션 수행 시 **온-프레미스 서버**를 선택합니다. 프로덕션 환경에서 올바른 원본 서버 유형을 선택합니다.
    1. 마이그레이션 옵션 - **유효성 검사 및 마이그레이션**.
    1. 마이그레이션 모드 - **오프라인**. 
    1. **다음: 런타임 서버 선택 >** 을 선택합니다.
    1. *런타임 서버 사용*에 대해 **아니요**를 선택합니다.
    1. **원본에 연결 >** 을 선택합니다.

    ![Azure Database for PostgreSQL 유연한 서버에서 오프라인 데이터베이스 마이그레이션 설정](./media/10-pgflex-migation-setup.png)

1. Azure Database for PostgreSQL에서 마이그레이션하는 경우 - **원본에 연결** 탭에서 각 필드에 다음과 같이 입력합니다.
    1. 서버 이름 - 원본으로 사용하는 서버의 주소입니다.
    1. 포트 - PostgreSQL 인스턴스가 원본 서버에서 사용하는 포트입니다(기본값: 5432).
    1. 서버 관리자 로그인 이름 - PostgreSQL 인스턴스의 관리 사용자 이름(기본값 pgAdmin)입니다.
    1. 암호 - 이전 단계에서 지정한 PostgreSQL 관리 사용자의 암호입니다.
    1. SSL 모드 - Prefer.
    1. **원본에 연결** 옵션을 클릭하여 제공된 연결 세부 정보의 유효성을 검사합니다.
    1. **다음: 마이그레이션 대상 선택** 단추를 클릭하여 진행합니다.

1. 마이그레이션하는 대상 서버에 대한 연결 세부 정보가 자동으로 완료되어야 합니다.
    1. 암호 필드에 bicep 스크립트를 사용하여 만든 **pgAdmin** 로그인에 대해 임의로 생성된 암호를 입력합니다.
    1. **대상에 연결** 옵션을 클릭하여 제공된 연결 세부 정보의 유효성을 검사합니다.
    1. **다음 : 마이그레이션할 데이터베이스 선택 >** 단추를 클릭하여 진행합니다.
1. **마이그레이션할 데이터베이스 선택** 탭에서 유연한 서버로 마이그레이션할 원본 서버의 **AdventureWorks**를 선택합니다.

    ![Azure Database for PostgreSQL 유연한 서버로의 마이그레이션을 위한 데이터베이스를 선택합니다.](./media/10-pgflex-migation-dbSelection.png)

1. **다음 : 요약 >** 단추를 클릭하여 진행하고 제공된 데이터를 검토합니다.
1. **요약** 탭에서 정보를 검토한 후 **유효성 검사 및 마이그레이션 시작** 단추를 클릭하여 유연한 서버로의 마이그레이션을 시작합니다.
1. **마이그레이션** 탭에서 위쪽 메뉴의 **새로 고침** 단추를 사용하여 마이그레이션 진행률을 모니터링하여 유효성 검사 및 마이그레이션 프로세스의 진행률을 볼 수 있습니다.
    1. **Migration-AdventureWorks** 작업을 클릭하면 마이그레이션 작업의 진행 상황에 대한 자세한 정보를 볼 수 있습니다.
1. 마이그레이션이 완료되면 대상 서버를 확인합니다. 이제 해당 서버 아래에 **adventureworks** 데이터베이스도 나열됩니다.

마이그레이션 프로세스가 완료되면, 새 데이터베이스에서 데이터 유효성 검사 및 고가용성 구성과 같은 마이그레이션 후 작업을 수행한 후 애플리케이션을 해당 데이터베이스에 연결하고 다시 켤 수 있습니다.

## 연습 정리

이 연습에서 배포한 Azure Database for PostgreSQL은 다음 연습에서 사용됩니다. 이 연습 후에 서버를 중지할 수 있도록 요금이 발생합니다. 또는 **rg-learn-work-with-postgresql-eastus** 리소스 그룹을 삭제하여 이 연습의 일부로 배포한 모든 리소스를 제거할 수 있습니다. 즉, 다음 연습을 완료하려면 이 연습의 단계를 반복해야 합니다.

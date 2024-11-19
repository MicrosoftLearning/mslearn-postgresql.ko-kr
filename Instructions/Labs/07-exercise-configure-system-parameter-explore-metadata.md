---
lab:
  title: 시스템 매개 변수 구성 및 시스템 카탈로그와 보기를 통한 메타데이터 탐색
  module: Configure and manage Azure Database for PostgreSQL
---

# 시스템 매개 변수 구성 및 시스템 카탈로그와 보기를 통한 메타데이터 탐색

이 연습에서는 PostgreSQL에서 시스템 매개 변수와 메타데이터를 살펴봅니다.

## 시작하기 전에

> [!IMPORTANT]
> 이 모듈의 연습을 완료하려면 자체 Azure 구독이 필요합니다. Azure 구독이 없는 경우 [Azure 무료 계정으로 클라우드에서 빌드](https://azure.microsoft.com/free/)에서 무료 평가판 계정을 설정할 수 있습니다.

## 연습 환경 만들기

### Azure 구독에 리소스 배포

이 단계에서는 Azure Cloud Shell에서 Azure CLI 명령을 사용하여 리소스 그룹을 만들고 Bicep 스크립트를 실행하여 이 연습을 완료하는 데 필요한 Azure 서비스를 Azure 구독에 배포하는 방법을 안내합니다.

> 참고 항목
>
> 이 학습 경로의 여러 모듈을 수행하는 경우 같은 Azure 환경에서 모두 진행할 수 있습니다. 이 경우 이 리소스 배포 단계는 한 번만 완료하면 됩니다.

1. 웹 브라우저를 열고 [Azure Portal](https://portal.azure.com/)로 이동합니다.

2. Azure Portal 도구 모음의 **Cloud Shell** 아이콘을 선택하여 브라우저 창 아래쪽에 새 [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 창을 엽니다.

    ![Cloud Shell 아이콘을 빨간색 상자로 강조 표시한 Azure 도구 모음의 스크린샷.](media/07-portal-toolbar-cloud-shell.png)

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
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

### Azure Data Studio로 데이터베이스에 연결

1. 아직 수행하지 않은 경우 [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git) GitHub 리포지토리에서 랩 스크립트를 로컬로 복제합니다.
    1. 명령줄/터미널을 엽니다.
    1. 명령 실행:
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > 참고
       > 
       > **git**이 설치되지 않은 경우 [***git*** 앱을 다운로드하여 설치](https://git-scm.com/download)하고 이전 명령을 다시 실행해 봅니다.
1. Azure Data Studio를 아직 설치하지 않은 경우 [***Azure Data Studio***를 다운로드하여 설치](https://go.microsoft.com/fwlink/?linkid=2282284)합니다.
1. Azure Data Studio에서 **PostgreSQL** 확장을 설치하지 않은 경우 지금 설치합니다.
1. Azure Data Studio를 엽니다.
1. **연결**을 선택합니다.
1. **서버**를 선택하고 **새 연결**을 선택합니다.
1. **연결 형식**에서 **PostgreSQL**을 선택합니다.
1. **서버 이름**에 서버를 배포할 때 지정한 값을 입력합니다.
1. **사용자 이름**에 **pgAdmin**을 입력합니다.
1. **암호**에서 앞서 생성한 **pgAdmin** 로그인에 대해 임의로 생성된 암호를 입력합니다.
1. **암호 저장**을 선택합니다.
1. **연결**을 클릭합니다.
1. zoodb 데이터베이스를 아직 만들지 않은 경우 **파일**, **파일 열기**를 선택하고 스크립트를 저장한 폴더로 이동합니다. **../Allfiles/Labs/02/Lab2_ZooDb.sql**와 **열기**를 선택합니다.
   1. **DROP** 및 **CREATE** 문을 강조 표시하고 실행합니다.
   1. 화면 위쪽에서 드롭다운 화살표를 사용하여 zoodb 및 시스템 데이터베이스를 비롯한 서버의 데이터베이스를 표시합니다. **zoodb** 데이터베이스를 선택합니다.
   1. **테이블 만들기**, **외래 키 만들기**, **테이블 채우기** 섹션을 강조 표시하고 실행합니다.
   1. 스크립트의 끝부분에 있는 3개의 **SELECT** 문을 강조 표시하고 실행하여 테이블이 만들어지고 채워졌는지 확인합니다.

## 작업 1: PostgreSQL에서 진공 프로세스 살펴보기

1. 열리지 않으면 Azure Data Studio를 엽니다.
1. Azure Data Studio에서 **파일**, **파일 열기**를 선택한 다음 랩 스크립트로 이동합니다. **../Allfiles/Labs/07/Lab7_vacuum.sql**을 선택한 다음 **열기**를 선택합니다. 필요한 경우 서버에 다시 연결합니다.
1. 데이터베이스 드롭다운에서 **zoodb** 데이터베이스를 선택합니다.
1. **zoodb 데이터베이스 확인이 선택됨** 섹션을 강조 표시하고 실행합니다. 필요한 경우 드롭다운 목록을 사용하여 zoodb를 현재 데이터베이스로 만듭니다.
1. **데드 튜플 표시** 섹션을 강조 표시하고 실행합니다. 이 쿼리는 데이터베이스의 데드 및 라이브 튜플 수를 표시합니다. 데드 튜플 수를 기록해 둡니다.
1. **가중치 변경** 섹션을 강조 표시하고 연속으로 10회 실행합니다. 이 쿼리는 모든 동물의 가중치 열을 업데이트합니다.
1. **데드 튜플 표시** 아래 섹션을 다시 실행합니다. 업데이트가 완료된 후 데드 튜플 수를 기록해 둡니다.
1. **VACUUM을 수동으로 실행** 아래 섹션을 실행하여 진공 프로세스를 실행합니다.
1. **데드 튜플 표시** 아래 섹션을 다시 실행합니다. 진공 프로세스가 실행된 후 데드 튜플 수를 기록해 둡니다.

## 작업 2: 자동 진공 서버 매개 변수 구성

1. Azure Portal에서 Azure Database for PostgreSQL 유연한 서버로 이동합니다.
1. **설정** 아래에서 **서버 매개 변수**를 선택합니다.
1. 검색 창에 **`vacuum`** 을(를) 입력합니다. 다음 매개 변수를 찾고 다음과 같이 값을 변경합니다.
    1. autovacuum = ON(기본적으로 ON이어야 함)
    1. autovacuum_vacuum_scale_factor = 0.1
    1. autovacuum_vacuum_threshold = 50

    이는 테이블의 10%에 삭제로 표시된 행이 있거나 한 테이블에서 50개의 행이 업데이트되거나 삭제된 경우 자동 진공 프로세스를 실행하는 것과 같습니다.

1. **저장**을 선택합니다. 서버가 다시 시작됩니다.

## 작업 3: Azure Portal에서 PostgreSQL 메타데이터 보기

1. [Azure Portal](https://portal.azure.com)로 이동하여 로그인합니다.
1. **Azure Database for PostgreSQL**을 검색하여 선택합니다.
1. 이 연습을 위해 만든 Azure Database for PostgreSQL 유연한 서버를 선택합니다.
1. **모니터링**에서 **메트릭**을 선택합니다.
1. **메트릭**을 선택하고 **CPU 백분율**을 선택합니다.
1. 데이터베이스에 대한 다양한 메트릭을 볼 수 있습니다.

## 작업 4: 시스템 카탈로그 테이블에서 데이터 보기

1. Azure Data Studio로 전환합니다.
1. **서버**에서 PostgreSQL 서버를 선택하고 연결이 만들어지고 녹색 원이 서버에 표시될 때까지 기다립니다.
1. 서버를 마우스 오른쪽 단추로 클릭하고 **새 쿼리**를 선택합니다.
1. 다음 SQL을 입력하고 **실행**을 선택합니다.

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. 각 데이터베이스에 대한 커밋 및 롤백을 볼 수 있습니다.

## 시스템 를 사용하여 복잡한 메타데이터 쿼리 보기

1. 서버를 마우스 오른쪽 단추로 클릭하고 **새 쿼리**를 선택합니다.
1. 다음 SQL을 입력하고 **실행**을 선택합니다.

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. 많은 양의 통계 정보를 볼 수 있습니다.
1. 시스템 뷰를 사용하면 작성해야 하는 SQL의 복잡성을 줄일 수 있습니다. 이전 쿼리에 **pg_stats** 보기를 사용하지 않은 경우 다음 코드가 필요합니다.

    ```sql
    SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    a.attname,
    s.stainherit AS inherited,
    s.stanullfrac AS null_frac,
    s.stawidth AS avg_width,
    s.stadistinct AS n_distinct,
        CASE
            WHEN s.stakind1 = 1 THEN s.stavalues1
            WHEN s.stakind2 = 1 THEN s.stavalues2
            WHEN s.stakind3 = 1 THEN s.stavalues3
            WHEN s.stakind4 = 1 THEN s.stavalues4
            WHEN s.stakind5 = 1 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_vals,
        CASE
            WHEN s.stakind1 = 1 THEN s.stanumbers1
            WHEN s.stakind2 = 1 THEN s.stanumbers2
            WHEN s.stakind3 = 1 THEN s.stanumbers3
            WHEN s.stakind4 = 1 THEN s.stanumbers4
            WHEN s.stakind5 = 1 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_freqs,
        CASE
            WHEN s.stakind1 = 2 THEN s.stavalues1
            WHEN s.stakind2 = 2 THEN s.stavalues2
            WHEN s.stakind3 = 2 THEN s.stavalues3
            WHEN s.stakind4 = 2 THEN s.stavalues4
            WHEN s.stakind5 = 2 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS histogram_bounds,
        CASE
            WHEN s.stakind1 = 3 THEN s.stanumbers1[1]
            WHEN s.stakind2 = 3 THEN s.stanumbers2[1]
            WHEN s.stakind3 = 3 THEN s.stanumbers3[1]
            WHEN s.stakind4 = 3 THEN s.stanumbers4[1]
            WHEN s.stakind5 = 3 THEN s.stanumbers5[1]
            ELSE NULL::real
        END AS correlation,
        CASE
            WHEN s.stakind1 = 4 THEN s.stavalues1
            WHEN s.stakind2 = 4 THEN s.stavalues2
            WHEN s.stakind3 = 4 THEN s.stavalues3
            WHEN s.stakind4 = 4 THEN s.stavalues4
            WHEN s.stakind5 = 4 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_elems,
        CASE
            WHEN s.stakind1 = 4 THEN s.stanumbers1
            WHEN s.stakind2 = 4 THEN s.stanumbers2
            WHEN s.stakind3 = 4 THEN s.stanumbers3
            WHEN s.stakind4 = 4 THEN s.stanumbers4
            WHEN s.stakind5 = 4 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_elem_freqs,
        CASE
            WHEN s.stakind1 = 5 THEN s.stanumbers1
            WHEN s.stakind2 = 5 THEN s.stanumbers2
            WHEN s.stakind3 = 5 THEN s.stanumbers3
            WHEN s.stakind4 = 5 THEN s.stanumbers4
            WHEN s.stakind5 = 5 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS elem_count_histogram
    FROM pg_statistic s
     JOIN pg_class c ON c.oid = s.starelid
     JOIN pg_attribute a ON c.oid = a.attrelid AND a.attnum = s.staattnum
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE NOT a.attisdropped AND has_column_privilege(c.oid, a.attnum, 'select'::text) AND (c.relrowsecurity = false OR NOT row_security_active(c.oid));
    ```

## 연습 정리

1. 이 연습에서 배포한 Azure Database for PostgreSQL에는 요금이 발생하며, 이 연습 후에 서버를 삭제할 수 있습니다. 또는 **rg-learn-work-with-postgresql-eastus** 리소스 그룹을 삭제하여 이 연습의 일부로 배포한 모든 리소스를 제거할 수 있습니다.
1. 필요한 경우 .\DP3021Lab 폴더를 삭제합니다.

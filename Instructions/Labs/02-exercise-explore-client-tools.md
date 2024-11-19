---
lab:
  title: 클라이언트 도구를 사용하여 PostgreSQL 살펴보기
  module: Understand client-server communication in PostgreSQL
---

# 클라이언트 도구를 사용하여 PostgreSQL 살펴보기

이 연습에서는 psql 및 Azure Data Studio를 다운로드하고 설치합니다. 컴퓨터에 Azure Data Studio가 이미 설치되어 있는 경우 바로 Azure Database for PostgreSQL 유연한 서버 연결로 이동할 수 있습니다.

## 시작하기 전에

이 연습을 완료하려면 자체 Azure 구독이 필요합니다. Azure 구독이 아직 없는 경우 [Azure 평가판](https://azure.microsoft.com/free)을 만들 수 있습니다.

## 연습 환경 만들기

이 연습과 이후 모든 연습에서는 Azure Cloud Shell에서 Bicep을 사용하여 PostgreSQL 서버를 배포합니다.

### Azure 구독에 리소스 배포

이 단계에서는 Azure Cloud Shell에서 Azure CLI 명령을 사용하여 리소스 그룹을 만들고 Bicep 스크립트를 실행하여 이 연습을 완료하는 데 필요한 Azure 서비스를 Azure 구독에 배포하는 방법을 안내합니다.

> 참고 항목
>
> 이 학습 경로의 여러 모듈을 수행하는 경우 같은 Azure 환경에서 모두 진행할 수 있습니다. 이 경우 이 리소스 배포 단계는 한 번만 완료하면 됩니다.

1. 웹 브라우저를 열고 [Azure Portal](https://portal.azure.com/)로 이동합니다.

2. Azure Portal 도구 모음의 **Cloud Shell** 아이콘을 선택하여 브라우저 창 아래쪽에 새 [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 창을 엽니다.

    ![Cloud Shell 아이콘을 빨간색 상자로 강조 표시한 Azure 도구 모음의 스크린샷.](media/02-portal-toolbar-cloud-shell.png)

3. 메시지가 표시되는 경우 *Bash* 셸을 여는 데 필요한 옵션을 선택합니다. 이전에 *PowerShell* 콘솔을 사용한 적이 있다면 이것을 *Bash* 셸로 전환합니다.

4. Cloud Shell 프롬프트에서 다음을 입력하여 연습 리소스가 포함된 GitHub 리포지토리를 복제합니다.

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

5. 그 다음 Azure CLI 명령을 사용하여 Azure 리소스를 만들 때 중복 입력을 줄이기 위해 변수를 정의하는 세 가지 명령을 실행합니다. 이 변수는 리소스 그룹에 할당할 이름(`RG_NAME`), 리소스를 배포할 Azure 지역(`REGION`), PostgreSQL 관리자 로그인용 임의 생성 암호(`ADMIN_PASSWORD`)를 나타냅니다.

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

6. 두 개 이상의 Azure 구독에 액세스할 수 있고 기본 구독이 이 연습에 대한 리소스 그룹 및 기타 리소스를 만드는 데 사용할 구독이 아닌 경우, 이 명령을 실행할 때 적절한 구독을 설정하고 `<subscriptionName|subscriptionId>` 토큰을 사용할 구독의 이름 또는 ID로 바꿉니다.

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

7. 다음 Azure CLI 명령을 실행하여 리소스 그룹을 만듭니다.

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

8. 마지막으로 Azure CLI를 사용하여 Bicep 배포 스크립트를 실행하는 방법으로 리소스 그룹에 Azure 리소스를 프로비전합니다.

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

## PostgreSQL에 연결하는 클라이언트 도구

### psql을 사용하여 Azure Database for PostgreSQL에 연결

psql을 로컬로 설치하거나 Azure Portal에서 연결할 수 있습니다. 이 경우 Cloud Shell이 열리고 관리자 계정의 암호를 묻는 메시지가 표시됩니다.

#### 로컬로 연결

1. [여기](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893)에서 psql을 설치합니다.
    1. 설치 마법사에서 **구성 요소 선택** 대화 상자에 도달하면 **명령줄 도구**를 선택합니다.
    > 참고 항목
    >
    > **psql**이 사용자 환경에 이미 설치되어 있는지 확인하려면 명령줄/터미널을 열고 ***psql*** 명령을 실행합니다. "*psql: 오류: 소켓의 서버에 연결...*"과 같은 메시지가 반환된다면 **psql** 도구가 사용자 환경에 이미 설치되어 있으며 다시 설치할 필요가 없다는 뜻입니다.



1. 명령줄을 불러옵니다.
1. 서버에 연결하는 구문은 다음과 같습니다.

    ```sql
    psql --h <servername> --p <port> -U <username> <dbname>
    ```

1. 명령 프롬프트에 **`--host=<servername>.postgres.database.azure.com`**(이)라고 입력합니다. 여기서 `<servername>` 부분은 위에서 만든 Azure Database for PostgreSQL의 이름입니다.
    1. Azure Portal의 **개요** 또는 bicep 스크립트의 출력에서 서버 이름을 찾을 수 있습니다.

    ```sql
   psql -h <servername>.postgres.database.azure.com -p 5432 -U pgAdmin postgres
    ```

    1. 위에서 복사한 관리자 계정의 암호를 입력하라는 메시지가 표시됩니다.

1. 프롬프트에서 빈 데이터베이스를 만들려면 다음을 입력합니다.

    ```sql
    CREATE DATABASE mypgsqldb;
    ```

1. 프롬프트에서 다음 명령을 실행하여 새로 만든 **mypgsqldb** 데이터베이스에 대한 연결로 전환합니다.

    ```sql
    \c mypgsqldb
    ```

1. 이제 서버에 연결하고 데이터베이스를 만들었으므로 데이터베이스에 테이블 만들기와 같은 친숙한 SQL 쿼리를 실행할 수 있습니다.

    ```sql
    CREATE TABLE inventory (
        id serial PRIMARY KEY,
        name VARCHAR(50),
        quantity INTEGER
        );
    ```

1. 테이블에 데이터 로드

    ```sql
    INSERT INTO inventory (id, name, quantity) VALUES (1, 'banana', 150);
    INSERT INTO inventory (id, name, quantity) VALUES (2, 'orange', 154);
    ```

1. 테이블의 데이터 쿼리 및 업데이트

    ```sql
    SELECT * FROM inventory;
    ```

1. 테이블의 데이터를 업데이트합니다.

    ```sql
    UPDATE inventory SET quantity = 200 WHERE name = 'banana';
    ```

## Azure Data Studio 설치

> 참고 항목
>
> Azure Data Studio가 이미 설치된 경우 *PostgreSQL 확장 설치* 단계로 이동합니다.

Azure Database for PostgreSQL에서 사용할 Azure Data Studio를 설치하려면 다음을 수행합니다.

1. 브라우저에서 [Azure Data Studio 다운로드 및 설치](https://go.microsoft.com/fwlink/?linkid=2282284)로 이동하고 Windows 플랫폼에서 **사용자 설치 관리자(권장)** 를 선택합니다. 실행 파일이 다운로드 폴더에 다운로드됩니다.
1. **폴더 열기**를 선택합니다.
1. 사용권 계약이 표시됩니다. **사용 약관을 읽고 이에 동의**한 후 **다음**을 선택합니다.
1. **추가 작업 선택**에서 **PATH에 추가** 및 필요한 기타 추가 항목을 선택합니다. **다음**을 선택합니다.
1. **설치 준비** 대화 상자가 표시됩니다. 설정을 검토합니다. **뒤로**를 선택하여 변경하거나 **설치**를 선택합니다.
1. **Azure Data Studio 설치 마법사 완료** 대화 상자가 표시됩니다. **마침**을 선택합니다. Azure Data Studio가 시작됩니다.

## PostgreSQL 확장 설치

1. Azure Data Studio가 열려 있지 않으면 엽니다.
2. 왼쪽 메뉴에서 **확장**을 선택하여 확장 패널을 표시합니다.
3. 검색 창에 **PostgreSQL**을 입력합니다. Azure Data Studio에 대한 PostgreSQL 확장 아이콘이 표시됩니다.
   
![Azure Data Studio용 PostgreSQL 확장의 스크린샷](media/02-postgresql-extension.png)
   
4. **설치**를 선택합니다. 확장이 설치됩니다.

## Azure Database for PostgreSQL 유연한 서버에 연결

1. Azure Data Studio가 열려 있지 않으면 엽니다.
2. 왼쪽 메뉴에서 **연결**을 선택합니다.
   
![Azure Data Studio의 연결을 보여 주는 스크린샷](media/02-connections.png)

3. **새 연결**을 선택합니다.
   
![Azure Data Studio에서 새 연결을 만드는 스크린샷](media/02-create-connection.png)

4. **연결 세부 정보** 아래 **연결 유형**의 드롭다운 목록에서 **PostgreSQL**을 선택합니다.
5. **서버 이름**에 Azure Portal에 표시되는 전체 서버 이름을 입력합니다.
6. **인증 유형**에서 암호를 그대로 둡니다.
7. 사용자 이름 및 암호에 사용자 이름으로는 **pgAdmin**, 암호로는 **위에서 만든 임의 관리자 암호**를 입력합니다.
8. [ x ] 암호 저장을 선택합니다.
9. 나머지 필드는 선택 사항입니다.
10. **연결**을 선택합니다. Azure Database for PostgreSQL 서버에 연결되어 있습니다.
11. 서버 데이터베이스 목록이 표시됩니다. 여기에는 시스템 데이터베이스 및 사용자 데이터베이스가 포함됩니다.

## Zoo 데이터베이스 만들기

1. 연습 스크립트 파일이 있는 폴더로 이동하거나 [MSLearn PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/02)에서 **Lab2_ZooDb.sql**을 다운로드합니다.
1. Azure Data Studio가 열려 있지 않으면 엽니다.
1. **파일**, **파일 열기**를 선택하고 스크립트를 저장한 폴더로 이동합니다. **../Allfiles/Labs/02/Lab2_ZooDb.sql**와 **열기**를 선택합니다.
   1. **DROP** 및 **CREATE** 문을 강조 표시하고 실행합니다.
   1. 화면 위쪽에서 드롭다운 화살표를 사용하여 zoodb 및 시스템 데이터베이스를 비롯한 서버의 데이터베이스를 표시합니다. **zoodb** 데이터베이스를 선택합니다.
   1. **테이블 만들기**, **외래 키 만들기**, **테이블 채우기** 섹션을 강조 표시하고 실행합니다.
   1. 스크립트의 끝부분에 있는 3개의 **SELECT** 문을 강조 표시하고 실행하여 테이블이 만들어지고 채워졌는지 확인합니다.

---
lab:
  title: Azure Database for PostgreSQL에서 저장 프로시저 만들기
  module: Procedures and functions in PostgreSQL
---

# Azure Database for PostgreSQL에서 저장 프로시저 만들기

이 연습에서는 몇 가지 저장 프로시저를 생성하고 실행합니다.

## 시작하기 전에

이 연습을 완료하려면 자체 Azure 구독이 필요합니다. Azure 구독이 아직 없는 경우 [Azure 평가판](https://azure.microsoft.com/free)을 만들 수 있습니다.

또한 컴퓨터에 다음이 설치되어 있어야 합니다.

- Visual Studio Code
- Microsoft의 Postgres Visual Studio Code 확장.
- Azure CLI
- Git

## 연습 환경 만들기

이 연습과 이후 연습에서는 Bicep 스크립트를 사용하여 Azure Database for PostgreSQL - 유연한 서버 및 기타 리소스를 Azure 구독에 배포합니다. Bicep 스크립트는 앞서 복제한 GitHub 리포지토리의 `/Allfiles/Labs/Shared` 폴더에 있습니다.

### Visual Studio Code 및 PostgreSQL 확장을 다운로드하여 설치합니다.

Visual Studio Code가 설치되어 있지 않은 경우:

1. 브라우저에서 [Visual Studio Code 다운로드](https://code.visualstudio.com/download)로 이동하여 운영 체제에 맞는 버전을 선택합니다.

1. 사용 중인 운영 체제의 설치 지침을 따르세요.

1. Visual Studio Code를 엽니다.

1. 왼쪽 메뉴에서 **확장**을 선택하여 확장 패널을 표시합니다.

1. 검색 창에 **PostgreSQL**을 입력합니다. Visual Studio Code용 PostgreSQL 확장 아이콘이 표시됩니다. Microsoft에서 제공하는 것을 선택해야 합니다.

1. **설치**를 선택합니다. 확장이 설치됩니다.

### Azure CLI 및 Git 다운로드 및 설치

Azure CLI 또는 Git이 설치되어 있지 않은 경우:

1. 브라우저에서 [Azure CLI 설치](https://learn.microsoft.com/cli/azure/install-azure-cli)로 이동하여 운영 체제별 지침을 따릅니다.

1. 브라우저에서 [Git 다운로드 및 설치](https://git-scm.com/downloads)로 이동하여 운영 체제별 지침을 따릅니다.

### 연습 파일 다운로드

연습 파일이 포함된 GitHub 리포지토리를 이미 복제한 경우 *연습 파일 다운로드 건너뛰기*로 이동합니다.

연습 파일을 다운로드하려면 연습 파일이 포함된 GitHub 리포지토리를 로컬 컴퓨터에 복제합니다. 리포지토리에는 이 연습을 완료하는 데 필요한 모든 스크립트와 리소스가 포함되어 있습니다.

1. 아직 열려 있지 않은 경우 Visual Studio Code를 엽니다.

1. 명령 팔레트를 열려면 **모든 명령 표시**(Ctrl+Shift+P)를 선택합니다.

1. 명령 팔레트에서 **Git: Clone**을 검색하고 선택합니다.

1. 명령 팔레트에서 다음을 입력하여 연습 리소스가 포함된 GitHub 리포지토리를 복제하고 **Enter** 키를 누릅니다.

    ```bash
    https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. 프롬프트에 따라 리포지토리를 복제할 폴더를 선택합니다. 리포지토리는 선택한 위치의 `mslearn-postgresql`라는 폴더에 복제됩니다.

1. 복제된 리포지토리를 열지 묻는 메시지가 표시되면 **열기**를 선택합니다. 리포지토리는 Visual Studio Code에서 열립니다.

### Azure 구독에서 리소스 배포

Azure 리소스가 이미 설치되어 있는 경우 *리소스 배포 건너뛰기*로 이동합니다.

이 단계에서는 Visual Studio Code에서 Azure CLI 명령을 사용하여 리소스 그룹을 만들고 Bicep 스크립트를 실행하여 이 연습을 완료하는 데 필요한 Azure 서비스를 Azure 구독에 배포하는 방법을 안내합니다.

> &#128221; 이 학습 경로에서 여러 모듈을 수행하는 경우 모듈 간에 Azure 환경을 공유할 수 있습니다. 이 경우 이 리소스 배포 단계는 한 번만 완료하면 됩니다.

1. 아직 열려 있지 않은 경우 Visual Studio Code를 열고 GitHub 리포지토리를 복제한 폴더를 엽니다.

1. 탐색기 창에서 **mslearn-postgresql** 폴더를 펼칩니다.

1. **Allfiles/Labs/Shared** 폴더를 확장합니다.

1. **Allfiles/Labs/Shared** 폴더를 마우스 오른쪽 단추로 클릭하고 **통합 터미널에서 열기**를 선택합니다. 이 선택 영역은 Visual Studio Code 창에서 터미널 창을 엽니다.

1. 터미널은 기본적으로 **PowerShell** 창을 열 수도 있습니다. 랩의 이 섹션에서는 **Bash 셸**을 사용하려고 합니다. **+** 아이콘 외에도 드롭다운 화살표가 있습니다. 이를 선택하고 사용 가능한 프로필 목록에서 **Git Bash** 또는 **Bash**를 선택합니다. 이 선택 영역은 **Bash 셸**이 있는 새 터미널 창을 엽니다.

    > &#128221; 원하는 경우 **PowerShell** 터미널 창을 닫을 수 있지만 필요하지는 않습니다. 여러 터미널 창을 동시에 열 수 있습니다.

1. 터미널 창에서 다음 명령을 실행하여 Azure 계정에 로그인합니다.

    ```bash
    az login
    ```

    이 명령은 Azure 계정에 로그인하라는 메시지를 표시하는 새 브라우저 창을 엽니다. 로그인한 후 터미널 창으로 돌아갑니다.

1. 다음으로, Azure CLI 명령을 사용하여 Azure 리소스를 만들 때 중복 입력을 줄이기 위해 변수를 정의하는 세 가지 명령을 실행합니다. 이 변수는 리소스 그룹에 할당할 이름(`RG_NAME`), 리소스가 배포되는 Azure 지역(`REGION`), PostgreSQL 관리자 로그인용 임의 생성 암호(`ADMIN_PASSWORD`)를 나타냅니다.

    첫 번째 명령에서는 해당 변수에 `eastus` 지역을 할당하지만, 원하는 위치로 바꿀 수도 있습니다.

    ```bash
    REGION=eastus
    ```

    다음 명령에서는 이 연습에서 사용할 모든 리소스를 저장하는 리소스 그룹에 사용할 이름을 할당합니다. 해당 변수에는 `rg-learn-work-with-postgresql-$REGION` 리소스 그룹 이름을 할당하며, 여기서 `$REGION` 부분은 이전에 지정한 위치입니다. *그러나 선호도에 맞거나 이미 가지고 있을 수 있는 다른 리소스 그룹 이름으로 변경할 수 있습니다*.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    마지막 명령은 PostgreSQL 관리자 로그인을 위한 암호를 무작위로 생성합니다. 나중에 PostgreSQL 유연한 서버에 연결하는 데 사용하기 위해 이 암호를 안전한 장소에 복사해 두어야 합니다.

    ```bash
    #!/bin/bash
    
    # Define array of allowed characters explicitly
    chars=( {a..z} {A..Z} {0..9} '!' '@' '#' '$' '%' '^' '&' '*' '(' ')' '_' '+' )
    
    a=()
    for ((i = 0; i < 100; i++)); do
        rand_char=${chars[$RANDOM % ${#chars[@]}]}
        a+=("$rand_char")
    done
    
    # Join first 18 characters without delimiter
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]:0:18}")
    
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo "$ADMIN_PASSWORD"
    echo "Please copy it to a safe place, as you will need it later to connect to your PostgreSQL flexible server."
    ```

1. (기본 구독을 사용하는 경우에는 건너뜁니다.) 두 개 이상의 Azure 구독에 접근할 수 있고, 현재 기본 구독이 이번 연습에서 리소스 그룹 및 기타 리소스를 만들고자 하는 구독이 *아니라면,* 아래 명령어를 실행하여 원하는 구독을 설정합니다. 이때 `<subscriptionName|subscriptionId>` 토큰은 사용하려는 구독의 이름이나 ID로 바꾸어 입력해야 합니다.

    ```azurecli
    az account set --subscription 16b3c013-d300-468d-ac64-7eda0820b6d3
    ```

1. (기존 리소스 그룹을 사용 중인 경우 건너뛰기) 다음 Azure CLI 명령을 실행하여 리소스 그룹을 만듭니다.

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. 마지막으로 Azure CLI를 사용하여 Bicep 배포 스크립트를 실행하는 방법으로 리소스 그룹에 Azure 리소스를 프로비전합니다.

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep 배포 스크립트는 이 연습을 완료하는 데 필요한 Azure 서비스를 리소스 그룹으로 프로비전합니다. 배포되는 리소스는 Azure Database for PostgreSQL- 유연한 서버입니다. 이 bicep 스크립트는 데이터베이스도 만듭니다. 이는 명령줄에서 매개 변수로 구성할 수 있습니다.

    배포를 완료하려면 보통 몇 분 정도 걸립니다. bash 터미널에서 모니터링하거나 이전에 만든 리소스 그룹의 **배포** 페이지로 이동하여 거기서 배포 진행 상황을 관찰할 수 있습니다.

1. 스크립트는 PostgreSQL 서버의 임의의 이름을 생성하므로 다음 명령을 실행하여 서버의 이름을 찾을 수 있습니다.

    ```azurecli
    az postgres flexible-server list --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" --output table
    ```

    나중에 이 연습에서 서버에 연결할 때 필요하므로 서버의 이름을 적어 둡니다.

    > &#128221; Azure Portal에서 서버 이름을 찾을 수도 있습니다. Azure Portal에서 **리소스 그룹**으로 이동하여 이전에 만든 리소스 그룹을 선택합니다. PostgreSQL 서버는 리소스 그룹에 나열됩니다.

### 배포 오류 문제 해결

Bicep 배포 스크립트를 실행할 때 몇 가지 오류가 발생할 수 있습니다. 가장 일반적인 메시지와 해결 단계는 다음과 같습니다.

- 이전에 이 학습 경로에 대한 Bicep 배포 스크립트를 실행한 후 리소스를 삭제한 경우, 리소스를 삭제한 후 48시간 이내에 스크립트를 다시 실행하려고 하면 다음과 같은 오류 메시지가 표시될 수 있습니다.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    이 메시지가 표시되면 이전 `azure deployment group create` 명령을 수정하여 `restore` 매개 변수를 `true`(으)로 설정하고 다시 실행합니다.

- 선택한 지역이 특정 리소스를 프로비전하지 못하도록 제한된 경우 `REGION` 변수를 다른 위치로 설정하고 명령을 다시 실행하여 리소스 그룹을 만들고 Bicep 배포 스크립트를 실행해야 합니다.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 랩에 AI 리소스가 필요한 경우 다음과 같은 오류가 발생할 수 있습니다. 이 오류는 스크립트가 책임 있는 AI 계약에 동의해야 하기 때문에 AI 리소스를 만들 수 없을 때 발생합니다. 이 경우 Azure Portal 사용자 인터페이스를 사용하여 Azure AI 서비스 리소스를 만든 다음 배포 스크립트를 다시 실행합니다.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Visual Studio Code에서 PostgreSQL 확장에 연결

이 섹션에서는 Visual Studio Code의 PostgreSQL 확장을 사용하여 PostgreSQL 서버에 연결합니다. PostgreSQL 확장을 사용하여 PostgreSQL 서버에 대해 SQL 스크립트를 실행합니다.

1. 아직 열리지 않은 경우 Visual Studio Code를 열고 GitHub 리포지토리를 복제한 폴더를 엽니다.

1. 왼쪽 메뉴에서 **PostgreSQL** 아이콘을 선택합니다.

    > &#128221; PostgreSQL 아이콘이 보이지 않는 경우, **확장** 아이콘을 선택한 다음 **PostgreSQL**을 검색합니다. Microsoft의 **PostgreSQL** 확장을 선택하고 **설치**를 선택합니다.

1. 이미 PostgreSQL 서버에 대한 연결을 만들었다면 다음 단계로 건너뜁니다. 새 연결을 만들려면 다음과 같이 합니다.

    1. **PostgreSQL** 확장에서 **+ 연결 추가**를 선택하여 새 연결을 추가합니다.

    1. **새 연결** 대화 상자에서 다음 정보를 입력합니다.

        - **서버 이름**: `<your-server-name>`.postgres.database.azure.com
        - **인증 유형**: 암호
        - **사용자 이름**: pgAdmin
        - **암호**: 이전에 생성한 임의의 암호입니다.
        - **비밀번호 저장** 확인란을 선택합니다.
        - **연결 이름**: `<your-server-name>`

    1. 연결을 테스트하려면 **연결 테스트**를 선택합니다. 연결에 성공하면 **저장 후 연결**을 선택하여 연결을 저장하고, 성공하지 않으면 연결 정보를 검토한 후 다시 시도합니다.

1. 아직 연결되지 않은 경우, PostgreSQL 서버에 대해 **연결**을 선택합니다. Azure Database for PostgreSQL 서버에 연결되어 있습니다.

1. 서버 노드 및 해당 데이터베이스를 확장합니다. 기존 데이터베이스가 나열됩니다.

1. zoodb 데이터베이스를 아직 만들지 않았다면 **파일**, **파일 열기**를 선택하고 스크립트를 저장한 폴더로 이동합니다. **../Allfiles/Labs/02/Lab2_ZooDb.sql**와 **열기**를 선택합니다.

1. Visual Studio Code의 오른쪽 하단에서 연결이 녹색인지 확인합니다. 그렇지 않은 경우 **PGSQL 연결 끊김**이라고 표시되어야 합니다. **PGSQL 연결 끊김** 텍스트를 선택한 다음 명령 팔레트의 목록에서 PostgreSQL 서버 연결을 선택합니다. 비밀번호를 묻는 메시지가 표시되면 이전에 생성한 비밀번호를 입력합니다.

1. 데이터베이스를 만들 시간입니다.

    1. **DROP** 및 **CREATE** 문을 강조 표시하고 실행합니다.

    1. **SELECT current_database()** 문만 강조 표시하고 실행하면 데이터베이스가 현재 `postgres`(으)로 설정되어 있는 것을 확인할 수 있습니다. `zoodb`(으)로 변경해야 합니다.

    1. 메뉴 모음에서 *실행* 아이콘이 있는 줄임표를 선택하고 **PostgreSQL 데이터베이스 변경**을 선택합니다. 데이터베이스 목록에서 `zoodb`을(를) 선택합니다.

        > &#128221; 쿼리 창에서 데이터베이스를 변경할 수도 있습니다. 쿼리 탭 자체에서 서버 이름과 데이터베이스 이름을 확인할 수 있습니다. 데이터베이스 이름을 선택하면 데이터베이스 목록이 표시됩니다. 목록에서 `zoodb` 데이터베이스를 선택합니다.

    1. **SELECT current_database()** 문을 다시 실행하여 데이터베이스가 이제 `zoodb`(으)로 설정되었는지 확인합니다.

    1. **테이블 만들기**, **외래 키 만들기**, **테이블 채우기** 섹션을 강조 표시하고 실행합니다.

    1. 스크립트의 끝부분에 있는 3개의 **SELECT** 문을 강조 표시하고 실행하여 테이블이 만들어지고 채워졌는지 확인합니다.

## repopulate_zoo() 저장 프로시저를 만듭니다.

이 섹션에서는 `repopulate_zoo()` 저장 프로시저를 만듭니다. 이 절차는 동물원 데이터베이스를 데이터로 다시 채우는 데 사용됩니다. 이 절차는 테이블의 모든 데이터를 잘라내어 삭제한 다음 새 데이터로 채웁니다.

1. Visual Studio Code 창에서 **파일**, **파일 열기**를 선택한 다음 랩 스크립트로 이동합니다. **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql**을 선택한 다음 **열기**를 선택합니다. 필요한 경우 **PGSQL 연결 끊김** 텍스트를 선택한 다음 명령 팔레트의 목록에서 PostgreSQL 서버 연결을 선택하여 서버에 다시 연결합니다. 비밀번호를 묻는 메시지가 표시되면 이전에 생성한 비밀번호를 입력합니다.

1. **SELECT current_database()** 문을 실행하여 현재 데이터베이스를 확인합니다. 다시 말하지만, 데이터베이스는 `postgres`(으)로 설정되어 있을 가능성이 큽니다. 그렇다면 `zoodb`(으)로 변경해야 합니다. 메뉴 모음에서 *실행* 아이콘이 있는 줄임표를 선택하고 **PostgreSQL 데이터베이스 변경**을 선택합니다. 데이터베이스 목록에서 `zoodb`을(를) 선택합니다. **SELECT current_database()** 문을 실행하여 연결을 다시 테스트합니다.

1. **저장 프로시저 만들기**에서 **DROP PROCEDURE**에서 **END $$** 까지의 섹션을 강조 표시합니다. 강조 표시된 텍스트를 실행합니다.

1. 다음 섹션을 계속 진행하려면 Visual Studio Code를 열어 둡니다.

## new_exhibit() 저장 프로시저를 만듭니다.

이 섹션에서는 `new_exhibit()` 저장 프로시저를 만듭니다. 이 절차는 동물원 데이터베이스에 새 전시물을 추가하는 데 사용됩니다. 이 절차는 엔클로저 테이블에 새 행을 삽입한 다음 전시회의 각 동물에 대해 동물 테이블에 행을 삽입합니다.

1. Visual Studio Code에서 **파일**, **파일 열기**를 선택한 다음 랩 스크립트로 이동합니다. **../Allfiles/Labs/05/Lab5_StoredProcedure.sql**을 선택한 다음 **열기**를 선택합니다. 필요한 경우 **PGSQL 연결 끊김** 텍스트를 선택한 다음 명령 팔레트의 목록에서 PostgreSQL 서버 연결을 선택하여 서버에 다시 연결합니다. 비밀번호를 묻는 메시지가 표시되면 이전에 생성한 비밀번호를 입력합니다.

1. **SELECT current_database()** 문을 실행하여 현재 데이터베이스를 확인합니다. 다시 말하지만, 데이터베이스는 `postgres`(으)로 설정되어 있을 가능성이 큽니다. 그렇다면 `zoodb`(으)로 변경해야 합니다. 메뉴 모음에서 *실행* 아이콘이 있는 줄임표를 선택하고 **PostgreSQL 데이터베이스 변경**을 선택합니다. 데이터베이스 목록에서 `zoodb`을(를) 선택합니다. **SELECT current_database()** 문을 실행하여 연결을 다시 테스트합니다.

1. **CALL repopulate_zoo()** 문을 강조 표시하고 실행하여 정리된 데이터로 시작합니다.

1. **저장 프로시저 만들기**에서 **DROP PROCEDURE**에서 **END $$** 까지의 섹션을 강조 표시합니다. 강조 표시된 텍스트를 실행합니다. 프로시저를 자세히 읽습니다. 몇 가지 입력 매개 변수를 선언하고 이를 사용하여 엔클로저 테이블과 동물 테이블에 행을 삽입하는 것을 볼 수 있습니다.

1. 다음 섹션을 계속 진행하려면 Visual Studio Code를 열어 둡니다.

## 저장 프로시저

이제 `new_exhibit()` 저장 프로시저를 만들었으므로 이를 호출하여 동물원 데이터베이스에 새 전시물을 추가할 수 있습니다. 이 절차에는 전시 이름, 엔클로저 유형, 전시 동물 수 등 몇 가지 입력 매개 변수가 필요합니다.

1. **저장 프로시저 호출** 주석 다음 섹션을 강조 표시합니다. 강조 표시된 텍스트를 실행합니다. 이 스크립트는 입력 매개 변수에 값을 전달하여 저장 프로시저를 호출합니다.

1. 두 **SELECT** 문을 강조 표시하고 실행합니다. 강조 표시된 텍스트를 실행합니다. 새 행이 엔클로저에 삽입되고 동물에 5개의 새 행이 삽입되는 것을 확인할 수 있습니다.

## 테이블 반환 함수 만들기 및 호출

이제 테이블 반환 함수를 만들 차례입니다. 테이블 반환 함수는 테이블을 반환하는 사용자 정의 함수입니다. 일반 테이블처럼 `SELECT` 문에서 테이블 반환 함수를 사용할 수 있습니다.

1. Visual Studio Code에서 **파일**, **파일 열기**를 선택한 다음 랩 스크립트로 이동합니다. **../Allfiles/Labs/05/Lab5_Table_Function.sql**을 선택한 다음 **열기**를 선택합니다. 필요한 경우 **PGSQL 연결 끊김** 텍스트를 선택한 다음 명령 팔레트의 목록에서 PostgreSQL 서버 연결을 선택하여 서버에 다시 연결합니다. 비밀번호를 묻는 메시지가 표시되면 이전에 생성한 비밀번호를 입력합니다.

1. **SELECT current_database()** 문을 실행하여 현재 데이터베이스를 확인합니다. 다시 말하지만, 데이터베이스는 `postgres`(으)로 설정되어 있을 가능성이 큽니다. 그렇다면 `zoodb`(으)로 변경해야 합니다. 메뉴 모음에서 *실행* 아이콘이 있는 줄임표를 선택하고 **PostgreSQL 데이터베이스 변경**을 선택합니다. 데이터베이스 목록에서 `zoodb`을(를) 선택합니다. **SELECT current_database()** 문을 실행하여 연결을 다시 테스트합니다.

1. 데이터 정리로 시작하려면 **CALL repopulate_zoo()** 저장 프로시저를 강조 표시하고 실행합니다.

1. **테이블 반환 함수 만들기** 주석 뒤에 있는 스크립트를 강조 표시하고 실행합니다. 이 함수는 **enclosure_summary**라는 테이블을 반환합니다. 함수 코드를 읽고 테이블이 채워지는 방식을 이해합니다.

1. 매번 다른 엔클로저 ID를 전달하여 두 개의 select 문을 강조 표시하고 실행합니다.

1. **How to use a table valued function with a LATERAL join** 주석 다음에 있는 스크립트를 강조 표시하고 실행합니다. 이 스크립트는 조인에서 표 이름 대신 사용되는 테이블 반환 함수를 보여 줍니다.

## 기본 제공 함수

이 섹션에서는 PostgreSQL에서 사용할 수 있는 몇 가지 기본 제공 함수를 살펴봅니다. PostgreSQL에는 데이터에 대한 다양한 작업을 수행하는 데 사용할 수 있는 다양한 기본 제공 함수가 있습니다. 이러한 함수는 SQL 쿼리에서 데이터를 조작하고 분석하는 데 사용할 수 있습니다.

1. Visual Studio Code에서 **파일**, **파일 열기**를 선택한 다음 랩 스크립트로 이동합니다. **../Allfiles/Labs/05/Lab5_SimpleFunctions.sql**을 선택한 다음 **열기**를 선택합니다. 필요한 경우 **PGSQL 연결 끊김** 텍스트를 선택한 다음 명령 팔레트의 목록에서 PostgreSQL 서버 연결을 선택하여 서버에 다시 연결합니다. 비밀번호를 묻는 메시지가 표시되면 이전에 생성한 비밀번호를 입력합니다.

> &#128221; 이 스크립트의 함수는 동물원 데이터베이스에만 국한되지 않습니다. 이 함수는 모든 데이터베이스에서 사용할 수 있는 일반적인 PostgreSQL 함수입니다. `postgres` 데이터베이스를 포함한 모든 데이터베이스에서 실행할 수 있습니다.

1. 각 함수를 강조 표시하고 실행하여 작동 방식을 확인합니다. 자세한 내용은 [온라인 문서](https://www.postgresql.org/docs/current/functions.html) 아티클에서 각 함수에 대한 정보를 확인합니다.

## 정리

1. 다른 연습에 이 PostgreSQL 서버가 더 이상 필요하지 않은 경우, 불필요한 Azure 비용이 발생하지 않도록 이 연습에서 만든 리소스 그룹을 삭제합니다.

1. PostgreSQL 서버를 계속 실행하려면 실행 상태로 두면 됩니다. 서버를 계속 실행하지 않으려면 bash 터미널에서 서버를 중지하여 불필요한 비용이 발생하지 않도록 할 수 있습니다. 서버를 중지하려면 다음 명령을 실행합니다.

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    `<your-server-name>`을(를) PostgreSQL 서버의 이름으로 바꿉니다.

    > &#128221; Azure Portal에서 서버를 중지할 수도 있습니다. Azure Portal에서 **리소스 그룹**으로 이동하여 이전에 만든 리소스 그룹을 선택합니다. PostgreSQL 서버를 선택한 다음 메뉴에서 **중지**를 선택합니다.

1. 필요한 경우 이전에 복제한 git 리포지토리를 삭제합니다.

이 연습을 성공적으로 완료했습니다. 몇 가지 저장 프로시저를 생성하고 실행했습니다. 또한 테이블 반환 함수를 만들고 PostgreSQL에서 사용할 수 있는 몇 가지 기본 제공 함수를 살펴봤습니다.

이제 이러한 저장 프로시저와 함수를 자체 PostgreSQL 데이터베이스에서 사용하여 데이터에 대한 다양한 작업을 수행할 수 있습니다.

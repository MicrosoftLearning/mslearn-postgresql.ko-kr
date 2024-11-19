---
lab:
  title: 온라인 PostgreSQL 데이터베이스 마이그레이션
  module: Migrate to Azure Database for PostgreSQL Flexible Server
---

## 온라인 PostgreSQL 데이터베이스 마이그레이션

이 연습에서는 온라인 마이그레이션 작업을 수행할 수 있도록 원본 PostgreSQL 서버와 Azure Database for PostgreSQL 유연한 서버 간에 논리적 복제를 구성합니다.

## 시작하기 전에

이 연습을 완료하려면 자체 Azure 구독이 필요합니다. Azure 구독이 아직 없는 경우 [Azure 평가판](https://azure.microsoft.com/free)을 만들 수 있습니다.

> **참고:**: 이 연습에서는 마이그레이션의 원본으로 사용하는 서버가 데이터베이스를 연결하고 마이그레이션할 수 있도록 Azure Database for PostgreSQL 유연한 서버에 액세스할 수 있어야 합니다. 이렇게 하려면 공용 IP 주소 및 포트를 통해 원본 서버에 액세스할 수 있어야 합니다. [Azure IP 범위 및 서비스 태그 - 공용 클라우드](https://www.microsoft.com/en-gb/download/details.aspx?id=56519)에서 Azure 지역 IP 주소 목록을 다운로드하여 사용하는 Azure 지역에 따라 방화벽 규칙에서 허용되는 IP 주소 범위를 최소화하는 데 도움을 줍니다.

서버의 방화벽을 열어 Azure Database for PostgreSQL 유연한 서버의 마이그레이션 기능이 기본적으로 TCP 포트 5432인 원본 PostgreSQL 서버에 액세스할 수 있도록 합니다.
>
원본 데이터베이스 앞에 방화벽 어플라이언스를 사용하는 경우, Azure Database for PostgreSQL 유연한 서버의 마이그레이션 기능에서 마이그레이션을 위해 원본 데이터베이스에 액세스할 수 있도록 방화벽 규칙을 추가해야 할 수 있습니다.
>
> 마이그레이션에 지원되는 PostgreSQL의 최대 버전은 버전 16입니다.

### 필수 조건

> **참고**: 이 연습은 이전 연습의 활동을 기반으로 하므로 이 연습을 시작하기 전에 원본 및 대상 데이터베이스를 준비하여 논리적 복제를 구성하려면 이전 연습을 완료해야 합니다.

## 게시 만들기 - 원본 서버

1. PGAdmin을 열고 데이터 동기화를 위해 Azure Database for PostgreSQL 유연한 서버의 원본으로 작동할 데이터베이스를 포함한 원본 서버에 연결합니다.
1. 동기화하려는 데이터를 사용하여 원본 데이터베이스에 연결된 새 쿼리 창을 엽니다.
1. 데이터 게시를 허용하도록 원본 서버 wal_level을 **논리적**으로 구성합니다.
    1. PostgreSQL 설치 디렉터리 내 bin 디렉터리에서 **postgresql.conf** 파일을 찾아서 엽니다.
    1. 구성 설정 **wal_level**이 있는 줄을 찾습니다.
    1. 해당 줄의 주석을 제거하고 값을 **논리적**으로 설정합니다.
    1. 파일을 저장 후 닫습니다.
    1. PostgreSQL 서비스를 다시 시작합니다.
1. 이제 데이터베이스 내의 모든 테이블을 포함할 게시를 구성합니다.

    ```SQL
    CREATE PUBLICATION migration1 FOR ALL TABLES;
    ```

## 구독 만들기 - 대상 서버

1. PGAdmin을 열고 원본 서버에서 데이터 동기화의 대상으로 사용할 데이터베이스가 포함된 Azure Database for PostgreSQL 유연한 서버에 연결합니다.
1. 동기화하려는 데이터를 사용하여 원본 데이터베이스에 연결된 새 쿼리 창을 엽니다.
1. 원본 서버에 대한 구독을 만듭니다.

    ```sql
    CREATE SUBSCRIPTION migration1
    CONNECTION 'host=<source server name> port=<server port> dbname=adventureworks application_name=migration1 user=<username> password=<password>'
    PUBLICATION migration1
    WITH(copy_data = false)
    ;    
    ```

1. 테이블 복제 상태를 확인합니다.

    ```SQL
    SELECT PT.schemaname, PT.tablename,
        CASE PS.srsubstate
            WHEN 'i' THEN 'initialize'
            WHEN 'd' THEN 'data is being copied'
            WHEN 'f' THEN 'finished table copy'
            WHEN 's' THEN 'synchronized'
            WHEN 'r' THEN ' ready (normal replication)'
            ELSE 'unknown'
        END AS replicationState
    FROM pg_publication_tables PT,
            pg_subscription_rel PS
            JOIN pg_class C ON (C.oid = PS.srrelid)
            JOIN pg_namespace N ON (N.oid = C.relnamespace)
    ;
    ```

## 데이터 복제 테스트

1. 원본 서버에서 workorder 테이블의 행 수를 확인합니다.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. 대상 서버에서 workorder 테이블의 행 수를 확인합니다.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. 행 수 값이 일치하는지 확인합니다.
1. 이제 리포지토리 [여기](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/11)에서 C:\로 Lab11_workorder.csv 파일을 다운로드합니다.
1. 다음 명령을 사용하여 CSV에서 원본 서버의 workorder 테이블에 새 데이터를 로드합니다.

    ```Bash
    psql --host=localhost --port=5432 --username=postgres --dbname=adventureworks --command="\COPY production.workorder FROM 'C:\Lab11_workorder.csv' CSV HEADER"
    ```

명령의 출력은 `COPY 490`이(가) 되어야 하며, 이는 CSV 파일의 테이블에 490개의 행이 추가로 기록되었음을 나타냅니다.

1. 원본(72,591행)과 대상의 workorder 테이블 행 수가 일치하는지 확인하여 데이터 복제가 제대로 작동하고 있는지 확인합니다.

## 연습 정리

이 연습에서 배포한 Azure Database for PostgreSQL에는 요금이 발생하며, 이 연습 후에 서버를 삭제할 수 있습니다. 또는 **rg-learn-work-with-postgresql-eastus** 리소스 그룹을 삭제하여 이 연습의 일부로 배포한 모든 리소스를 제거할 수 있습니다.

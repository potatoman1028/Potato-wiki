---
type: 지식
created: 2026-08-20
updated: 2026-08-21
tags: [MSSQL, SQLServer, InMemoryOLTP, Windows, 개발환경]
aliases: [SQL Server 로컬 DB 구성, MEMORY_OPTIMIZED_FILEGROUP 설정]
source: ChatGPT 대화 정리
---

# MSSQL 로컬 개발 DB 구성과 메모리 최적화 설정

## 요약

- `MEMORY_OPTIMIZED = ON`인 테이블이나 사용자 정의 테이블 형식을 만들려면 데이터베이스에 `MEMORY_OPTIMIZED_DATA` 파일 그룹과 컨테이너가 하나 이상 있어야 한다.
- 로컬 개발 DB는 데이터, 로그, 메모리 최적화 컨테이너 경로를 분리해 두면 관리하기 쉽다.
- SQL Server 2019의 제품 주 버전은 `15`이며 호환성 수준은 `150`이다. SQL Server 2022는 `16`, 호환성 수준은 `160`이다.
- `CREATE DATABASE`까지 성공하고 이후 `SET COMPATIBILITY_LEVEL`에서 실패하면 DB는 이미 생성되어 있을 수 있다. 이때 삭제하지 말고 호환성 수준만 서버 버전에 맞게 변경하면 된다.
- 공개 저장소에는 실제 계정 비밀번호와 전체 접속 문자열을 기록하지 않는다.

## 내용

## 1. 오류 원인

다음 오류는 메모리 최적화 테이블을 만들려고 했지만 대상 DB에 필요한 파일 그룹이나 컨테이너가 없을 때 발생한다.

```text
메모리 최적화 테이블을 만들 수 없습니다.
메모리 최적화 테이블을 만들려면 데이터베이스가 온라인이고
데이터베이스에 하나 이상의 컨테이너가 있는 MEMORY_OPTIMIZED_FILEGROUP이 있어야 합니다.
```

메모리 최적화 테이블을 사용하려면 다음 두 가지가 모두 필요하다.

1. `CONTAINS MEMORY_OPTIMIZED_DATA`로 생성한 파일 그룹
2. 해당 파일 그룹에 속하는 파일 또는 컨테이너 경로

일반 테이블만 필요하다면 생성문에서 다음 옵션을 제거하는 방법도 있다.

```sql
WITH (MEMORY_OPTIMIZED = ON)
```

`DURABILITY = SCHEMA_ONLY` 또는 `DURABILITY = SCHEMA_AND_DATA`도 메모리 최적화 테이블과 함께 사용하는 옵션이므로 목적에 맞는지 확인한다.

## 2. 로컬 디렉터리 준비

이 문서에서는 다음 경로를 사용한다.

```text
D:\MSSQL\DATA
D:\MSSQL\LOG
D:\MSSQL\MEMORY
```

`D:` 드라이브가 없다면 스크립트 상단의 경로만 다음처럼 바꾼다.

```text
C:\MSSQL\DATA
C:\MSSQL\LOG
C:\MSSQL\MEMORY
```

주의할 점은 다음과 같다.

- `DATA`, `LOG`, `MEMORY` 부모 폴더는 미리 만든다.
- `EXAMPLE_DEV_AUTH_mod` 같은 마지막 컨테이너 폴더는 미리 만들지 않고 SQL Server가 생성하게 둔다.
- SQL Server 서비스 계정에 해당 부모 폴더의 쓰기 권한이 있어야 한다.

## 3. 대상 DB

로컬 개발 환경에 다음 DB를 구성한다.

```text
EXAMPLE_DEV_AUTH
EXAMPLE_DEV_CONTENTS
EXAMPLE_DEV_GAME
EXAMPLE_DEV_LOG
```

공통 설정은 다음과 같다.

| 항목 | 값 |
| --- | --- |
| Collation | `Korean_Wansung_CI_AS` |
| Recovery model | `SIMPLE` |
| 메모리 최적화 파일 그룹 | `FG_MEMORY_OPTIMIZED` |
| 데이터 초기 크기 | `10MB` |
| 로그 초기 크기 | `10MB` |
| 자동 증가 | `256MB` |
| Compatibility level | 설치된 SQL Server 주 버전에 맞춰 자동 계산 |

## 4. 신규 생성과 기존 DB 보정을 함께 처리하는 스크립트

다음 스크립트는 DB가 없으면 생성하고, 이미 있으면 생성을 건너뛴다. 기존 DB에 메모리 최적화 파일 그룹이나 컨테이너가 없다면 추가한다.

```sql
USE master;
GO

DECLARE @DataRoot   nvarchar(260) = N'D:\MSSQL\DATA';
DECLARE @LogRoot    nvarchar(260) = N'D:\MSSQL\LOG';
DECLARE @MemoryRoot nvarchar(260) = N'D:\MSSQL\MEMORY';

DECLARE @MajorVersion int = TRY_CONVERT(int, SERVERPROPERTY('ProductMajorVersion'));
DECLARE @CompatibilityLevel int;

-- SQL Server 2014 이상에서는 주 버전 × 10이 일반적인 호환성 수준과 대응한다.
-- 예: 15 -> 150, 16 -> 160
IF @MajorVersion IS NULL OR @MajorVersion < 12
    THROW 50000, '메모리 최적화 테이블을 지원하는 SQL Server 버전인지 확인하세요.', 1;

SET @CompatibilityLevel = @MajorVersion * 10;

DECLARE @Databases table
(
    DbName sysname NOT NULL PRIMARY KEY
);

INSERT INTO @Databases (DbName)
VALUES
    (N'EXAMPLE_DEV_AUTH'),
    (N'EXAMPLE_DEV_CONTENTS'),
    (N'EXAMPLE_DEV_GAME'),
    (N'EXAMPLE_DEV_LOG');

DECLARE @DbName sysname;
DECLARE @Sql nvarchar(max);
DECLARE @MemoryFilegroup sysname;
DECLARE @MemoryFileCount int;

DECLARE db_cursor CURSOR LOCAL FAST_FORWARD FOR
SELECT DbName
FROM @Databases
ORDER BY DbName;

OPEN db_cursor;
FETCH NEXT FROM db_cursor INTO @DbName;

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT N'========================================';
    PRINT N'처리 중: ' + @DbName;

    IF DB_ID(@DbName) IS NULL
    BEGIN
        SET @Sql = N'
CREATE DATABASE ' + QUOTENAME(@DbName) + N'
ON PRIMARY
(
    NAME = N''' + @DbName + N''',
    FILENAME = N''' + @DataRoot + N'\' + @DbName + N'.mdf'',
    SIZE = 10MB,
    FILEGROWTH = 256MB
),
FILEGROUP [FG_MEMORY_OPTIMIZED] CONTAINS MEMORY_OPTIMIZED_DATA
(
    NAME = N''' + @DbName + N'_mod'',
    FILENAME = N''' + @MemoryRoot + N'\' + @DbName + N'_mod''
)
LOG ON
(
    NAME = N''' + @DbName + N'_log'',
    FILENAME = N''' + @LogRoot + N'\' + @DbName + N'_log.ldf'',
    SIZE = 10MB,
    FILEGROWTH = 256MB
)
COLLATE Korean_Wansung_CI_AS;';

        EXEC sys.sp_executesql @Sql;
        PRINT N'생성 완료: ' + @DbName;
    END
    ELSE
    BEGIN
        PRINT N'이미 존재하여 생성 생략: ' + @DbName;
    END;

    SET @Sql =
        N'ALTER DATABASE ' + QUOTENAME(@DbName) + N' SET RECOVERY SIMPLE;' +
        N'ALTER DATABASE ' + QUOTENAME(@DbName) + N' SET COMPATIBILITY_LEVEL = '
        + CONVERT(nvarchar(10), @CompatibilityLevel) + N';';

    EXEC sys.sp_executesql @Sql;

    SET @MemoryFilegroup = NULL;

    SET @Sql = N'
USE ' + QUOTENAME(@DbName) + N';

SELECT TOP (1)
    @MemoryFilegroupOut = name
FROM sys.filegroups
WHERE type = ''FX'';';

    EXEC sys.sp_executesql
        @Sql,
        N'@MemoryFilegroupOut sysname OUTPUT',
        @MemoryFilegroupOut = @MemoryFilegroup OUTPUT;

    IF @MemoryFilegroup IS NULL
    BEGIN
        SET @MemoryFilegroup = N'FG_MEMORY_OPTIMIZED';

        SET @Sql = N'
ALTER DATABASE ' + QUOTENAME(@DbName) + N'
ADD FILEGROUP ' + QUOTENAME(@MemoryFilegroup) + N'
CONTAINS MEMORY_OPTIMIZED_DATA;';

        EXEC sys.sp_executesql @Sql;
    END;

    SET @MemoryFileCount = 0;

    SET @Sql = N'
USE ' + QUOTENAME(@DbName) + N';

SELECT
    @MemoryFileCountOut = COUNT(*)
FROM sys.database_files
WHERE data_space_id = FILEGROUP_ID(@MemoryFilegroupName);';

    EXEC sys.sp_executesql
        @Sql,
        N'@MemoryFilegroupName sysname, @MemoryFileCountOut int OUTPUT',
        @MemoryFilegroupName = @MemoryFilegroup,
        @MemoryFileCountOut = @MemoryFileCount OUTPUT;

    IF @MemoryFileCount = 0
    BEGIN
        SET @Sql = N'
ALTER DATABASE ' + QUOTENAME(@DbName) + N'
ADD FILE
(
    NAME = N''' + @DbName + N'_mod'',
    FILENAME = N''' + @MemoryRoot + N'\' + @DbName + N'_mod''
)
TO FILEGROUP ' + QUOTENAME(@MemoryFilegroup) + N';';

        EXEC sys.sp_executesql @Sql;
    END;

    FETCH NEXT FROM db_cursor INTO @DbName;
END;

CLOSE db_cursor;
DEALLOCATE db_cursor;
GO
```

이 스크립트는 기존 DB의 Collation을 강제로 변경하지 않는다. 기존 DB의 Collation이 다르다면 데이터와 오브젝트에 미치는 영향을 확인한 뒤 별도로 변경한다.

## 5. `160` 설정에 실패했을 때

다음 구문에서 실패했더라도 `CREATE DATABASE`가 먼저 성공했다면 DB는 이미 존재한다.

```sql
ALTER DATABASE [EXAMPLE_DEV_AUTH] SET COMPATIBILITY_LEVEL = 160;
```

현재 SQL Server 버전을 확인한다.

```sql
SELECT
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('ProductMajorVersion') AS ProductMajorVersion,
    SERVERPROPERTY('Edition') AS Edition,
    @@VERSION AS FullVersion;
```

대표적인 대응 관계는 다음과 같다.

| ProductMajorVersion | SQL Server | Compatibility level |
| ---: | --- | ---: |
| `15` | SQL Server 2019 | `150` |
| `16` | SQL Server 2022 | `160` |

SQL Server 2019라면 삭제하지 않고 다음처럼 보정한다.

```sql
ALTER DATABASE [EXAMPLE_DEV_AUTH] SET COMPATIBILITY_LEVEL = 150;
```

## 6. 생성 결과 확인

DB의 상태, 호환성 수준, 복구 모델, Collation을 확인한다.

```sql
SELECT
    name,
    state_desc,
    compatibility_level,
    recovery_model_desc,
    collation_name
FROM sys.databases
WHERE name IN
(
    N'EXAMPLE_DEV_AUTH',
    N'EXAMPLE_DEV_CONTENTS',
    N'EXAMPLE_DEV_GAME',
    N'EXAMPLE_DEV_LOG'
)
ORDER BY name;
```

특정 DB의 파일 그룹과 실제 경로를 확인한다.

```sql
USE [EXAMPLE_DEV_AUTH];
GO

SELECT
    name,
    type_desc
FROM sys.filegroups;

SELECT
    name,
    type_desc,
    physical_name
FROM sys.database_files;
GO
```

정상이라면 메모리 최적화 파일 그룹의 `type_desc`에 다음 값이 보인다.

```text
MEMORY_OPTIMIZED_DATA_FILEGROUP
```

## 7. SQL Server 시스템 현재 시간 확인

DB 서버의 로컬 현재 시간은 다음 함수로 확인한다.

```sql
SELECT GETDATE() AS CurrentDateTime;
```

더 높은 정밀도가 필요하면 다음을 사용한다.

```sql
SELECT SYSDATETIME() AS CurrentDateTime;
```

로컬 시간과 UTC, 오프셋을 함께 비교할 수 있다.

```sql
SELECT
    GETDATE() AS LocalDateTime,
    SYSDATETIME() AS LocalDateTimeHighPrecision,
    GETUTCDATE() AS UtcDateTime,
    SYSUTCDATETIME() AS UtcDateTimeHighPrecision,
    SYSDATETIMEOFFSET() AS DateTimeWithOffset;
```

이 값은 애플리케이션 PC 시간이 아니라 SQL Server가 실행 중인 서버 운영체제의 시간을 기준으로 한다.

## 8. 자주 나는 문제

### 경로를 찾을 수 없거나 액세스가 거부된다

부모 디렉터리가 존재하는지, SQL Server 서비스 계정에 쓰기 권한이 있는지 확인한다.

```text
D:\MSSQL\DATA
D:\MSSQL\LOG
D:\MSSQL\MEMORY
```

### DB를 삭제한 뒤 같은 이름으로 다시 만들 수 없다

삭제 과정에서 다음 파일이나 컨테이너 폴더가 남았을 수 있다.

```text
D:\MSSQL\DATA\<database>.mdf
D:\MSSQL\LOG\<database>_log.ldf
D:\MSSQL\MEMORY\<database>_mod
```

DB가 실제로 삭제된 것을 확인한 뒤 남은 항목만 정리한다. 다른 DB가 사용하는 경로는 삭제하지 않는다.

### 메모리 최적화 파일 그룹은 보이지만 테이블 생성이 실패한다

파일 그룹만 있고 컨테이너가 없을 수 있다. `sys.database_files`에서 해당 파일 그룹에 속한 경로가 하나 이상 있는지 확인한다.

### 실제 비밀번호가 로그에 출력된다

접속 문자열 전체를 로그로 남기지 않는다. 서버, DB명, 사용자 ID 등 필요한 항목만 기록하고 비밀번호는 반드시 마스킹한다.

## 관련 문서

- [[SQL Server TLS 인증서 신뢰 오류 해결]]
- [[C# 트랜잭션 처리와 인메모리 롤백 예제]]
- [[지식_인덱스]]

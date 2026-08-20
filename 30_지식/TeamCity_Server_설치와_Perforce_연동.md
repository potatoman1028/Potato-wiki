---
type: 지식
created: 2026-08-20
updated: 2026-08-20
tags: [TeamCity, CI_CD, Perforce, MSSQL, WindowsServer]
aliases: [TeamCity 설치, 팀시티 설치, TeamCity P4 연동]
source: 대화 기록, TeamCity 설치 PDF 메모
---

# TeamCity Server 설치와 Perforce 연동

## 요약

- Windows Server 2022에 TeamCity On-Premises를 설치할 때는 **Server**, **Data Directory**, **외부 DB**, **Build Agent**를 분리해서 생각한다.
- 설치 마법사에서 나오는 Agent 설정 화면의 `systemDir`, `workDir`, `tempDir`는 TeamCity Server의 Data Directory가 아니라 **Build Agent 작업 폴더**다.
- 운영용 TeamCity는 Internal DB보다 **MSSQL 외부 DB**를 사용하는 쪽으로 잡는다.
- Perforce 연동은 Agent만으로 끝나지 않는다. TeamCity Server도 변경 감지와 VCS Root 테스트를 위해 `p4.exe`를 찾을 수 있어야 한다.
- 공개 저장소에는 비밀번호, 라이선스 키, 실제 P4 주소, 실제 내부 계정 암호를 기록하지 않는다.

## 목표 구성

```text
TeamCity Server
- Windows Server 2022
- TeamCity On-Premises
- Server URL: http://<TEAMCITY_SERVER>:8111
- Data Directory: D:\TeamCityData

Database
- MSSQL 외부 DB
- DB name: TeamcityServer
- Collation: Latin1_General_CS_AS
- Login: teamcity
- Role: db_owner

Build Agent
- TeamCity Server와 분리 권장
- Agent work/system/temp는 D드라이브 등 여유 있는 디스크 사용
- Perforce Command-Line Client 설치
```

## 설치 순서

1. TeamCity On-Premises Windows installer를 다운로드한다.
2. 설치 마법사에서 TeamCity Server를 설치한다.
3. 서버와 Agent를 같이 선택한 경우 Agent 설정 창이 먼저 뜰 수 있다.
4. Agent 설정 창에서는 `systemDir`, `workDir`, `tempDir`를 빌드 작업용 디스크로 지정한다.
5. TeamCity Server 서비스 계정을 지정한다.
6. 웹 초기 설정에서 TeamCity Data Directory를 `D:\TeamCityData`로 지정한다.
7. MSSQL 외부 DB를 생성하고 TeamCity DB 연결 화면에서 지정한다.
8. 라이선스를 등록한다.
9. Agent를 별도 장비에 설치하고 서버 URL을 실제 접속 가능한 주소로 맞춘다.
10. Perforce VCS Root를 생성한다.

## 다운로드와 설치 파일

TeamCity 다운로드 화면에서는 Cloud가 추천으로 보일 수 있지만, 자체 서버에 설치하는 경우에는 **On-premises**를 선택한다.

회사 라이선스가 이미 있더라도 다운로드 단계의 입력 폼은 설치파일 다운로드 또는 트라이얼 신청 흐름이다. 회사 설치라면 개인 계정보다는 회사 이메일과 회사명을 사용하는 편이 낫다.

```text
선택 대상
- On-premises
- Windows (.exe)
```

## 설치 마법사에서 헷갈린 부분

PDF 메모의 설치 화면에서는 Server와 Build Agent가 같이 선택되어 있었다. 이 경우 설치 도중 Agent 설정 창이 먼저 나온다.

이 화면은 TeamCity Server의 Data Directory를 묻는 것이 아니다.

```text
Agent 설정 항목
- serverUrl: Agent가 접속할 TeamCity Server URL
- name: TeamCity 화면에 표시될 Agent 이름
- systemDir: Agent 시스템/캐시성 데이터
- workDir: 실제 checkout/build 작업 폴더
- tempDir: Agent 임시 폴더
```

권장 예시는 다음과 같다.

```text
TeamCity Server Data Directory
D:\TeamCityData

Build Agent
systemDir = D:\TeamCityBuildAgent\system
workDir   = D:\TeamCityBuildAgent\work
tempDir   = D:\TeamCityBuildAgent\temp
```

C드라이브 용량이 충분하면 기본값으로 진행해도 되지만, 빌드 산출물과 checkout 데이터가 커질 수 있으므로 Agent 작업 폴더는 별도 디스크로 빼는 편이 안전하다.

## 서비스 계정

서비스 계정 화면은 TeamCity Server 서비스를 어떤 Windows 계정으로 실행할지 묻는 화면이다.

중요한 점은 비밀번호 입력 칸이 **새 비밀번호를 만드는 곳이 아니라 기존 Windows 계정의 실제 로그인 암호를 입력하는 곳**이라는 점이다.

```text
예시
Domain   = <MACHINE_NAME>
Username = Administrator 또는 svc_teamcity
Password = 해당 Windows 계정의 실제 암호
```

다음 오류가 나오면 계정명, 암호, 서비스 로그온 권한을 확인한다.

```text
Failed to install the service.
Selected account does not have enough rights to run as service.
Login failed.
```

확인 순서:

```powershell
whoami
```

```text
secpol.msc
→ Local Policies
→ User Rights Assignment
→ Log on as a service
→ 서비스 실행 계정 추가
```

한국어 Windows에서는 다음 경로로 보면 된다.

```text
secpol.msc
→ 로컬 정책
→ 사용자 권한 할당
→ 서비스로 로그온
```

## TeamCity Data Directory

TeamCity 웹 초기 실행 화면에서 Data Directory를 묻는다. 여기서 서버 데이터 위치를 지정한다.

```text
D:\TeamCityData
```

이 위치는 Agent의 `workDir`와 다르다.

```text
D:\TeamCityData
- TeamCity 서버 설정
- 프로젝트/빌드 구성
- 빌드 기록 관련 데이터
- 캐시/운영 데이터

D:\TeamCityBuildAgent\work
- Agent checkout 작업 폴더
- 실제 build 작업 폴더
```

## MSSQL 외부 DB 설정

운영용으로는 Internal DB가 아니라 MSSQL 외부 DB를 사용한다.

전제:

```text
SQL Login: teamcity
DB name: TeamcityServer
Collation: Latin1_General_CS_AS
Data path: D:\DATA
Log path: D:\LOG
```

회사 기존 DB 스타일을 참고해 `Latin1_General_CS_AS` 형식으로 맞췄다. 핵심은 대소문자를 구분하는 `CS`가 포함되는 것이다.

### DB 생성 스크립트

```sql
USE [master];
GO

CREATE DATABASE [TeamcityServer]
ON PRIMARY
(
    NAME = N'TeamcityServer',
    FILENAME = N'D:\DATA\TeamcityServer.mdf',
    SIZE = 4096MB,
    FILEGROWTH = 512MB
)
LOG ON
(
    NAME = N'TeamcityServer_log',
    FILENAME = N'D:\LOG\TeamcityServer_log.ldf',
    SIZE = 1024MB,
    FILEGROWTH = 512MB
)
COLLATE Latin1_General_CS_AS;
GO

ALTER LOGIN [teamcity]
WITH DEFAULT_DATABASE = [TeamcityServer];
GO

USE [TeamcityServer];
GO

IF NOT EXISTS (
    SELECT 1
    FROM sys.database_principals
    WHERE name = N'teamcity'
)
BEGIN
    CREATE USER [teamcity] FOR LOGIN [teamcity];
END
GO

IF ISNULL(IS_ROLEMEMBER(N'db_owner', N'teamcity'), 0) <> 1
BEGIN
    ALTER ROLE [db_owner] ADD MEMBER [teamcity];
END
GO
```

### TeamCity 웹 DB 설정값

```text
Database type: MS SQL Server
Host: <DB_SERVER>
Port: 1433
Database name: TeamcityServer
Authentication: SQL Server Authentication
User: teamcity
Password: <teamcity SQL 계정 암호>
```

### JDBC 드라이버

TeamCity 웹 DB 설정 화면에서 MSSQL JDBC 드라이버가 필요하다고 나오면 다음 위치에 드라이버를 둔다.

```text
D:\TeamCityData\lib\jdbc
```

## Agent 설정

Agent 다운로드 페이지의 링크가 `http://localhost:8111/...`로 보이면 외부 Agent에서는 사용할 수 없다. 외부 Agent의 `localhost`는 TeamCity Server가 아니라 Agent 자기 자신이다.

TeamCity 웹 UI에서 서버 URL을 실제 접근 가능한 주소로 바꾼다.

```text
Administration
→ Global Settings
→ Server URL
→ http://<TEAMCITY_SERVER>:8111
```

Agent 설치 시 `serverUrl`도 동일한 주소를 사용한다.

```text
serverUrl = http://<TEAMCITY_SERVER>:8111
```

### Agent 이름 변경

Agent 이름은 각 Agent 장비의 `buildAgent.properties`에서 바꾼다.

```text
C:\TeamCity\buildAgent\conf\buildAgent.properties
```

```properties
name=CSI-BUILD-A1
```

수정 후 Agent 서비스를 재시작한다.

```powershell
Restart-Service "TeamCity Build Agent"
```

### 서버에 같이 설치된 로컬 Agent 제거

TeamCity Server 장비에서 로컬 Agent를 제거할 때는 Server 서비스가 아니라 Build Agent 서비스만 제거한다.

```powershell
Get-Service *TeamCity*

Stop-Service "TeamCity Build Agent"

cd C:\TeamCity\buildAgent\bin
.\service.stop.bat
.\service.uninstall.bat

Remove-Item -Recurse -Force C:\TeamCity\buildAgent
```

## 라이선스 등록

라이선스는 설치파일 다운로드 단계가 아니라 TeamCity 웹 UI에서 등록한다.

```text
Administration
→ Licenses
→ JetBrains Account 또는 Activation/License Key 등록
```

회사 라이선스가 있는 경우 담당자에게 JetBrains Account 접근 권한 또는 activation key를 받아 등록한다.

## Perforce 연결

TeamCity와 Perforce를 연결할 때는 VCS Root를 만든다.

중요한 점:

- Agent-side checkout을 사용하더라도 TeamCity Server가 VCS Root 테스트와 변경 감지를 수행한다.
- 따라서 Build Agent뿐 아니라 TeamCity Server에도 `p4.exe`가 필요하다.
- P4V 전체가 필수는 아니고, TeamCity 연동에는 Command-Line Client인 `p4.exe`가 핵심이다.

### 발생한 오류

```text
Unable to run Perforce Helix command-line client using path 'p4'.
Host '<TEAMCITY_SERVER>'. OS user: 'Administrator'.
Cannot run program "p4": CreateProcess error=2, 지정된 파일을 찾을 수 없습니다
```

원인:

```text
TeamCity Server 서비스가 p4.exe를 PATH에서 찾지 못함
```

해결:

```powershell
where.exe p4
p4 -V
Restart-Service "TeamCity Server"
```

PATH 인식이 안 되면 TeamCity Internal Properties에 직접 경로를 지정할 수 있다.

```properties
teamcity.perforce.customP4Path=C:/Program Files/Perforce/p4.exe
```

### P4 접속 확인

```powershell
p4 -V
p4 -p ssl:<P4_SERVER>:1666 -u teamcity info
p4 -p ssl:<P4_SERVER>:1666 trust
p4 -p ssl:<P4_SERVER>:1666 -u teamcity login
```

실제 비밀번호나 ticket은 문서에 기록하지 않는다.

## VCS Root 생성

TeamCity 웹 UI에서 다음 순서로 생성한다.

```text
Project
→ Project Settings
→ VCS Roots
→ Create VCS Root
→ Perforce P4 (Helix Core)
```

Stream 기반 예시:

```text
Type of VCS: Perforce P4 (Helix Core)
VCS root name: P4_Server_Dev
VCS root ID: P4_Server_Dev
Port: ssl:<P4_SERVER>:1666
Mode: Stream
Stream: //<STREAM_DEPOT>/dev
Username: teamcity
Password or ticket: <P4 계정 암호 또는 ticket>
```

Stream을 사용할 경우 `Client`와 `Client mapping`은 건드리지 않는다.

일반 depot mapping을 사용하는 경우에는 `Client mapping`을 선택하고 오른쪽 client 이름 대신 `team-city-agent`를 사용한다.

```text
//depot/... //team-city-agent/...
```

## 프로젝트 구조

원하는 구조는 TeamCity에서 Project, Subproject, Build Configuration으로 나누면 된다.

```text
<Root project>
├─ Server                  # Project
│  └─ dev stream            # Subproject
│     ├─ Build Debug        # Build Configuration
│     └─ Build Release      # Build Configuration
└─ Client
   └─ dev stream
      ├─ Build Debug
      └─ Build Release
```

Server부터 만들 때의 순서:

```text
1. Projects → Create project → Manually
2. Project name: Server
3. Project ID: Server
4. Server 프로젝트 안에서 Create subproject
5. Project name: dev stream
6. Project ID: Server_Dev
7. Server_Dev에 P4_Server_Dev VCS Root 연결
8. Build Debug, Build Release Build Configuration 생성
```

## 삽질 포인트 정리

### 다운로드 버튼이 헷갈림

Cloud가 추천으로 보였지만 Windows Server 2022에 설치하는 목적이므로 On-premises의 Windows `.exe`를 받아야 했다.

### Agent 설정 화면을 서버 Data Directory로 착각

설치 중 나온 Agent 설정 화면은 Agent 작업 폴더 설정이었다. Server Data Directory는 웹 첫 실행 화면에서 `D:\TeamCityData`로 지정한다.

### 서비스 계정 오류

처음에는 권한 문제처럼 보였지만 실제로는 Windows 계정 암호를 헷갈린 것이 원인이었다. 그래도 같은 오류가 반복되면 `secpol.msc`에서 `서비스로 로그온` 권한을 확인한다.

### DB Collation 결정

다른 프로젝트 DB 스타일을 참고해 `Latin1_General_CS_AS`로 생성했다. TeamCity용 DB는 대소문자 구분 Collation을 사용한다.

### Agent 다운로드 링크가 localhost

Global Settings의 Server URL이 `localhost`로 되어 있으면 외부 Agent 설치 링크도 localhost로 생성된다. 실제 서버 주소로 바꿔야 한다.

### TeamCity Server에서 p4를 못 찾음

Agent 장비에만 P4를 설치했지만, TeamCity Server에도 `p4.exe`가 필요했다. Server에 Command-Line Client를 설치하고 TeamCity Server 서비스를 재시작했다.

## 명령어 모음

### Windows 서비스

```powershell
Get-Service *TeamCity*
Restart-Service "TeamCity Server"
Restart-Service "TeamCity Build Agent"
```

### 폴더 생성

```powershell
New-Item -ItemType Directory -Force D:\TeamCityData
New-Item -ItemType Directory -Force D:\TeamCityData\lib\jdbc
```

### P4 확인

```powershell
where.exe p4
p4 -V
p4 -p ssl:<P4_SERVER>:1666 -u teamcity info
p4 -p ssl:<P4_SERVER>:1666 trust
p4 -p ssl:<P4_SERVER>:1666 -u teamcity login
```

### DB 확인

```sql
SELECT
    name,
    collation_name,
    state_desc
FROM sys.databases
WHERE name = N'TeamcityServer';
```

```sql
USE [TeamcityServer];

SELECT
    DP1.name AS DatabaseRoleName,
    DP2.name AS DatabaseUserName
FROM sys.database_role_members DRM
JOIN sys.database_principals DP1
    ON DRM.role_principal_id = DP1.principal_id
JOIN sys.database_principals DP2
    ON DRM.member_principal_id = DP2.principal_id
WHERE DP2.name = N'teamcity';
```

## 다음에 보강할 내용

- Build Debug / Build Release의 실제 `dotnet build` 또는 MSBuild Step
- Artifact 경로와 Clean-up 정책
- TeamCity Data Directory와 MSSQL 백업 정책
- HTTPS 또는 Reverse Proxy 적용 과정
- Perforce Stream별 권한 정책

## 관련 문서

- [[지식_인덱스]]

---
type: 지식
created: 2026-08-20
updated: 2026-08-20
tags: [MSSQL, SQLServer, TLS, SSL, 인증서, Windows]
aliases: [SQL Server SSL 공급자 오류, 신뢰되지 않은 인증서 체인 해결]
source: ChatGPT 대화 정리
---

# SQL Server TLS 인증서 신뢰 오류 해결

## 요약

- `신뢰되지 않은 기관에서 인증서 체인을 발급했습니다` 오류는 DB 계정이나 DB 데이터 자체의 문제가 아니라, 클라이언트가 SQL Server가 제시한 TLS 인증서를 신뢰하지 못해서 발생한다.
- `TrustServerCertificate=True`는 인증서 검증을 건너뛰는 빠른 우회 방법이지만, 접속 문자열을 수정하지 않고 해결하려면 서버 인증서와 클라이언트 신뢰 체인을 정상적으로 구성해야 한다.
- 로컬 개발 환경에서는 `localhost`와 컴퓨터 이름이 SAN에 포함된 인증서를 SQL Server에 지정하고, 앱이 실행되는 Windows의 신뢰할 수 있는 루트 인증 기관에 등록할 수 있다.
- SQL Server 서비스 계정에는 인증서 개인키 읽기 권한이 필요하다.
- `Force Encryption`은 인증서 지정과 별개다. 모든 클라이언트에 암호화를 강제해야 할 때만 켠다.
- 전체 접속 문자열을 로그에 남기면 비밀번호가 노출될 수 있으므로 반드시 마스킹한다.

## 내용

## 1. 오류의 의미

대표적인 오류는 다음과 같다.

```text
서버에 연결했지만 로그인하는 동안 오류가 발생했습니다.
(provider: SSL 공급자, error: 0 - 신뢰되지 않은 기관에서 인증서 체인을 발급했습니다.)
```

연결 흐름은 다음과 같다.

```text
애플리케이션
→ SQL Server에 암호화 연결 요청
→ SQL Server가 TLS 인증서 제시
→ 애플리케이션이 발급자, 유효기간, 서버 이름을 검증
→ 신뢰할 수 없으면 로그인 단계에서 연결 중단
```

따라서 정확한 의미는 다음과 같다.

> 외부 DB 자체를 신뢰하지 못하는 것이 아니라, 클라이언트가 DB 서버의 TLS 인증서를 신뢰하지 못하는 상태다.

이 문제는 원격 DB뿐 아니라 자체 서명 인증서를 사용하는 `localhost` SQL Server에서도 동일하게 발생할 수 있다.

## 2. 해결 방식 비교

| 방식 | 장점 | 단점 | 권장 범위 |
| --- | --- | --- | --- |
| `TrustServerCertificate=True` | 적용이 빠름 | 서버 신원 검증을 생략함 | 임시 로컬 테스트 |
| `Encrypt=False` | 일부 환경에서 빠르게 연결 가능 | 암호화가 비활성화될 수 있고 서버 정책에 따라 실패 | 비추천 |
| 신뢰 가능한 인증서 구성 | 접속 문자열을 유지하면서 암호화와 서버 검증 가능 | 최초 설정 필요 | 공용 개발·운영, 제대로 구성할 로컬 환경 |

접속 문자열을 수정하지 않는 목표라면 세 번째 방식을 사용한다.

## 3. 전제 조건 확인

### 3.1 앱이 사용하는 서버 이름

인증서의 Subject Alternative Name, 즉 SAN에는 앱이 실제로 사용하는 서버 이름이 포함되어야 한다.

예를 들어 다음 접속 문자열이라면 인증서 SAN에 `localhost`가 필요하다.

```text
Data Source=localhost;Initial Catalog=<database>;User ID=<user>;Password=<password>;
```

대표적인 대응은 다음과 같다.

| 접속 대상 | 인증서 SAN에 필요한 이름 |
| --- | --- |
| `localhost` | `localhost` |
| `localhost\SQLEXPRESS` | `localhost` |
| `MY-PC` | `MY-PC` |
| `MY-PC\SQLEXPRESS` | `MY-PC` |
| `127.0.0.1` | DNS SAN이 아니라 IP SAN이 필요할 수 있음 |

이 문서의 절차는 `Data Source=localhost`인 로컬 Windows 환경을 기준으로 한다.

### 3.2 SQL Server 서비스 계정

SQL Server Configuration Manager에서 확인한다.

```text
SQL Server 서비스
→ SQL Server (MSSQLSERVER)
→ 속성
→ 로그온 탭
```

기본 인스턴스의 가상 서비스 계정은 보통 다음과 같다.

```text
NT SERVICE\MSSQLSERVER
```

SQLEXPRESS라면 보통 다음 형식이다.

```text
NT SERVICE\MSSQL$SQLEXPRESS
```

서비스 계정 속성에서는 값을 변경하지 않고 현재 계정 이름만 확인한다.

## 4. 로컬 개발용 인증서 생성

> 로컬 한 대에서만 사용하는 실습용 구성이다. 공용 개발·운영 환경에서는 사내 CA 또는 조직에서 승인한 CA가 발급한 서버 인증서를 사용한다.

관리자 권한 PowerShell에서 실행한다.

```powershell
$dnsNames = @(
    $env:COMPUTERNAME,
    "localhost"
)

$cert = New-SelfSignedCertificate `
    -Type SSLServerAuthentication `
    -DnsName $dnsNames `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyAlgorithm RSA `
    -KeyLength 2048 `
    -HashAlgorithm SHA256 `
    -KeySpec KeyExchange `
    -Provider "Microsoft RSA SChannel Cryptographic Provider" `
    -KeyUsage DigitalSignature, KeyEncipherment `
    -NotAfter (Get-Date).AddYears(3)

$cert | Select-Object Subject, Thumbprint, NotAfter, HasPrivateKey, DnsNameList
```

정상 인증서는 다음 조건을 만족해야 한다.

- `HasPrivateKey`가 `True`
- `DnsNameList`에 `localhost` 포함
- 아직 만료되지 않음
- 서버 인증 용도의 인증서

컴퓨터 이름을 첫 번째 DNS 이름으로 넣으면 Subject가 `CN=<컴퓨터 이름>`으로 표시될 수 있다. 중요한 것은 표시 이름보다 SAN과 Thumbprint다.

## 5. 클라이언트 신뢰 저장소에 등록

같은 관리자 PowerShell에서 인증서를 내보내고 로컬 컴퓨터의 신뢰할 수 있는 루트 인증 기관에 등록한다.

```powershell
$certPath = "C:\Temp\sqlserver-local-dev.cer"

New-Item -ItemType Directory -Force -Path "C:\Temp" | Out-Null

Export-Certificate `
    -Cert $cert `
    -FilePath $certPath

Import-Certificate `
    -FilePath $certPath `
    -CertStoreLocation "Cert:\LocalMachine\Root"
```

등록 여부를 Thumbprint로 확인한다.

```powershell
$thumbprint = $cert.Thumbprint

Get-ChildItem Cert:\LocalMachine\Root |
    Where-Object Thumbprint -eq $thumbprint |
    Select-Object Subject, Thumbprint, NotAfter
```

앱과 SQL Server가 같은 PC에서 실행된다면 이 PC에 등록하면 된다. 앱이 다른 Windows 서버에서 실행되면 그 앱 실행 서버에도 신뢰 인증서를 등록해야 한다.

## 6. SQL Server 서비스 계정에 개인키 읽기 권한 부여

Windows 실행창에서 다음을 실행한다.

```text
certlm.msc
```

다음 위치로 이동한다.

```text
인증서 - 로컬 컴퓨터
→ 개인
→ 인증서
→ 생성한 인증서 우클릭
→ 모든 작업
→ 개인 키 관리
```

SQL Server 서비스 계정을 추가하고 `읽기` 권한을 부여한다.

```text
NT SERVICE\MSSQLSERVER
```

계정 선택 창의 위치가 회사 도메인으로 잡혀 있다면 `위치`를 SQL Server가 설치된 로컬 컴퓨터로 바꾼다. 이름을 확인할 때는 다음과 같이 전체 이름을 입력한다.

```text
NT SERVICE\MSSQLSERVER
```

이름 확인이 성공한 뒤 권한을 저장한다.

## 7. SQL Server에 인증서 지정

이 설정은 SSMS가 아니라 SQL Server Configuration Manager에서 한다.

### 7.1 Configuration Manager 실행

Windows 실행창에서 SQL Server 버전에 맞는 명령을 실행한다.

| SQL Server | 실행 명령 |
| --- | --- |
| SQL Server 2022 | `SQLServerManager16.msc` |
| SQL Server 2019 | `SQLServerManager15.msc` |
| SQL Server 2017 | `SQLServerManager14.msc` |

파일은 일반적으로 다음 위치에 있다.

```text
C:\Windows\SysWOW64\SQLServerManager15.msc
```

### 7.2 인증서 선택

왼쪽 메뉴에서 32비트 항목이 아닌 일반 네트워크 구성을 선택한다.

```text
SQL Server 네트워크 구성
→ MSSQLSERVER에 대한 프로토콜
→ 우클릭
→ 속성
→ 인증서 탭
```

목록에서 생성한 인증서를 선택한다. 이름이 여러 개라면 PowerShell에서 확인한 Thumbprint와 SAN을 기준으로 구분한다.

### 7.3 Force Encryption 결정

같은 속성 창의 `플래그` 탭에 `Force Encryption`이 있다.

- `No`: 클라이언트가 암호화를 요청하는 연결만 암호화한다.
- `Yes`: 이 인스턴스에 접속하는 모든 네트워크 클라이언트에 암호화를 요구한다.

현재 오류를 해결하는 핵심은 `인증서 지정 + 클라이언트 신뢰 등록`이다. `Force Encryption = Yes`는 필수 조건이 아니며 다른 클라이언트에 영향을 줄 수 있으므로 정책상 필요한 경우에만 설정한다.

### 7.4 서비스 재시작

인증서를 지정한 뒤 반드시 SQL Server 서비스를 재시작한다.

```text
SQL Server 서비스
→ SQL Server (MSSQLSERVER)
→ 우클릭
→ 다시 시작
```

## 8. 적용 확인

### 8.1 SQL Server 에러 로그 확인

SSMS에서 다음을 실행한다.

```sql
EXEC xp_readerrorlog 0, 1, N'certificate';
EXEC xp_readerrorlog 0, 1, N'encryption';
```

다음과 같은 메시지가 보이면 인증서를 로드하지 못한 것이다.

```text
Unable to load user-specified certificate
The certificate was not found
The server could not load the certificate
```

이 경우 다음 항목을 다시 확인한다.

- 인증서가 `LocalMachine\My`에 있는가
- 개인키가 있는가
- SQL Server 서비스 계정에 개인키 읽기 권한이 있는가
- Configuration Manager에서 올바른 인스턴스에 인증서를 지정했는가

### 8.2 연결 암호화 여부 확인

접속 후 다음 쿼리를 실행한다.

```sql
SELECT
    session_id,
    encrypt_option,
    auth_scheme,
    client_net_address
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

`encrypt_option = TRUE`이면 현재 세션이 암호화되어 있다.

인증서 신뢰까지 검증하려면 SSMS 접속 옵션의 `Trust server certificate`를 체크하지 않은 상태에서 정상 접속되는지도 확인한다.

## 9. 그래도 실패할 때

### 인증서는 있는데 같은 체인 오류가 난다

다음을 확인한다.

1. SQL Server 재시작 여부
2. 인증서가 `LocalMachine\Root`에도 등록되었는지
3. 앱이 실제로 실행되는 머신의 신뢰 저장소에 등록했는지
4. SQL Server가 새 인증서를 로드했는지
5. 다른 자체 서명 인증서가 선택되어 있지는 않은지

### 이름 불일치 오류로 바뀐다

체인 신뢰 문제를 해결한 뒤 서버 이름 불일치 오류가 새로 나타날 수 있다. `Data Source=localhost`라면 SAN에 `localhost`가 있어야 한다.

### 앱이 Docker, WSL 또는 Linux에서 실행된다

Windows의 `LocalMachine\Root`는 해당 런타임의 신뢰 저장소가 아니다. 컨테이너나 Linux 배포판 내부의 CA 저장소에도 인증서를 등록해야 한다.

### 인증서가 Configuration Manager 목록에 보이지 않는다

다음을 확인한다.

- 로컬 컴퓨터의 개인 인증서 저장소에 있는지
- 개인키가 있는지
- Server Authentication 용도인지
- 유효기간이 남아 있는지
- SQL Server 서비스 계정이 개인키를 읽을 수 있는지

## 10. 보안상 주의점

접속 오류 로그에 다음 값을 함께 출력하면 실제 비밀번호가 그대로 노출될 수 있다.

```text
connectionString:Data Source=...;User ID=...;Password=...;
```

전체 접속 문자열을 기록하지 말고 필요한 속성만 분리해서 남긴다.

```text
server=localhost
database=CSI_DEV_AUTH
user=sa
password=***
```

이미 비밀번호가 로그, 메신저, 이슈 또는 공개 저장소에 노출되었다면 해당 비밀번호를 변경한다.

## 관련 문서

- [[MSSQL 로컬 개발 DB 구성과 메모리 최적화 설정]]
- [[지식_인덱스]]

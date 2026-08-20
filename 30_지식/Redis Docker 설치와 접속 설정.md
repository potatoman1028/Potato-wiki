---
type: 지식
created: 2026-08-20
updated: 2026-08-20
tags: [Redis, Docker, Windows, 개발환경]
aliases: [레디스 도커 설치, RedisInsight 연결]
source: ChatGPT 대화 정리
---

# Redis Docker 설치와 접속 설정

## 요약

- Windows 개발 PC에서는 Redis를 직접 설치하기보다 Docker로 실행하는 방식이 관리하기 쉽다.
- Docker Desktop에서 Redis 이미지만 받은 상태와 컨테이너가 실행 중인 상태는 다르다.
- 외부 도구에서 접속하려면 컨테이너 실행 시 `-p 6379:6379` 포트 매핑이 필요하다.
- `docker ps`의 `PORTS`가 `6379/tcp`만 보이면 Windows의 `127.0.0.1:6379`로는 접속할 수 없다.
- 공개 저장소에는 실제 비밀번호를 기록하지 않는다. 예시는 `<password>` 같은 자리 표시자를 사용한다.

## 내용

## 1. Docker 설치 여부 확인

PowerShell에서 다음 명령어로 Docker 설치 여부를 확인한다.

```powershell
docker --version
```

Docker Desktop이 실행 중인지 확인한다.

```powershell
docker ps
```

정상이라면 컨테이너 목록이 출력된다. Docker Desktop이 꺼져 있으면 Docker daemon에 연결할 수 없다는 오류가 날 수 있다.

## 2. Redis 컨테이너 실행

기존 컨테이너가 꼬였거나 포트 매핑 없이 생성되었다면 삭제 후 다시 만든다.

```powershell
docker rm -f redis-dev
```

Redis를 로컬 개발용으로 실행한다.

```powershell
docker run -d --name redis-dev -p 6379:6379 redis:latest
```

`-p 6379:6379`는 Windows의 6379 포트를 컨테이너 내부 6379 포트로 연결한다.

## 3. 포트 매핑 확인

```powershell
docker ps
```

정상 포트 매핑은 `PORTS`에 다음과 비슷하게 표시된다.

```text
0.0.0.0:6379->6379/tcp
```

문제가 있는 상태는 다음처럼 보인다.

```text
6379/tcp
```

이 경우 컨테이너 내부에서만 Redis가 열려 있고, Windows의 `127.0.0.1:6379`에서는 접근할 수 없다. 컨테이너를 삭제하고 `-p 6379:6379` 옵션으로 다시 만들어야 한다.

## 4. Redis 실행 확인

Docker Desktop에서 `redis-dev` 컨테이너를 선택하고 `Exec` 탭에서 다음을 실행한다.

```bash
redis-cli ping
```

정상이라면 다음이 출력된다.

```text
PONG
```

간단한 저장/조회 테스트는 다음처럼 할 수 있다.

```bash
redis-cli
```

```redis
set test hello
get test
del test
exit
```

## 5. Redis 계정 생성

Redis CLI에 들어간다.

```bash
redis-cli
```

예시 계정 `gameuser`를 만들고 비밀번호를 설정한다.

```redis
ACL SETUSER gameuser on ><password> ~* &* +@all
```

의미는 다음과 같다.

| 항목 | 의미 |
| --- | --- |
| `gameuser` | 생성할 Redis 사용자 이름 |
| `on` | 사용자 활성화 |
| `><password>` | 비밀번호 설정 |
| `~*` | 모든 key 접근 허용 |
| `&*` | 모든 Pub/Sub channel 접근 허용 |
| `+@all` | 모든 명령어 허용 |

생성 확인은 다음 명령어로 한다.

```redis
ACL LIST
```

CLI에서 나간다.

```redis
exit
```

계정 접속 테스트는 다음처럼 한다.

```bash
redis-cli -u redis://gameuser:<password>@127.0.0.1:6379 ping
```

정상이라면 `PONG`이 출력된다.

## 6. RedisInsight 연결 설정

RedisInsight에서 직접 입력할 경우 다음과 같이 설정한다.

| 항목 | 값 |
| --- | --- |
| Database alias | `Local Redis` 또는 원하는 이름 |
| Host | `127.0.0.1` |
| Port | `6379` |
| Username | `gameuser` |
| Password | `<password>` |
| Timeout | `30` |
| Select Logical Database | 기본값 유지 |
| Force Standalone Connection | 기본값 유지 |

Connection URL 형식이 필요한 경우 다음을 사용한다.

```text
redis://gameuser:<password>@127.0.0.1:6379/0
```

RedisInsight가 Docker 확장 또는 컨테이너 형태로 실행되어 `127.0.0.1` 접속에 실패하면 Host를 다음 값으로 바꿔 테스트한다.

```text
host.docker.internal
```

## 7. StackExchange.Redis 접속 문자열

C# `StackExchange.Redis` 형식의 접속 문자열은 다음과 같이 쓴다.

```text
127.0.0.1:6379,user=gameuser,allowAdmin=true,password=<password>
```

예시 코드:

```csharp
var redis = await ConnectionMultiplexer.ConnectAsync(
    "127.0.0.1:6379,user=gameuser,allowAdmin=true,password=<password>"
);

var db = redis.GetDatabase();
await db.StringSetAsync("test", "hello");
var value = await db.StringGetAsync("test");
```

URL 형식이 필요한 도구에서는 다음을 사용한다.

```text
redis://gameuser:<password>@127.0.0.1:6379/0
```

## 8. 자주 나는 문제

### RedisInsight에서 `Could not connect to 127.0.0.1:6379`가 뜬다

대부분 컨테이너 포트 매핑 문제다.

```powershell
docker ps
```

`PORTS`가 `6379/tcp`만 보이면 잘못 띄운 상태다. 아래처럼 다시 만든다.

```powershell
docker rm -f redis-dev
docker run -d --name redis-dev -p 6379:6379 redis:latest
```

### 비밀번호 오류가 난다

접속 자체가 되는 상태에서 비밀번호만 틀리면 보통 인증 오류가 난다. Redis CLI에서 먼저 확인한다.

```bash
redis-cli -u redis://gameuser:<password>@127.0.0.1:6379 ping
```

### Exec에서 `ACL SETUSER`를 한 줄로 바로 실행하면 이상하게 동작한다

쉘에서 `>`와 `&`가 리다이렉션 또는 백그라운드 실행으로 해석될 수 있다. 안전하게 `redis-cli`에 먼저 들어간 뒤 Redis 프롬프트에서 실행한다.

## 관련 문서

- [[지식_인덱스]]

---
type: 학습
created: 2026-08-20
updated: 2026-08-20
status: 완료
step: 5
completed: 2026-08-20
tags: [게임서버, Kubernetes, Pod, Deployment, ReplicaSet]
---

# 5단계 — Pod·Deployment·ReplicaSet

상위 문서: [[00_프로젝트_홈|게임 서버 배포 학습]]  
이전 단계: [[04_Kubernetes의_선언적_운영]]

## 이번 단계의 목표

4단계에서는 Kubernetes가 원하는 상태와 현재 상태를 계속 맞춘다고 배웠다. 이번 단계에서는 그 상태가 실제로 어떤 리소스 관계로 표현되는지 배운다.

가장 먼저 다음 구조를 기억한다.

```text
Deployment
  └─ ReplicaSet
       └─ Pod
            └─ Container
                 └─ 서버 Process
```

이 문서를 읽은 뒤에는 다음 질문에 답할 수 있어야 한다.

- Pod와 Container는 무엇이 다른가?
- Deployment와 ReplicaSet은 왜 따로 존재하는가?
- GameWorker 세 개는 어떤 리소스와 프로세스로 표현되는가?
- Pod가 사라졌을 때 누가 새 Pod를 만드는가?

## 1. 이것이 무엇인가

### Container — 실제 서버 프로세스의 실행 환경

Container는 이미지에서 만들어진 격리된 실행 인스턴스다. 게임 서버 프로젝트에서는 일반적으로 컨테이너 하나에서 대표 서버 프로세스 하나가 실행된다고 생각하면 된다.

```text
runtime 이미지
  → Container
      → GameWorker Process
```

Container는 Kubernetes 리소스가 아니라 Container Runtime이 실제로 실행하는 프로세스 환경이다.

### Pod — Kubernetes의 최소 실행 단위

Kubernetes는 Container를 단독 리소스로 직접 배치하지 않는다. 하나 이상의 Container를 Pod 안에 정의하고 Pod 단위로 노드에 배치한다.

```text
Pod
  └─ 대표 Container
       └─ GameWorker Process
```

같은 Pod 안의 Container들은 다음 실행 환경을 밀접하게 공유한다.

- 같은 네트워크 공간과 Pod IP
- `localhost`를 통한 상호 통신
- 함께 정의된 Volume
- 같은 노드에 함께 배치되는 수명 주기

Pod 하나에 여러 Container를 넣을 수 있지만, 단지 서버 프로세스가 여러 개라는 이유로 한 Pod에 모두 넣는 것은 아니다. 서로 반드시 함께 배치되고 함께 살아야 하는 보조 Container일 때 멀티컨테이너 Pod를 고려한다.

게임 서버 프로젝트의 세 서버는 역할과 수명 주기가 다르므로 각각 별도의 Pod로 실행된다고 이해한다.

```text
ControlServer Pod
  └─ ControlServer Container

GameWorker Pod
  └─ GameWorker Container

GatewayServer Pod
  └─ GatewayServer Container
```

### Replica — 같은 실행 단위의 복제본

Replica는 같은 Pod Template으로 만들어진 실행 복제본을 뜻한다.

GameWorker replica가 세 개이고 Pod마다 대표 Container가 하나라면 다음과 같다.

```text
GameWorker replica 3
  ├─ Pod 1 → Container 1 → Process 1
  ├─ Pod 2 → Container 2 → Process 2
  └─ Pod 3 → Container 3 → Process 3
```

세 Pod는 같은 이미지와 기본 구조로 만들어질 수 있지만 각각 별도의 이름, IP, Container와 Process를 가진다. 애플리케이션 수준의 서버 ID가 필요하다면 별도 설정으로 전달해야 한다.

### ReplicaSet — Pod 개수를 유지하는 Controller

ReplicaSet은 특정 Pod가 원하는 개수만큼 존재하도록 유지하는 Kubernetes Controller다.

```text
원하는 replica: 3
현재 Pod: 2
ReplicaSet의 조치: Pod 1개 추가 생성
```

Pod 하나가 삭제되거나 노드 장애로 사라지면 ReplicaSet은 기존 Pod를 되살리는 것이 아니라 같은 Template으로 **새 Pod를 생성**한다.

### Deployment — 버전 변경까지 관리하는 상위 Controller

Deployment는 애플리케이션의 Pod Template과 replica 수를 선언하고, ReplicaSet을 통해 Pod를 유지한다. 새 이미지로 변경할 때 새로운 ReplicaSet을 만들고 이전 ReplicaSet과의 전환을 관리할 수 있다.

```text
Deployment
  ├─ 현재 ReplicaSet
  │    └─ 현재 버전 Pod들
  └─ 이전 ReplicaSet
       └─ 롤백 이력을 위해 축소된 상태로 남을 수 있음
```

평소 사용자는 ReplicaSet을 직접 만들기보다 Deployment를 작성한다. Deployment가 필요한 ReplicaSet을 만들고 관리한다.

## 2. 왜 필요한가

### Pod만 직접 만들면 생기는 문제

Pod 하나를 직접 만들 수는 있다. 하지만 그 Pod가 삭제되면 누가 같은 Pod를 다시 만들어야 하는가? 직접 만든 단독 Pod에는 원하는 개수를 지속적으로 유지해 줄 상위 Controller가 없다.

운영 애플리케이션은 일반적으로 Pod를 직접 관리하지 않고 Deployment 같은 Controller를 사용한다.

```text
직접 만든 Pod
  → 삭제됨
  → 자동으로 같은 Pod를 새로 만들 상위 Controller가 없음

Deployment가 관리하는 Pod
  → 삭제됨
  → ReplicaSet이 부족한 개수를 감지
  → 새 Pod 생성
```

### 역할을 나누는 이유

각 리소스가 담당하는 관심사가 다르다.

| 리소스 | 주된 책임 |
| --- | --- |
| Pod | Container를 함께 실행할 환경 정의 |
| ReplicaSet | 같은 Pod가 필요한 개수만큼 존재하도록 유지 |
| Deployment | Pod Template의 버전과 ReplicaSet 전환 관리 |

이 역할 분리 덕분에 복제 수 유지와 새 버전 배포를 같은 리소스 하나에 억지로 섞지 않고 계층적으로 처리할 수 있다.

## 3. 주요 구성 요소

### Pod Template

Deployment 안에는 새 Pod를 만들 때 사용할 Pod Template이 들어 있다.

Pod Template에는 다음과 같은 내용이 포함될 수 있다.

- 사용할 이미지와 tag
- 실행 명령
- 환경 변수
- 사용할 포트
- Volume Mount
- CPU와 메모리 요청량
- 상태 확인 방법
- Pod에 붙일 Label

Template이 바뀌면 Deployment는 새 버전의 ReplicaSet을 만들어 새 Pod로 교체할 수 있다. 구체적인 교체 전략은 11단계에서 배운다.

### Label과 Selector

Label은 Kubernetes 리소스에 붙이는 키-값 형식의 표식이다.

```text
app=game-worker
environment=dev
```

Selector는 원하는 Label을 가진 리소스를 선택하는 조건이다.

ReplicaSet은 Selector를 이용해 “내가 개수를 유지해야 하는 Pod”를 찾는다.

```text
ReplicaSet Selector: app=game-worker
  ├─ Pod A: app=game-worker → 관리 대상
  ├─ Pod B: app=game-worker → 관리 대상
  └─ Pod C: app=gateway     → 관리 대상 아님
```

Label과 Selector가 맞지 않으면 Controller가 Pod를 자신의 replica로 세지 못한다. 여러 Controller의 Selector가 잘못 겹쳐도 예상하지 못한 소유 문제가 생길 수 있다.

### Owner Reference — 리소스의 소유 관계

Kubernetes 리소스는 자신을 만든 상위 리소스를 Owner Reference로 기록할 수 있다.

```text
Deployment owns ReplicaSet
ReplicaSet owns Pod
```

이 관계는 다음 동작에 사용된다.

- 어떤 상위 Controller가 하위 리소스를 관리하는지 추적
- 상위 리소스 삭제 시 하위 리소스 정리
- Controller가 자신의 관리 대상을 구분

Pod를 수동으로 삭제해도 ReplicaSet의 원하는 개수는 바뀌지 않는다. ReplicaSet은 새 Pod를 만들어 다시 개수를 맞춘다.

### `replicas`

`replicas`는 원하는 Pod 복제 수다.

```text
replicas: 1 → Pod 1개 유지
replicas: 3 → Pod 3개 유지
replicas: 0 → 실행 Pod를 0개로 축소
```

replicas 값을 늘리는 것을 scale out, 줄이는 것을 scale in이라고 부른다. GameWorker는 상태와 서버 ID를 고려해야 하므로 값을 늘리는 것만으로 안전한 확장이 완성되는 것은 아니다. 이 제약은 10단계에서 배운다.

## 4. 다른 기술과 어떤 관계인가

### 이미지와 실행 명령

Deployment의 Pod Template은 Harbor에 저장된 이미지 주소와 tag를 지정한다. 같은 runtime 이미지를 사용해도 실행 명령을 바꾸면 서로 다른 서버 Pod를 만들 수 있다.

```text
ControlServer Deployment
  └─ 같은 runtime 이미지 + ControlServer 실행 명령

GatewayServer Deployment
  └─ 같은 runtime 이미지 + GatewayServer 실행 명령
```

이미지는 실행 재료이고 Deployment는 그 재료로 어떤 Pod를 몇 개 유지할지 선언한다.

### Helm

실제 구성에서는 Deployment YAML을 매번 복사해서 작성하지 않고 Helm Template으로 만든다. 환경별 values가 image tag, replica 수와 설정을 채운다.

```text
Helm Template + values
  → Deployment YAML
  → ReplicaSet
  → Pod
  → Container
```

Helm의 세부 구조는 14단계에서 배운다.

### ArgoCD

ArgoCD는 Git에 저장된 Helm·Kubernetes 선언을 클러스터에 반영한다. Deployment가 API Server에 생성된 다음부터 ReplicaSet과 Pod 유지 책임은 Kubernetes Controller가 맡는다.

```text
Git 상태와 Cluster API 상태의 차이 → ArgoCD가 동기화
원하는 Pod 상태와 실제 실행 상태의 차이 → Kubernetes Controller가 조정
```

### Service

새로 만들어진 Pod는 이전 Pod와 이름과 IP가 달라질 수 있다. 다른 서버가 계속 같은 주소로 접근하려면 Pod 앞에 안정적인 연결 지점이 필요하다. 이 역할을 Service가 담당하며 6단계에서 배운다.

### Deployment와 Rollout

Rollout은 Argo Rollouts가 제공하는 별도 리소스다. Deployment처럼 Pod Template과 replica를 관리하지만 배포 전략을 더 세밀하게 제어할 수 있다.

현재는 다음 차이만 기억한다.

```text
Deployment = Kubernetes 기본 배포 Controller
Rollout    = 고급 배포 전략을 위한 확장 Controller
```

GameWorker가 Rollout을 사용하는 구체적인 이유와 동작은 11단계에서 다룬다.

## 5. 실제 프로젝트에서는 어디에 사용되는가

현재 확인한 사내 개발용 배포 구성을 공개용 역할명으로 표현하면 다음과 같다.

```text
ControlServer Deployment
  └─ ReplicaSet
       └─ ControlServer Pod
            └─ runtime 이미지에서 ControlServer 실행

GatewayServer Deployment
  └─ ReplicaSet
       └─ GatewayServer Pod
            └─ runtime 이미지에서 GatewayServer 실행

GameWorker Rollout
  └─ ReplicaSet 계열 관리
       └─ 여러 GameWorker Pod
            └─ runtime 이미지에서 GameWorker 실행
```

확인된 의미는 다음과 같다.

- ControlServer와 GatewayServer는 Kubernetes 기본 Deployment로 관리된다.
- GameWorker는 일반 Deployment가 아니라 Argo Rollout으로 관리된다.
- 세 서버는 같은 runtime 이미지를 사용할 수 있으며 Pod Template의 실행 명령이 다르다.
- GameWorker 수를 늘리면 일반적으로 같은 Template을 사용한 Pod가 추가된다.
- 각 GameWorker에 필요한 애플리케이션 ID와 상태 소유권은 Kubernetes replica 수와 별도로 고려해야 한다.

아직 단정하지 않는 부분도 있다.

- GameWorker가 Rollout을 선택한 정확한 운영 배경은 추가 확인이 필요하다.
- 실제 라이브 환경에서도 같은 Controller 종류와 replica 정책을 사용하는지는 확인되지 않았다.
- 상태 확인 probe와 구체적인 업데이트 전략은 해당 단계에서 별도 확인한다.

## 6. 반드시 이해해야 할 핵심

1. **Pod는 Container와 같지 않다.** Pod는 하나 이상의 Container를 함께 실행하는 Kubernetes의 최소 배치 단위다.
2. **일반적으로 Pod 하나에 대표 서버 Container 하나가 실행된다.** 그러면 Pod 수와 서버 Process 인스턴스 수가 대응된다.
3. **ReplicaSet은 같은 Pod가 원하는 개수만큼 존재하도록 유지한다.**
4. **Deployment는 Pod Template과 버전 변경을 관리하고 ReplicaSet을 생성한다.**
5. **Deployment가 직접 Container를 실행하는 것은 아니다.** 계층을 따라 생성된 Pod를 노드의 kubelet과 Runtime이 실행한다.
6. **Pod가 사라지면 같은 Pod가 부활하는 것이 아니라 새 Pod가 만들어진다.** 이름과 IP가 달라질 수 있다.
7. **Label과 Selector가 Controller와 Pod를 연결한다.** Owner Reference는 실제 소유 관계를 기록한다.
8. **replicas는 인프라 실행 개수다.** 게임 세션과 서버 ID 같은 애플리케이션 상태까지 자동 해결하지 않는다.
9. **Rollout은 Deployment와 비슷한 상위 역할을 하는 확장 리소스다.** 자세한 배포 전략은 11단계에서 배운다.

한 문장으로 압축하면 다음과 같다.

> Deployment가 원하는 이미지·명령·개수를 Pod Template으로 선언하면 ReplicaSet이 그 수만큼 Pod를 유지하고, 각 Pod 안의 Container가 실제 게임 서버 Process를 실행한다.

## 7. 지금 단계에서 몰라도 되는 내용

다음 내용은 뒤에서 다루거나 선택 학습으로 남긴다.

- Service와 Pod IP 변화 대응 — 6단계
- ConfigMap과 Secret으로 서버별 설정 주입 — 7단계
- StatefulSet의 안정적인 이름과 저장 공간 — 8단계에서 기초 소개
- GameWorker의 자동·수동 스케일링과 서버 ID — 10단계
- Rolling Update, Recreate, Argo Rollout — 11단계
- readiness·liveness·startup probe — 12단계
- ReplicaSet 이름에 붙는 Template hash의 계산 방식
- Controller Manager와 Argo Rollouts Controller의 내부 구현
- Pod Sandbox와 Container Runtime의 상세 동작

지금은 `Deployment → ReplicaSet → Pod → Container → Process` 관계만 단단히 잡으면 된다.

## 8. 스스로 확인할 질문

1. Pod와 Container의 차이는 무엇인가?
2. 일반적인 게임 서버 구성에서 GameWorker Pod 세 개는 서버 Process 몇 개와 대응되는가?
3. ReplicaSet의 가장 중요한 책임은 무엇인가?
4. Deployment와 ReplicaSet을 따로 사용하는 이유는 무엇인가?
5. Deployment가 직접 Container를 실행한다고 말하면 왜 틀린가?
6. Deployment가 관리하는 Pod 하나를 수동으로 삭제하면 어떤 일이 일어나는가?
7. 새로 생성된 대체 Pod가 이전 Pod와 완전히 같은 개체가 아닌 이유는 무엇인가?
8. Label, Selector, Owner Reference는 각각 어떤 역할을 하는가?
9. `replicas: 3`이 GameWorker의 애플리케이션 상태와 서버 ID 문제까지 해결해 주지 않는 이유는 무엇인가?
10. Deployment와 Rollout은 지금 단계에서 어떤 차이로 이해하면 되는가?

### 통과 기준

다음 상황을 리소스 계층으로 설명할 수 있으면 5단계를 이해한 것이다.

> “GameWorker 세 개를 유지하도록 선언했다. 실제로 어떤 Kubernetes 리소스와 Container·Process가 만들어지며, 그중 Pod 하나가 사라지면 누가 무엇을 새로 만드는가?”

## 다음 단계 예고

5단계 학습을 완료했다. 다음 문서는 [[06_Service와_네트워크|6단계 — Service와 네트워크]]다. 교체될 때마다 이름과 IP가 달라질 수 있는 Pod에 다른 서버가 어떻게 안정적으로 접속하는지 배운다.

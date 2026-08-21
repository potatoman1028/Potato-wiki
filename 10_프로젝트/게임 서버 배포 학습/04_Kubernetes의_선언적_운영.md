---
type: 학습
created: 2026-08-20
updated: 2026-08-20
status: 완료
step: 4
completed: 2026-08-20
tags: [게임서버, Kubernetes, 클러스터, 선언적운영, desired-state]
---

# 4단계 — Kubernetes의 선언적 운영

상위 문서: [[00_프로젝트_홈|게임 서버 배포 학습]]  
이전 단계: [[03_Harbor와_이미지_태그]]

## 이번 단계의 목표

지금까지 Dockerfile로 이미지를 만들고 Harbor에 저장하는 과정을 배웠다. 이제 그 이미지를 서버 장비에서 실행하고, 장애가 생겨도 필요한 수만큼 계속 유지하는 시스템이 필요하다.

이 문서를 읽은 뒤에는 다음 문장을 자신의 말로 설명할 수 있어야 한다.

> Kubernetes는 어느 장비에서 어떤 명령을 한 번 실행하는 도구가 아니라, 선언한 원하는 상태를 실제 클러스터 상태와 계속 비교해 일치시키는 시스템이다.

이번 단계에서는 Kubernetes의 큰 구조만 배운다.

```text
배포 선언
  → Kubernetes API에 원하는 상태 저장
  → 적절한 노드 선택
  → Harbor에서 이미지 pull
  → 컨테이너 실행
  → 상태를 계속 관찰하고 차이가 생기면 복구
```

아직 Pod, Deployment, ReplicaSet의 세부 관계는 외우지 않는다. 이 용어들은 5단계에서 처음부터 연결한다.

## 1. 이것이 무엇인가

### Kubernetes

Kubernetes는 여러 서버 장비에서 컨테이너 애플리케이션을 배치하고 유지하는 컨테이너 오케스트레이션 플랫폼이다.

Docker를 사용하면 한 장비에서 이미지를 컨테이너로 실행할 수 있다. 그러나 컨테이너가 수십 개가 되고 장비도 여러 대가 되면 사람이 직접 다음 일을 계속 처리하기 어렵다.

- 어떤 장비에 컨테이너를 실행할지 선택한다.
- 필요한 수만큼 컨테이너를 유지한다.
- 컨테이너가 죽으면 다시 실행한다.
- 새 이미지로 순서대로 교체한다.
- 서버 간 연결에 사용할 안정적인 주소를 제공한다.
- 설정, 비밀값과 저장 공간을 연결한다.
- 각 컨테이너의 상태와 자원 사용을 확인한다.

Kubernetes는 이러한 운영 작업을 공통 리소스와 제어 루프로 자동화한다.

### 오케스트레이션

오케스트레이션은 여러 실행 요소를 원하는 구성에 맞게 배치하고 조정하는 것을 뜻한다.

오케스트라에서 지휘자가 모든 악기를 직접 연주하지 않는 것처럼, Kubernetes도 서버 애플리케이션의 비즈니스 로직을 실행하지 않는다. 어떤 컨테이너를 몇 개 실행하고 어디에 연결할지 조정한다.

```text
Docker 이미지가 답하는 질문
  “이 서버 프로세스를 실행하려면 무엇이 필요한가?”

Kubernetes가 답하는 질문
  “이 이미지를 어디에서 몇 개 실행하고, 원하는 상태를 어떻게 유지할 것인가?”
```

### 클러스터

Kubernetes가 관리하는 전체 환경을 클러스터(Cluster)라고 한다. 클러스터는 크게 컨트롤 플레인과 노드로 나뉜다.

```text
Kubernetes Cluster
  ├─ Control Plane
  │    └─ 클러스터 전체를 판단하고 조정
  └─ Node 1, Node 2, ...
       └─ 실제 애플리케이션 컨테이너 실행
```

클러스터가 항상 여러 물리 서버여야 하는 것은 아니다. 학습용 로컬 클러스터는 한 장비 안에 컨트롤 플레인과 하나의 노드를 함께 둘 수 있다.

## 2. 왜 필요한가

서버 한 대에서 컨테이너 하나를 시험 실행하는 것만으로는 Docker도 충분하다. Kubernetes가 필요한 이유는 실행보다 **지속적인 운영**에 있다.

예를 들어 GameWorker 세 개가 필요한 상황을 생각해 보자.

사람이 직접 운영하면 다음 작업을 반복해야 한다.

1. 여유가 있는 서버 장비를 찾는다.
2. Harbor에서 올바른 이미지를 내려받는다.
3. 서로 다른 ID와 설정으로 세 프로세스를 실행한다.
4. 실행 중인지 계속 확인한다.
5. 하나가 죽으면 원인을 확인하고 다시 실행한다.
6. 새 버전이 나오면 실행 순서와 중단 순서를 조정한다.

Kubernetes에서는 “GameWorker가 세 개 존재해야 한다”는 원하는 상태를 선언한다. Kubernetes는 현재 두 개만 정상이라면 하나를 더 만들려고 한다.

```text
원하는 상태: GameWorker 3개
현재 상태:   GameWorker 2개
차이:        1개 부족
Kubernetes:  1개를 추가로 생성
```

이것이 Kubernetes의 가장 중요한 사고방식이다.

## 3. 주요 구성 요소

### Control Plane — 판단하고 조정하는 영역

컨트롤 플레인은 클러스터 전체의 상태를 관리하고 다음 행동을 결정한다. 대표 구성 요소는 다음과 같다.

#### API Server

API Server는 Kubernetes의 중앙 출입구다.

- 사용자가 제출한 리소스 요청을 받는다.
- ArgoCD 같은 외부 도구의 요청을 받는다.
- 요청의 인증과 권한을 확인한다.
- 원하는 상태와 현재 상태를 조회하고 변경할 API를 제공한다.

`kubectl`, Helm, ArgoCD는 노드에 직접 명령하기보다 대부분 API Server를 통해 Kubernetes와 대화한다.

#### 상태 저장소

클러스터의 리소스와 원하는 상태를 보관하는 내부 저장소가 있다. 일반적으로 컨트롤 플레인의 분산 키-값 저장소가 이 역할을 한다.

게임 데이터나 서버 로그를 저장하는 DB가 아니다. Kubernetes 자체의 상태를 보관하는 공간이다.

#### Scheduler

Scheduler는 아직 실행 위치가 정해지지 않은 작업을 어느 노드에 배치할지 선택한다.

판단에는 다음과 같은 정보가 사용될 수 있다.

- 각 노드의 남은 CPU와 메모리
- 애플리케이션이 요청한 자원
- 특정 노드를 요구하거나 피하는 배치 규칙
- 장애 분산과 우선순위

Scheduler는 컨테이너를 직접 실행하지 않는다. **실행할 노드를 선택하는 역할**이다.

#### Controller

Controller는 원하는 상태와 현재 상태를 반복해서 비교하고 차이를 줄이는 제어 루프다.

```text
관찰 → 비교 → 조치 → 다시 관찰
```

예를 들어 원하는 개수가 세 개인데 현재 두 개라면 새 실행 단위를 만들도록 조치한다. 어떤 Controller가 무엇을 소유하고 유지하는지는 5단계에서 배운다.

### Node — 실제 컨테이너가 실행되는 장비

노드는 게임 서버 컨테이너가 실제로 실행되는 물리 서버 또는 가상머신이다.

대표 구성 요소는 다음과 같다.

#### kubelet

kubelet은 각 노드에서 동작하는 Kubernetes Agent다.

- API Server를 통해 자신에게 배정된 실행 요청을 확인한다.
- 필요한 이미지를 Harbor에서 가져오도록 컨테이너 Runtime에 요청한다.
- 컨테이너를 시작하고 상태를 관찰한다.
- 실행 결과와 상태를 컨트롤 플레인에 보고한다.

#### Container Runtime

Container Runtime은 이미지를 내려받고 실제 컨테이너 프로세스를 만드는 소프트웨어다.

Kubernetes 노드에 반드시 Docker Desktop이나 Docker Engine이 있어야 하는 것은 아니다. Kubernetes는 표준 인터페이스를 지원하는 전용 Runtime을 사용할 수 있다. 내부 구현은 지금 몰라도 된다.

#### 네트워크 구성 요소

컨테이너 실행 단위가 서로 통신하고 Service를 통해 연결되도록 노드의 네트워크 규칙을 구성한다. 구체적인 통신 방식은 6단계에서 배운다.

### Pod — 지금은 이름만 알기

Kubernetes는 컨테이너를 맨몸으로 직접 관리하지 않고 Pod라는 실행 단위 안에 둔다.

이번 단계에서는 다음 한 문장만 기억한다.

> Pod는 Kubernetes가 컨테이너를 실행하는 가장 작은 배치 단위다.

일반적인 게임 서버 구성에서는 Pod 하나에 대표 서버 컨테이너 하나가 실행된다고 우선 생각하면 된다. 정확한 관계와 예외는 5단계에서 배운다.

## 4. 다른 기술과 어떤 관계인가

### 선언적 상태는 어떻게 동작하는가

### 명령형과 선언형

명령형 방식은 수행할 행동을 직접 지시한다.

```text
“2번 서버에서 GameWorker 프로세스를 지금 하나 실행해.”
```

선언형 방식은 최종적으로 유지되어야 할 상태를 제출한다.

```text
“GameWorker rev-12345가 항상 세 개 정상 상태로 존재해야 해.”
```

선언형 방식에서는 Kubernetes가 어느 노드에서 무엇을 실행할지 판단하고, 상태가 달라지면 다시 맞춘다.

### Desired State와 Current State

- Desired State: 사용자가 선언한 원하는 상태
- Current State 또는 Actual State: 지금 클러스터에 실제로 존재하는 상태

```text
Desired State
  - 이미지: rev-12345
  - 개수: 3
  - 필요한 설정과 저장 공간

Current State
  - rev-12345 인스턴스 2개 정상
  - 인스턴스 1개 장애

Controller의 조치
  - 부족한 정상 인스턴스 생성 시도
```

중요한 점은 한 번 맞추고 끝나는 것이 아니라 계속 비교한다는 것이다.

### Reconciliation — 상태 맞추기

원하는 상태와 현재 상태의 차이를 줄이는 과정을 Reconciliation이라고 한다.

```text
1. 원하는 상태를 읽는다.
2. 현재 상태를 관찰한다.
3. 둘의 차이를 계산한다.
4. 필요한 리소스를 생성·수정·삭제한다.
5. 결과를 다시 관찰한다.
```

장애가 발생했을 때 Kubernetes가 복구를 시도하는 것도 이 반복 과정의 결과다. 다만 애플리케이션의 잘못된 코드나 DB 데이터까지 자동으로 고쳐주는 것은 아니다. 자기 복구의 범위와 한계는 12단계에서 다룬다.

### Kubernetes 리소스와 YAML

### Resource

Kubernetes에서 원하는 상태를 표현하는 API 객체를 리소스(Resource)라고 한다.

Pod, Deployment, Service, ConfigMap 등이 모두 서로 다른 종류의 리소스다. 지금은 이름을 외우지 말고 각각 실행, 유지, 연결, 설정 같은 역할을 선언한다고 이해한다.

### YAML Manifest

리소스의 원하는 상태를 YAML 형식으로 작성한 파일을 manifest라고 부른다.

일반적인 형태는 다음과 같다.

```yaml
apiVersion: <API 버전>
kind: <리소스 종류>
metadata:
  name: <리소스 이름>
  namespace: <논리 영역>
spec:
  <원하는 상태>
```

- `apiVersion`: 어떤 Kubernetes API 규격을 사용하는가
- `kind`: 어떤 종류의 리소스인가
- `metadata`: 이름, namespace, label 같은 식별 정보
- `spec`: 사용자가 원하는 상태
- `status`: Kubernetes가 관찰한 현재 상태로, 보통 시스템이 갱신

YAML은 단순 실행 스크립트가 아니다. API Server에 제출할 리소스의 원하는 상태를 표현한다.

### Helm과 ArgoCD가 있는 경우

게임 서버 프로젝트에서는 사람이 완성된 YAML을 매번 직접 제출하는 것이 중심 흐름이 아니다.

여기서 Helm은 GitLab과 같은 저장소 서비스가 아니다.

| 기술 | 지금 단계에서의 역할 |
| --- | --- |
| GitLab 같은 Git 저장소 서비스 | Chart와 환경별 설정 파일을 저장하고 변경 이력을 관리 |
| Helm | 저장된 Chart와 설정값을 읽어 Kubernetes YAML을 생성 |
| ArgoCD | Git의 변경을 감지하고 생성된 리소스를 Kubernetes에 반영 |
| Kubernetes | 반영된 원하는 상태에 맞춰 컨테이너를 실행하고 유지 |

Helm의 Chart, Template, Values가 정확히 무엇인지는 14단계에서 배운다. 지금은 **GitLab은 파일을 보관하고, Helm은 그 파일을 재료로 Kubernetes YAML을 만든다**는 차이만 알면 된다.

```text
Helm Chart + 환경별 values
  → Kubernetes YAML 렌더링
  → ArgoCD가 API Server에 반영
  → Kubernetes가 원하는 상태 유지
```

Helm은 YAML을 만들어 주고, ArgoCD는 Git에 있는 선언을 클러스터에 반영하며, Kubernetes는 제출된 상태를 실제 실행 상태로 유지한다. 각 도구의 책임은 다르다.

### kubectl, kubeconfig, Namespace

### kubectl

`kubectl`은 사용자가 Kubernetes API Server에 요청을 보내고 상태를 조회하는 명령행 도구다.

예를 들어 다음과 같은 용도로 사용한다.

- 클러스터와 노드 상태 확인
- 리소스 목록과 상세 상태 확인
- 컨테이너 로그 확인
- 테스트용 manifest 제출
- 장애 상태 조사

명령어를 지금 외울 필요는 없다. kubectl이 노드에 직접 접속해 프로세스를 실행하는 도구가 아니라 API Server의 클라이언트라는 점이 중요하다.

### kubeconfig

kubeconfig는 kubectl 같은 도구가 어느 클러스터의 API Server에 어떤 사용자와 권한으로 접속할지 알려주는 설정이다.

일반적으로 다음 정보를 포함한다.

- 접속할 클러스터와 API Server
- 사용자 또는 인증 정보
- Cluster와 User를 묶은 Context
- 현재 선택된 Context

kubeconfig에는 인증 정보가 포함될 수 있으므로 공개 저장소에 올리면 안 된다.

### Namespace

Namespace는 하나의 클러스터 안에서 리소스를 논리적으로 나누는 공간이다.

```text
한 Kubernetes Cluster
  ├─ dev namespace
  ├─ test namespace
  └─ operations namespace
```

환경이나 팀별로 이름이 같은 리소스를 분리하고 조회 범위를 줄이는 데 도움이 된다. 그러나 Namespace 하나만으로 완전한 보안 격리가 자동 제공되는 것은 아니다. 실제 권한과 네트워크 제한에는 별도 정책이 필요하다.

### 지금까지 배운 기술과 Kubernetes의 관계

지금까지 배운 흐름에 Kubernetes를 연결하면 다음과 같다.

```text
Dockerfile
  → runtime 이미지 생성
  → Harbor에 repository:tag로 저장
  → Git의 배포 선언에서 이미지 tag 지정
  → Helm이 Kubernetes 리소스 YAML 생성
  → ArgoCD가 API Server에 원하는 상태 제출
  → Scheduler가 실행 노드 선택
  → kubelet이 Harbor 이미지 pull 및 컨테이너 실행 요청
  → Controller가 원하는 상태와 현재 상태를 계속 비교
```

각 기술의 책임을 구분한다.

| 기술 | 하는 일 | 하지 않는 일 |
| --- | --- | --- |
| Dockerfile·Docker build | 이미지 제작 | 여러 노드의 실행 상태 유지 |
| Harbor | 이미지 저장과 제공 | 컨테이너 실행 |
| Helm | Kubernetes YAML 생성 | 클러스터 상태의 지속적 유지 |
| ArgoCD | Git의 선언을 클러스터와 동기화 | 컨테이너 프로세스 직접 실행 |
| Kubernetes | 선언된 실행 상태를 배치하고 유지 | 소스 빌드와 이미지 장기 보관 |

Kubernetes의 상태 복구와 ArgoCD의 self-heal도 구분해야 한다.

- Kubernetes Controller: API Server 안에 선언된 상태와 실제 실행 상태의 차이를 줄인다.
- ArgoCD self-heal: Git에 선언된 상태와 클러스터 API 안의 상태가 달라지면 Git 기준으로 되돌린다.

이 차이는 16단계에서 다시 다룬다.

## 5. 실제 프로젝트에서는 어디에 사용되는가

현재 확인한 사내 개발용 배포 흐름을 공개용으로 표현하면 다음과 같다.

1. TeamCity가 게임 서버 런타임 이미지를 빌드한다.
2. 리비전 tag를 붙여 Harbor에 저장한다.
3. 배포 설정 저장소에서 사용할 이미지 tag를 변경한다.
4. Helm이 환경별 값을 이용해 Kubernetes 리소스를 만든다.
5. ArgoCD가 변경을 감지하고 Kubernetes API Server에 반영한다.
6. Kubernetes가 ControlServer, GameWorker, GatewayServer에 필요한 실행 단위를 만든다.
7. Scheduler가 각 실행 단위를 노드에 배정한다.
8. 노드의 kubelet과 Container Runtime이 Harbor에서 이미지를 받아 컨테이너를 실행한다.
9. Controller가 선언된 개수와 버전이 유지되는지 계속 관찰한다.

아직 확인되지 않은 부분도 구분한다.

- 실제 라이브 클러스터의 노드 수와 구성은 확인되지 않았다.
- 컨트롤 플레인을 누가 어떤 방식으로 운영하는지는 외부 확인이 필요하다.
- 라이브 환경의 Namespace, 리소스 제한과 배치 정책은 확인되지 않았다.

사내 개발용 배포 저장소에서 본 구성을 라이브 클러스터의 사실로 확대해서는 안 된다.

## 6. 반드시 이해해야 할 핵심

1. **Kubernetes는 컨테이너를 여러 장비에 배치하고 원하는 상태로 유지하는 시스템이다.**
2. **클러스터는 컨트롤 플레인과 노드로 나뉜다.** 컨트롤 플레인은 판단하고 노드는 실제 컨테이너를 실행한다.
3. **API Server가 Kubernetes의 중앙 출입구다.** kubectl, Helm, ArgoCD는 이 API를 사용한다.
4. **Scheduler는 실행할 노드를 선택한다.** 컨테이너를 직접 실행하는 것은 노드의 kubelet과 Container Runtime 쪽이다.
5. **Controller는 원하는 상태와 현재 상태의 차이를 반복해서 줄인다.** 이것이 선언적 운영의 핵심이다.
6. **YAML manifest는 수행 명령 목록이 아니라 원하는 리소스 상태의 선언이다.**
7. **Kubernetes는 이미지를 만들거나 장기 보관하지 않는다.** Harbor에서 지정된 이미지를 pull해 실행한다.
8. **Pod는 Kubernetes의 최소 실행 단위다.** 자세한 소유 관계와 복제는 다음 단계에서 배운다.

한 문장으로 압축하면 다음과 같다.

> ArgoCD가 Kubernetes API에 원하는 게임 서버 상태를 제출하면, 컨트롤 플레인이 적절한 노드를 선택하고 각 노드가 Harbor 이미지를 컨테이너로 실행하며, Controller가 그 상태가 계속 유지되도록 조정한다.

## 7. 지금 단계에서 몰라도 되는 내용

다음 내용은 뒤에서 다루거나 선택 학습으로 남긴다.

- Pod, Deployment, ReplicaSet의 세부 소유 관계 — 5단계
- Service와 클러스터 네트워크 — 6단계
- ConfigMap과 Secret — 7단계
- Volume, PV, PVC — 8단계
- Helm의 Chart, Template, Values — 14단계
- Scheduler의 고급 배치 알고리즘 — 10단계
- readiness·liveness probe와 자기 복구 한계 — 12단계
- 컨트롤 플레인 내부 저장소의 분산 합의 방식
- Container Runtime과 CRI의 세부 구현
- 멀티 노드 클러스터 직접 구축
- Azure와 AWS 같은 클라우드별 Kubernetes 구성

지금은 `원하는 상태 선언 → API Server → 노드 선택 → 컨테이너 실행 → 계속 상태 맞추기`만 이해하면 된다.

## 8. 스스로 확인할 질문

1. Docker로 컨테이너를 실행할 수 있는데도 Kubernetes가 필요한 이유는 무엇인가?
2. Kubernetes 클러스터에서 컨트롤 플레인과 노드는 각각 어떤 역할을 하는가?
3. API Server, Scheduler, Controller, kubelet은 각각 무엇을 담당하는가?
4. 원하는 상태와 현재 상태의 차이는 무엇인가?
5. GameWorker를 세 개로 선언했는데 정상 실행이 두 개뿐이면 Kubernetes는 어떤 방향으로 행동하는가?
6. Scheduler가 직접 컨테이너를 실행한다고 말하면 왜 틀린가?
7. Kubernetes YAML의 `spec`과 `status`는 각각 누가 무엇을 표현하는가?
8. Harbor, ArgoCD, Kubernetes는 이미지 배포 과정에서 각각 어떤 책임을 가지는가?
9. kubectl과 kubeconfig는 각각 무엇인가?
10. Namespace를 사용하면 무엇을 분리할 수 있으며, Namespace만으로 완전한 보안 격리가 되지 않는 이유는 무엇인가?

### 통과 기준

다음 상황을 순서대로 설명할 수 있으면 4단계를 이해한 것이다.

> “Git의 배포 선언에는 GameWorker 세 개와 특정 이미지 tag가 적혀 있다. ArgoCD가 이를 Kubernetes에 반영한 뒤 어떤 구성 요소들이 원하는 상태를 실제 컨테이너 실행으로 바꾸며, 실행 중 하나가 사라지면 왜 다시 만들어지는가?”

## 다음 단계 예고

4단계 학습을 완료했다. 다음 문서는 [[05_Pod_Deployment_ReplicaSet|5단계 — Pod·Deployment·ReplicaSet]]이다. 지금까지 하나의 “실행 단위”라고만 부른 Pod가 무엇이며, Deployment와 ReplicaSet이 여러 Pod를 어떻게 소유하고 유지하는지 배운다.

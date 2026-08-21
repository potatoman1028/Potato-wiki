---
type: 학습
created: 2026-08-20
updated: 2026-08-20
status: 학습중
step: 6
tags: [게임서버, Kubernetes, Service, 네트워크, DNS, TCP]
---

# 6단계 — Service와 네트워크

상위 문서: [[00_프로젝트_홈|게임 서버 배포 학습]]  
이전 단계: [[05_Pod_Deployment_ReplicaSet]]

## 이번 단계의 목표

5단계에서 Pod는 사라지면 같은 Template으로 새로 만들어지며 이름과 IP가 달라질 수 있다고 배웠다. 그렇다면 다른 서버가 특정 Pod IP를 기억해서 접속하는 방식은 안정적이지 않다.

이번 단계의 핵심 관계는 다음과 같다.

```text
접속하는 쪽
  → 변하지 않는 Service 이름과 주소
  → 현재 정상인 Pod 중 하나
  → Pod 안의 Container와 서버 Process
```

이 문서를 읽은 뒤에는 다음 질문에 답할 수 있어야 한다.

- Pod IP를 직접 사용하면 왜 문제가 되는가?
- Service는 Pod를 어떻게 찾고 트래픽을 전달하는가?
- ClusterIP, NodePort, LoadBalancer는 무엇이 다른가?
- HTTP 웹 트래픽과 장시간 TCP 게임 연결은 무엇이 다른가?

## 1. 이것이 무엇인가

### Pod IP

각 Pod는 클러스터 네트워크에서 사용할 IP를 받는다. 다른 Pod는 이 IP를 이용해 통신할 수 있다.

하지만 Pod는 영구적인 서버 장비가 아니다.

```text
기존 Pod
  이름: game-worker-a
  IP: 10.0.1.15

Pod 장애 후 새로 생성
  이름: game-worker-b
  IP: 10.0.2.27
```

새 Pod는 같은 이미지와 설정으로 만들어져도 다른 이름과 IP를 받을 수 있다. 따라서 다른 서버 설정에 `10.0.1.15` 같은 Pod IP를 고정해서 넣으면 교체 후 연결이 끊어진다.

### Service

Service는 여러 Pod 앞에 놓는 안정적인 네트워크 접점이다.

Service는 다음 두 가지를 제공한다.

- 변하지 않는 가상 IP와 DNS 이름
- 현재 대상이 되는 Pod로 트래픽 전달

```text
game-worker Service
  ├─ GameWorker Pod A
  ├─ GameWorker Pod B
  └─ GameWorker Pod C
```

Pod A가 사라지고 Pod D가 생겨도 접속하는 쪽은 같은 Service 이름을 사용한다. Kubernetes가 Service의 실제 목적지 목록을 갱신한다.

Service가 Pod를 생성하거나 고치는 것은 아니다.

- ReplicaSet: Pod 개수를 유지
- Service: 현재 대상 Pod로 네트워크 트래픽 전달

### Service가 Pod를 찾는 방법

Service는 일반적으로 Selector를 사용해 특정 Label을 가진 Pod를 대상으로 선택한다.

```text
Service Selector
  app=game-worker

Pod A Label: app=game-worker → 대상
Pod B Label: app=game-worker → 대상
Pod C Label: app=gateway     → 대상 아님
```

Kubernetes는 선택된 Pod의 IP와 준비 상태를 EndpointSlice 같은 리소스에 반영한다. Service의 실제 목적지는 이 목록을 바탕으로 계속 바뀔 수 있다.

### Service DNS

Kubernetes는 Service에 클러스터 내부 DNS 이름을 제공한다.

공개용 예시는 다음과 같다.

```text
game-worker.dev.svc.cluster.local
```

- `game-worker`: Service 이름
- `dev`: Namespace
- `svc.cluster.local`: 클러스터 내부 Service DNS 영역

같은 Namespace에서는 짧게 `game-worker`만 사용해도 해석될 수 있다. 다른 Namespace의 Service라면 Namespace까지 포함한 이름을 사용하는 편이 명확하다.

애플리케이션은 Pod IP 목록 대신 Service DNS 이름을 설정으로 사용할 수 있다.

## 2. 왜 필요한가

### Pod의 수명과 네트워크 주소를 분리

Pod는 복구, 스케일링과 새 버전 배포 때문에 계속 바뀔 수 있다. Service는 변하는 Pod 집합과 접속하는 클라이언트 사이를 분리한다.

```text
접속하는 서버가 아는 것
  → Service 이름

Kubernetes가 관리하는 것
  → 현재 Pod IP 목록
```

접속하는 서버는 Pod가 교체될 때마다 설정을 바꿀 필요가 없다.

### 여러 Pod에 트래픽 분산

Service 대상 Pod가 여러 개라면 새 네트워크 연결을 대상 중 하나로 전달할 수 있다.

```text
Service
  ├─ 연결 1 → Pod A
  ├─ 연결 2 → Pod B
  └─ 연결 3 → Pod C
```

하지만 이것을 애플리케이션 수준의 완전한 부하 분산이나 게임 세션 분배와 같다고 보면 안 된다. Kubernetes Service는 특정 사용자가 어떤 GameWorker에 있어야 하는지 알지 못한다.

### 정상 대상으로만 연결

Pod가 Service의 Selector에 맞더라도 준비 상태가 아니면 일반적으로 정상 트래픽 대상에서 제외할 수 있다. 어떤 조건을 정상으로 볼지는 readiness probe로 판단하며 12단계에서 자세히 배운다.

```text
Pod A: Ready    → Service 대상
Pod B: NotReady → 일반 트래픽 대상에서 제외
```

Service가 애플리케이션 오류를 스스로 진단하는 것은 아니다. Kubernetes가 제공받은 준비 상태를 이용한다.

## 3. 주요 구성 요소

### ClusterIP

ClusterIP는 기본 Service 유형이다. 클러스터 내부에서 접근할 수 있는 가상 IP와 DNS 이름을 제공한다.

```text
다른 Pod
  → ClusterIP Service
  → 대상 Pod
```

ControlServer와 GameWorker처럼 클러스터 내부 서버끼리 통신할 때 기본 선택이 될 수 있다. 클러스터 외부 사용자는 일반적으로 ClusterIP에 직접 접속할 수 없다.

### NodePort

NodePort는 각 노드의 특정 포트를 열고 그 포트로 들어온 트래픽을 Service로 전달한다.

```text
클러스터 외부
  → Node IP:NodePort
  → Service
  → Pod
```

구조가 단순해 테스트에 사용할 수 있지만 외부 사용자가 노드 주소와 포트를 알아야 하고 포트 관리 범위도 제한적이다. 운영 외부 노출의 최종 형태라고 단정할 수는 없다.

### LoadBalancer

LoadBalancer 유형은 클러스터 외부에 안정적인 주소를 제공하는 Load Balancer와 Service를 연결한다.

```text
외부 Client
  → External Load Balancer
  → Kubernetes Service
  → Pod
```

실제 외부 Load Balancer 생성 방식은 클라우드나 사내 인프라 구성에 따라 다르다. 모든 로컬 클러스터에서 자동으로 외부 주소가 생기는 것은 아니다.

### Service Port와 Target Port

Service가 받는 포트와 Pod Container가 실제로 듣는 포트는 구분된다.

```text
Client
  → Service port
  → targetPort
  → Container Process
```

두 포트가 같을 수도 있고 다를 수도 있다. Service는 공개된 접점과 애플리케이션 내부 포트 사이의 연결을 선언한다.

### EndpointSlice

EndpointSlice는 Service가 실제로 연결할 Pod IP와 포트 목록을 효율적으로 표현하는 Kubernetes 리소스다.

```text
Service Selector
  → 대상 Pod 발견
  → EndpointSlice에 현재 목적지 반영
  → Service 트래픽이 목적지 중 하나로 전달
```

사용자가 EndpointSlice를 매번 직접 만들 필요는 없다. Service와 Pod의 Label 관계가 올바르면 Kubernetes Controller가 관리한다.

## 4. 다른 기술과 어떤 관계인가

### Deployment·ReplicaSet과 Service

Deployment와 ReplicaSet이 Pod를 만들고 교체하면 Service는 Label Selector를 이용해 현재 Pod 집합을 따라간다.

```text
Deployment
  → ReplicaSet
  → Pod A, B, C
       ▲
       │ Label로 선택
Service
```

Deployment와 Service는 서로 상하 소유 관계가 아니다. 같은 Label과 Selector를 통해 실행 대상과 네트워크 대상을 연결한다.

### DNS와 Service Discovery

Service Discovery는 애플리케이션이 통신 대상을 찾는 과정이다. Kubernetes에서는 Service DNS가 대표적인 발견 방법이다.

```text
ControlServer가 `game-worker` 조회
  → Cluster DNS가 Service 주소 응답
  → Service가 현재 GameWorker Pod로 전달
```

하지만 GameWorker가 개별 서버 ID로 등록되고 특정 인스턴스를 직접 찾아야 하는 구조라면 Service DNS 하나만으로 충분하지 않을 수 있다. 애플리케이션의 서버 등록과 발견 방식은 Kubernetes Service와 별도로 확인해야 한다.

### HTTP와 TCP

HTTP는 요청과 응답 단위가 비교적 명확하며, 웹 Load Balancer는 URL 경로나 Host 이름을 이해해 요청을 라우팅할 수 있다.

게임 서버 연결은 장시간 유지되는 TCP 연결일 수 있다. Kubernetes Service는 TCP 연결을 대상 Pod 하나에 전달하지만 게임 프로토콜 내부의 사용자나 채널 의미를 이해하지 못한다.

```text
TCP 연결 시작
  → Service가 Pod B로 전달
  → 연결이 유지되는 동안 Pod B와 통신
```

Pod B가 장애로 사라지면 기존 TCP 연결은 끊어진다. Service가 연결 중이던 세션을 다른 Pod로 자동 이전하지 않는다. 클라이언트 재접속과 세션 복구는 애플리케이션이 처리해야 한다.

### Ingress

Ingress는 주로 HTTP와 HTTPS 요청을 Host와 경로 규칙에 따라 Service로 전달하는 Kubernetes 리소스다. 실제로 동작하려면 Ingress Controller도 필요하다.

```text
HTTP Client
  → Ingress Controller
  → Ingress Rule
  → Service
  → Pod
```

일반적인 Ingress 규칙이 임의의 TCP 게임 프로토콜을 자동 처리하는 것은 아니다. raw TCP 외부 노출에는 LoadBalancer나 NodePort 또는 특정 Ingress Controller의 별도 TCP 기능을 고려할 수 있다.

Ingress와 HTTPS는 핵심 배포 흐름을 완주한 뒤 선택 학습으로 남긴다.

### Helm과 ArgoCD

Helm Template은 Service의 이름, Selector, Service 유형과 포트를 생성할 수 있다. ArgoCD는 Git에 저장된 이 선언을 Kubernetes API에 반영한다.

```text
Git의 Helm Chart와 values
  → Service YAML 생성
  → ArgoCD 동기화
  → Kubernetes Service 생성·갱신
```

## 5. 실제 프로젝트에서는 어디에 사용되는가

현재 확인한 사내 개발용 구성을 공개용 역할명으로 단순화하면 다음 연결을 생각할 수 있다.

```text
외부 Client
  → Gateway Service
  → GatewayServer Pod

ControlServer·GatewayServer
  → 내부 연결 또는 애플리케이션의 서버 발견 방식
  → GameWorker Pod들
```

확인된 범위는 다음과 같다.

- Kubernetes Service 리소스가 배포 구성에 포함되어 있다.
- 외부 Client가 사용하는 TCP 서버 포트를 노출하는 구성이 중요하다.
- ControlServer, GameWorker, GatewayServer 사이에는 서버 간 통신이 필요하다.
- Pod는 교체될 수 있으므로 고정 Pod IP에 의존해서는 안 된다.

아직 추가 확인이 필요한 부분도 명확히 나눈다.

- 세 서버 사이의 모든 연결이 Service DNS를 사용하는지는 확인이 필요하다.
- GameWorker 개별 인스턴스 발견에 Service, 직접 등록 또는 별도 메시징 시스템 중 무엇을 사용하는지 추가 확인이 필요하다.
- 실제 라이브 환경의 외부 Load Balancer와 방화벽 구성은 확인되지 않았다.
- 외부 Client 연결이 어떤 Service 유형과 네트워크 장비를 거치는지는 라이브 구성 확인이 필요하다.

따라서 위 그림은 개념을 위한 단순화이며 실제 라이브 네트워크 토폴로지로 단정하지 않는다.

### 게임 서버에서 특별히 주의할 점

1. Service는 새 TCP 연결을 Pod에 전달하지만 기존 연결의 상태를 옮기지 않는다.
2. 특정 사용자를 특정 GameWorker로 보내야 한다면 애플리케이션 수준의 라우팅이 필요할 수 있다.
3. Pod 교체 전에 신규 연결 차단과 기존 세션 정리가 필요할 수 있다.
4. Service가 정상으로 판단한 네트워크 대상과 게임 로직상 입장 가능한 서버가 항상 같지는 않을 수 있다.

이 내용은 롤아웃과 상태 확인 단계에서 다시 연결한다.

## 6. 반드시 이해해야 할 핵심

1. **Pod IP는 영구 주소가 아니다.** 새 Pod는 새 이름과 IP를 받을 수 있다.
2. **Service는 변하는 Pod 앞에 안정적인 IP와 DNS 이름을 제공한다.**
3. **Service는 Label Selector로 대상 Pod를 찾는다.** EndpointSlice에는 현재 실제 목적지가 반영된다.
4. **Service는 Pod를 생성하거나 복구하지 않는다.** Pod 수는 ReplicaSet 같은 Controller가 유지한다.
5. **ClusterIP는 내부, NodePort와 LoadBalancer는 외부 접근 경로를 제공할 수 있다.**
6. **Service port와 Container target port는 구분된다.**
7. **Kubernetes DNS는 Service 이름으로 통신 대상을 찾게 해 준다.**
8. **Service는 게임 세션과 서버 ID를 이해하지 못한다.** 애플리케이션 수준의 라우팅은 별도 문제다.
9. **Pod 장애 시 기존 TCP 연결은 다른 Pod로 자동 이전되지 않는다.** 재접속과 상태 복구가 필요하다.
10. **Ingress는 주로 HTTP·HTTPS용이다.** raw TCP 게임 트래픽과 같은 방식이라고 보면 안 된다.

한 문장으로 압축하면 다음과 같다.

> Service는 교체와 복제로 계속 달라지는 Pod 집합 앞에 안정적인 DNS와 포트를 제공하고 새 연결을 현재 대상 Pod로 전달하지만, 게임 세션의 의미나 끊어진 TCP 연결의 복구까지 담당하지는 않는다.

## 7. 지금 단계에서 몰라도 되는 내용

다음 내용은 뒤에서 다루거나 선택 학습으로 남긴다.

- readiness probe로 정상 Endpoint를 판단하는 구체적인 방법 — 12단계
- NetworkPolicy로 Pod 통신을 제한하는 방법 — 선택 학습
- Ingress Controller 선택과 HTTPS 인증서 — 선택 학습
- 외부 Load Balancer 제품과 클라우드별 구현
- kube-proxy, IPVS, eBPF와 Service 가상 IP의 내부 구현
- CNI 플러그인과 Pod 네트워크 구성 방식
- Headless Service와 StatefulSet DNS
- Session Affinity의 세부 옵션
- 게임 세션 재접속 프로토콜 설계

지금은 `Service DNS → Service → 현재 대상 Pod → Container Process` 관계만 이해하면 된다.

## 8. 스스로 확인할 질문

1. 다른 서버가 Pod IP를 설정에 직접 저장하면 안 되는 이유는 무엇인가?
2. Service가 제공하는 두 가지 핵심 기능은 무엇인가?
3. Service는 어떤 Pod를 대상으로 삼을지 어떻게 결정하는가?
4. ReplicaSet과 Service는 Pod에 대해 각각 어떤 책임을 가지는가?
5. ClusterIP, NodePort, LoadBalancer는 접근 범위가 어떻게 다른가?
6. Service port와 target port는 무엇이 다른가?
7. Service DNS를 사용하면 Pod 교체 시 애플리케이션 설정을 바꾸지 않아도 되는 이유는 무엇인가?
8. Ready 상태가 아닌 Pod를 Service 대상에서 제외해야 하는 이유는 무엇인가?
9. 장시간 연결된 GameWorker Pod가 장애로 사라지면 Service가 기존 TCP 세션을 다른 Pod로 옮길 수 있는가?
10. HTTP Ingress와 raw TCP 게임 트래픽을 같은 방식으로 다루면 안 되는 이유는 무엇인가?

### 통과 기준

다음 상황을 설명할 수 있으면 6단계를 이해한 것이다.

> “GameWorker Pod 세 개 중 하나가 삭제되고 새 IP를 가진 Pod가 생성됐다. 다른 서버는 왜 설정을 바꾸지 않고 계속 같은 Service 이름을 사용할 수 있으며, 이미 삭제된 Pod와 연결되어 있던 TCP 세션은 왜 별도로 복구해야 하는가?”

## 다음 단계 예고

확인 질문을 통과하면 [[00_프로젝트_홈|프로젝트 홈]]의 6단계를 완료로 바꾸고 **7단계 — ConfigMap과 Secret**으로 넘어간다. 다음 단계에서는 이미지에 고정하면 안 되는 환경별 설정과 민감 정보를 Pod의 Container에 어떻게 전달하는지 배운다.


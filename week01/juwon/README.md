# **CKA 강의 정리: Section 2 - 핵심 개념 (Core Concepts)**

## 1. 쿠버네티스 클러스터 아키텍처 (Cluster Architecture)

<aside>
📌

**섹션 요약**  

쿠버네티스 클러스터는 **마스터 노드**와 **워커 노드**로 구성되며, 애플리케이션을 안정적으로 배포·관리하기 위한 전체 구조와 각 노드의 역할을 다룬다.

</aside>

<details>
<summary><strong>1.1 개요 및 핵심 목적</strong></summary>

<aside>
💡

 쿠버네티스 아키텍처는 컨테이너화된 애플리케이션을 안정적으로 배포·운영하기 위한 **클러스터 구성도(청사진)** 이다.

</aside>

- **왜 아키텍처를 먼저 이해해야 할까?**
    - 쿠버네티스가 애플리케이션을 **어떻게 배포/관리**하는지 전체 그림을 보여준다.
    - 각 구성 요소가 **어떤 역할**을 하고 **어떻게 상호작용**하는지 이해하는 것은
        - CKA 시험의 핵심
        - 실무에서 쿠버네티스를 사용하는 기반
- **쿠버네티스의 핵심 목적 (두 가지)**
    - **애플리케이션 호스팅**
        - 컨테이너화된 애플리케이션을 **자동화된 방식**으로 호스팅한다.
    - **배포 및 통신 지원**
        - 필요에 따라 애플리케이션의 인스턴스를 **쉽게 배포**할 수 있게 한다.
        - 여러 서비스 간의 **통신을 원활하게 지원**한다.
- **이 목적을 위한 클러스터 구조**
    - 쿠버네티스는 다음 두 가지 노드 유형으로 클러스터를 구성한다.
        - **마스터 노드 (Master Node)**
            - 클러스터 전체를 **제어·관리**하는 노드
        - **워커 노드 (Worker Node)**
            - 실제로 **애플리케이션 컨테이너를 실행**하는 노드
    - **선박 비유**를 사용

</details>

<details>
<summary><strong>선박 비유로 이해하는 쿠버네티스</strong></summary>

<aside>
⛴️

**선박 비유**  

쿠버네티스 클러스터를 거대한 **항구(Port)** 로, 노드들을 **선박(Ships)** 으로 비유해서 이해할 수 있다.

</aside>

- **항구(쿠버네티스 클러스터)**
    - 여러 종류의 선박(노드)들이 드나들며 화물을 싣고 내리는 **작업 현장**이다.
- **화물선(워커 노드, Worker Node)**
    - 실제 컨테이너(애플리케이션)를 싣고 바다를 건너는 역할
    - 즉, 애플리케이션을 **직접 호스팅하고 실행**하는 작업의 주체
- **제어 선박(마스터 노드, Master Node)**
    - 화물선들을 모니터링하고 관리하는 **중앙 통제실** 역할
    - 하는 일
        - 어떤 컨테이너를 **어떤 화물선(워커 노드)에 실을지 계획**한다.
        - 선박들의 **상태를 추적**한다.
        - 전체 **운항 프로세스를 조율**한다.

</details>

---

<details>
<summary><strong>1.2 노드 구성: 마스터 노드 vs 워커 노드</strong></summary>

<aside>
🧩

**핵심 개념**  

쿠버네티스 클러스터는 **마스터 노드**와 **워커 노드**로 구성되며, 각 노드는 고유한 책임을 가진 여러 컴포넌트들의 집합이다.

</aside>

- **클러스터 구성 요약**

| 노드 유형 | 핵심 책임 |
| --- | --- |
| 마스터 노드(Master Node) | 클러스터의 두뇌(Brain)이다. 모든 관리, 계획, 스케줄링, 감시 작업을 수행한다. |
| 워커 노드(Worker Node) | 클러스터의 일꾼(Worker)이다. 마스터 노드의 지시를 받아 실제 애플리케이션 컨테이너를 호스팅하고 실행한다. |

</details>

<details>
<summary><strong>각 노드의 핵심 컴포넌트</strong></summary>

**마스터 노드 / 워커 노드 구성 요소 요약**

- **마스터 노드(Master Node)**
    - **ETCD**
        - 클러스터의 **모든 상태 정보**를 저장하는 데이터베이스
    - **Kube-API Server**
        - 클러스터의 **모든 작업을 조정하는 중앙 관문**
    - **Kube-Scheduler**
        - **파드를 어떤 워커 노드에 배치할지** 결정
    - **Kube-Controller-Manager**
        - 노드, 레플리카 등 다양한 **컨트롤러를 관리**
- **워커 노드(Worker Node)**
    - **Kubelet**
        - 마스터와 통신하며 **파드를 관리**하는 에이전트
    - **Kube-Proxy**
        - 노드 내 **네트워크 규칙을 관리**하여 서비스 통신을 지원
    - **Container Runtime**
        - 컨테이너를 실제로 실행하는 소프트웨어
        - 예: `containerd`

</details>

---

## 2. 마스터 노드: 클러스터의 두뇌

마스터 노드는 쿠버네티스 클러스터의 모든 제어 및 관리 작업을 수행하는 **중앙 통제 센터**이다.

마스터 노드를 구성하는 ETCD, API 서버, 스케줄러, 컨트롤러 매니저는 각자의 역할을 수행하며 유기적으로 협력한다.

이를 통해 클러스터가 항상 **원하는 상태(Desired State)** 를 유지하도록 보장한다.

---

<details>
<summary><strong>2.1 ETCD</strong></summary>

<details>
<summary><strong>무슨 역할인가</strong></summary>

- ETCD는 단순한 데이터베이스가 아니라 다음과 같은 특징을 가진다.
    - 클러스터의 모든 상태 정보(노드, 파드, 설정, 시크릿 등)를 저장하는 **분산형 신뢰 키-값 저장소**이다.
    - `kubectl get` 명령어로 확인하는 **모든 정보의 근원지**이다.
    - Key-Value Store 형태

</details>

<details>
<summary><strong>쿠버네티스 내 ETCD</strong></summary>

1. **상태 저장소**
    - 클러스터의 모든 정보(Nodes, Pods, Configs, Secrets 등)가 저장되는 **중앙 저장소**이다.
2. **변경 완료의 기준**
    - 클러스터에 대한 모든 변경 사항은 ETCD에 해당 정보가 **업데이트되어야만 완료**된 것으로 간주된다.

</details>

<details>
<summary><strong>배포 방식 비교</strong></summary>

- **From Scratch(수동 설치)**
    - ETCD 바이너리를 직접 다운로드하여 서비스로 구성한다.
    - 이때 `-advertise-client-urls` 옵션은 **Kube-API 서버가 ETCD에 접근할 수 있는 주소**를 지정하는 중요한 설정이다.
- **kubeadm 사용**
    - ETCD가 `kube-system` 네임스페이스 내에 **파드(Pod)** 형태로 배포된다.
    - 따라서 `kubectl`로 관리할 수 있다.

<aside>
📌

- ETCD **버전 2와 3은 명령어가 다르다.**
- v3 명령어를 사용하려면 **`ETCDCTL_API=3` 환경 변수를 반드시 설정**해야 한다.
- kubeadm 환경에서는 ETCD가 **파드로 실행**되므로 `kube-system` 네임스페이스에서 확인한다.
- HA 환경에서는 TLS로 보호되므로 ETCD에 접근할 때 `--cacert`, `--cert`, `--key`를 지정하여 **인증**해야 한다.
</aside>

</details>

</details>

---

<details>
<summary><strong>2.2 Kube-API Server</strong></summary>

<details>
<summary><strong>무슨 역할인가</strong></summary>

- Kube-API 서버는:
    - 클러스터와 상호작용할 수 있는 **유일한 관문(Access Point)** 이다.
    - ETCD와 **직접 통신하는 유일한 컴포넌트**이다.
    - 스케줄러, Kubelet 등 다른 모든 컴포넌트는 API 서버를 통해서만 ETCD의 정보에 접근하고 상태를 변경할 수 있다.

</details>

<details>
<summary><strong>요청 처리 흐름</strong></summary>

사용자가 파드 생성을 요청하면, API 서버는 다음 흐름으로 작업을 처리한다.

1. 사용자 요청을 접수한다.
    - `kubectl` 명령 또는 API 직접 호출을 통해 요청이 들어온다.
2. 사용자를 **인증(Authenticate)** 하고, 요청 내용이 유효한지 **검증(Validate)** 한다.
3. ETCD에 **노드가 할당되지 않은 파드 정보**를 업데이트한다.
4. Kube-Scheduler가 API 서버를 모니터링하다가 **새 파드**를 감지한다.
5. Scheduler가 파드를 배치할 **최적의 노드**를 결정하여 결과를 API 서버에 전달한다.
6. API 서버가 ETCD의 파드 정보를 **특정 노드가 할당된 상태**로 업데이트하고, 해당 노드의 Kubelet에게 **파드 생성을 지시**한다.
7. Kubelet이 파드를 생성하고, 상태를 다시 API 서버에 보고한다.
8. API 서버가 파드의 최종 상태(예: `Running`)를 ETCD에 업데이트하며 과정을 완료한다.

<aside>
📌

- API 서버 설정 위치는 **설치 방식에 따라 다르다.**
    - **kubeadm 설치 시**
        - API 서버가 **파드로 실행**되므로, 매니페스트 파일에서 설정을 확인한다.
    - **수동 설치 시**
        - 서비스 설정 또는 실행 옵션에서 확인한다.
- 실행 중인 **프로세스 옵션을 확인**하는 방식도 활용할 수 있다.
- 주요 옵션 중 하나는 `-etcd-servers` 이며,
    
    API 서버가 통신할 ETCD 서버의 위치를 지정한다.
    
</aside>

</details>

</details>

---

<details>
<summary><strong>2.3 Kube Controller Manager</strong></summary>

<details>
<summary><strong>무슨 역할인가</strong></summary>

- 컨트롤러 매니저는 **여러 컨트롤러들을 관리하는 단일 프로세스**이다.
- 여기서 **컨트롤러**란:
    - 현재 상태를 지속적으로 **모니터링(Observe)** 하고
    - 원하는 상태(Desired State)와의 **차이(Analyze Diff)** 를 분석한 뒤
    - 필요한 작업을 **실행(Act)** 하는 자동화 프로세스이다.
- 이 **제어 루프(Control Loop)** 를 끊임없이 수행함으로써 쿠버네티스의 **지능과 자동 복구 기능의 핵심** 역할을 한다.

</details>

<details>
<summary><strong>주요 컨트롤러 예시</strong></summary>

- **Node Controller(노드 컨트롤러)**
    - **역할**
        - 노드 상태를 모니터링하고 관리한다.
    - **동작 방식 (노드 헬스체크 관련 옵션)**
        - `node-monitor-period=5s`
            - 5초마다 노드 상태를 확인한다.
        - `node-monitor-grace-period=40s`
            - 응답이 없으면 40초 대기 후 `unreachable` 로 표시한다.
        - `pod-eviction-timeout=5m`
            - 5분 이상 `unreachable` 이면 해당 노드의 파드를 제거(evict)하여 재배치한다.
- **Replication Controller(복제 컨트롤러)**
    - **역할**
        - 정의된 수만큼의 파드가 **항상 실행**되도록 보장한다.
    - **동작 방식**
        - 파드가 종료되거나 삭제되면 즉시 **새 파드를 생성**한다.
        

<aside>
📌

- 컨트롤러 매니저 옵션 확인 방법은 **설치 방식에 따라 다르다.**
    - **kubeadm 설치 시**
        - 컨트롤러 매니저가 파드로 실행되므로, **파드 매니페스트에서 옵션을 확인**한다.
    - 실행 중인 **프로세스 옵션을 직접 확인**할 수도 있다.
- `controllers` 옵션을 사용해 **특정 컨트롤러를 활성/비활성화**할 수 있다.
    - : **모든 컨트롤러를 활성화**한다.
    - `-nodeipam` : `nodeipam` 컨트롤러를 **비활성화**한다.
</aside>

</details>

</details>

---

<details>
<summary><strong>2.4 Kube Scheduler</strong></summary>

<details>
<summary><strong>무슨 역할인가</strong></summary>

- 스케줄러의 핵심 역할:
    - **새로 생성된 파드를 어떤 워커 노드에 배치할지 결정**하는 것.
- 중요한 점:
    - 스케줄러는 **노드를 선택**만 한다.
    - 실제 파드를 생성하는 작업은 **각 노드의 Kubelet**이 수행한다.
    - 이 두 역할(노드 선택 vs 파드 생성)을 **명확하게 구분**해야 한다.

</details>

<details>
<summary><strong>스케줄링 프로세스</strong></summary>

스케줄러는 다음 **2단계 프로세스**로 최적 노드를 선택한다.

1. **필터링(Filtering)**
    - 파드가 요구하는 리소스(CPU, 메모리 등) 또는 **제약 조건을 만족하지 못하는 노드**를 후보군에서 제외한다.
2. **순위 부여(Ranking)**
    - 남은 노드에 점수를 매겨 **순위를 정한 뒤**, 가장 높은 점수를 받은 노드를 최종 선택한다.

<aside>
📌

- 스케줄러 옵션 확인 방법 역시 **설치 방식에 따라 다르다.**
    - **kubeadm 설치 시**
        - 스케줄러가 파드로 실행되므로, **파드 매니페스트에서 설정을 확인**한다.
    - 실행 중인 **프로세스 옵션**을 확인해 세부 동작을 파악할 수 있다.
</aside>

</details>

</details>

---

## 3. 워커 노드: 작업 실행의 현장

워커 노드는 마스터 노드의 지시를 받아 실제 애플리케이션 컨테이너를 호스팅하고 실행하는 역할을 수행한다.

이곳에서 **파드의 생명주기**가 관리되고, **서비스 간 네트워킹**이 실제로 이루어진다.

---

<details>
<summary><strong>3.1 Kubelet</strong></summary>

<details>
<summary><strong>무슨 역할인가</strong></summary>

- Kubelet은 각 워커 노드에 존재하는 **핵심 에이전트**이다.
- API 서버와 지속적으로 통신하며, 아래와 같은 지시를 받아 수행한다.
    - 파드를 **배포하라**
    - 파드를 **제거하라**

</details>

<details>
<summary><strong>주요 기능</strong></summary>

- **노드 등록**
    - 워커 노드를 API 서버에 등록하여 **클러스터의 일부**로 만든다.
- **파드 생성 및 관리**
    - 파드 명세(spec)에 따라 컨테이너 런타임(containerd 등)을 통해 컨테이너를 생성한다.
    - 파드의 **생명주기(Lifecycle)** 를 관리한다.
- **상태 보고**
    - 노드와 파드 상태를 주기적으로 모니터링하고 API 서버에 보고한다.

<aside>
📌

- kubeadm은 API 서버, 스케줄러 등과 달리 **Kubelet을 파드로 자동 배포하지 않는다.**
- 따라서 Kubelet은 **모든 노드(마스터 포함)** 에 설치되어 있어야 한다.
- 실행 중인 **프로세스를 확인하는 방식**으로 Kubelet의 상태를 점검할 수 있다.
</aside>

</details>

</details>

---

<details>
<summary><strong>3.2 Kube-Proxy</strong></summary>

<details>
<summary><strong>무슨 역할인가</strong></summary>

- 파드의 IP는 생성, 삭제, 재시작마다 변경될 수 있어 **불안정**하다.
- 이를 해결하기 위해 쿠버네티스는 **서비스(Service)** 라는 안정적인 추상화 계층을 제공한다.
- Kube-Proxy는 이 서비스를 실제 파드와 연결하기 위해 **네트워크 규칙**을 관리한다.
    - 예: `iptables` rule

</details>

<details>
<summary><strong>동작 원리</strong></summary>

- 서비스(Service)는 실제 IP를 가진 실체가 아니라 쿠버네티스 내부에 존재하는 **가상 컴포넌트**이다.
- Kube-Proxy는 각 노드에서:
    - 서비스의 가상 IP(ClusterIP)로 들어오는 트래픽을 감지
    - 해당 서비스에 연결된 파드 중 하나로 트래픽을 전달하는 규칙을 설정
- API 서버를 통해 **서비스/엔드포인트 변경을 감시(watch)** 하며 규칙을 **동적으로 업데이트**한다.

</details>

<details>
<summary><strong>배포 형태</strong></summary>

- Kube-Proxy는 **DaemonSet 형태**로 배포된다.
    - 노드가 추가될 때마다 해당 노드에 Kube-Proxy 파드가 **자동 배포**된다.

<aside>
📌

- Kube-Proxy는 `kube-system` 네임스페이스에서 **DaemonSet** 으로 확인한다.
</aside>

</details>

</details>

---

<details>
<summary><strong>3.3 컨테이너 런타임 (Container Runtime)</strong></summary>

<details>
<summary><strong>역할 정의</strong></summary>

- 컨테이너 런타임은 Kubelet의 지시를 받아 **실제 컨테이너를 실행하고 관리**하는 소프트웨어이다.
- 예시:
    - Docker
    - containerd
    - rkt

</details>

<details>
<summary><strong>Docker vs containerd</strong></summary>

- 다양한 런타임을 지원하기 위해 **CRI(Container Runtime Interface)** 라는 표준 인터페이스가 도입되었다.
- Docker는 CRI를 **직접 만족하지 않아** `dockershim` 이 필요했다.
    - 쿠버네티스 v1.24부터 `dockershim` 지원이 **제거**되었다.
- **containerd**는 CRI를 준수하므로 쿠버네티스에서 **직접 사용되는 대표 런타임**이다.
- 런타임과 상호작용하는 CLI 도구 예:
    - `crictl`
    - `nerdctl`

</details>

</details>

---

## 4. 쿠버네티스 핵심 오브젝트

---

<details>
<summary><strong>4.1 Pod</strong></summary>

<details>
<summary><strong>개념 정의</strong></summary>

파드는 **쿠버네티스에서 생성하고 관리할 수 있는 가장 작은 배포 단위**이며, 애플리케이션의 **단일 인스턴스**이다. 쿠버네티스는 컨테이너를 직접 배포하지 않고, 하나 이상의 컨테이너를 파드로 캡슐화하여 배포한다.

</details>

<details>
<summary><strong>파드와 컨테이너 관계</strong></summary>

- **1:1 관계**: 가장 일반적이며 하나의 파드가 하나의 컨테이너를 가진다. 확장은 파드를 추가 생성하는 방식이다.
- **Multi-Container Pods**: 주 컨테이너를 보조하는 헬퍼 컨테이너가 필요한 경우 하나의 파드에 여러 컨테이너를 함께 배치한다. 이 경우 네트워크와 스토리지를 공유하며 [`localhost`](http://localhost)로 통신할 수 있다.

</details>

<details>
<summary><strong>YAML을 이용한 파드 정의</strong></summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: nginx-container
    image: nginx
```

- `apiVersion`: 사용할 API 버전
- `kind`: 생성할 오브젝트 종류
- `metadata`: 이름, 레이블 등 식별 정보
- `spec`: 상세 명세

<aside>
📌

- 파드 생성, 조회, 상세 조회, 삭제, 적용은 기본이다.
- `--dry-run=client -o yaml`로 YAML 템플릿을 빠르게 생성하는 기술이 매우 중요하다.
</aside>

</details>

</details>

---

<details>
<summary><strong>4.2 ReplicaSet & ReplicationController</strong></summary>

<details>
<summary><strong>목적 분석</strong></summary>

ReplicaSet은 파드를 단독으로 사용할 때 발생하는 문제를 해결하며 다음 가치를 제공

- **고가용성(High Availability)**: 파드가 종료되거나 삭제되면 즉시 새 파드를 생성하여 지정된 수를 유지한다.
- **로드밸런싱/스케일링**: 트래픽 증가 시 파드 수를 늘려 부하를 분산한다.

</details>

<details>
<summary><strong>ReplicationController vs ReplicaSet</strong></summary>

- ReplicationController는 ReplicaSet의 구버전이다.
- ReplicationController는 `apiVersion: v1`을 사용한다.
- ReplicaSet은 `apiVersion: apps/v1`을 사용한다.
- ReplicaSet은 `matchExpressions` 등 더 풍부한 셀렉터를 지원한다.

</details>

<details>
<summary><strong>YAML 구조 분석</strong></summary>

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      type: front-end
  template:
    metadata:
      labels:
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
```

- `spec.replicas`: 유지할 파드 수
- `spec.selector`: 관리 대상 파드를 선택
- `spec.template`: 새 파드 생성 템플릿

<aside>
📌

스케일링 방법은 세 가지

1. YAML 수정 후 `kubectl apply -f`로 반영한다.
2. `kubectl scale --replicas=… -f` 로 파일 기반 스케일링한다.
3. `kubectl scale --replicas=… replicaset` 로 직접 스케일링한다. 이 방식은 원본 YAML과 불일치가 생길 수 있다.
</aside>

</details>

</details>

---

<details>
<summary><strong>4.3 Deployment</strong></summary>

<details>
<summary><strong>개념</strong></summary>

Deployment는 ReplicaSet을 관리하고 배포 전략을 제공하는 상위 오브젝트이다. 대부분의 경우 ReplicaSet을 직접 만들지 않고 Deployment를 통해 배포한다.

</details>

<details>
<summary><strong>핵심 기능 분석</strong></summary>

- **롤링 업데이트(Rolling Update)**: 구버전 파드를 점진적으로 신버전으로 교체하여 중단을 줄인다.
- **롤백(Roll Back)**: 문제가 있으면 이전 버전으로 되돌린다.

</details>

<details>
<summary><strong>YAML 구조</strong></summary>

Deployment YAML 구조는 ReplicaSet과 거의 동일하며 `kind`가 `Deployment`로 바뀐다.

<aside>
📌

- `--dry-run=client -o yaml`로 기본 템플릿을 빠르게 만든 뒤 수정하는 방식이 가장 효율적이다.
</aside>

</details>

</details>

---

<details>
<summary><strong>4.4 Service</strong></summary>

<details>
<summary><strong>필요성</strong></summary>

파드의 IP는 변경 가능하므로 직접 통신 대상으로 부적합하다. Service는 여러 파드를 하나의 그룹으로 묶고 안정적인 단일 진입점(endpoint)을 제공한다.

</details>

<details>
<summary><strong>서비스 타입</strong></summary>

- **NodePort**
    - 목적: 외부에서 노드 포트를 통해 파드에 접근한다.
    - 포트 관계
        - `targetPort`: 파드의 컨테이너 포트
        - `port`: 서비스(ClusterIP)에 할당되는 포트
        - `nodePort`: 외부 접근 노드 포트이며 기본 범위는 `30000~32767`
- **ClusterIP**
    - 목적: 클러스터 내부 통신에 사용한다.
    - 특징: 기본 서비스 타입
- **LoadBalancer**
    - 목적: 클라우드 로드밸런서를 자동으로 생성한다.
    - 특징: 로컬 환경에서는 동작하지 않을 수 있다.

<aside>
📌

- `kubectl expose` 를 사용하면 서비스를 빠르게 생성할 수 있다.
</aside>

</details>

</details>

---

<details>
<summary><strong>4.5 Namespace</strong></summary>

<details>
<summary><strong>개념</strong></summary>

네임스페이스는 하나의 물리적 클러스터 안에서 여러 논리적 격리 공간을 만드는 기능이다. 이를 통해 환경(개발/운영) 분리, 팀별 권한 제어, 리소스 제한이 가능하다.

</details>

<details>
<summary><strong>기본 네임스페이스</strong></summary>

- `default`: 기본 공간
- `kube-system`: 쿠버네티스 시스템 컴포넌트가 실행되는 공간
- `kube-public`: 모든 사용자가 읽을 수 있는 리소스 공간

</details>

<details>
<summary><strong>네임스페이스 간 통신(DNS)</strong></summary>

- 동일 네임스페이스: 서비스 이름만 사용 예) `db-service`
- 다른 네임스페이스: FQDN을 사용 예) [`db-service.dev](http://db-service.dev).svc.cluster.local`

</details>

<details>
<summary><strong>리소스 제한(ResourceQuota)</strong></summary>

ResourceQuota로 네임스페이스별 리소스 사용량을 제한할 수 있다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: '10'
    requests.cpu: '4'
    limits.memory: 10Gi
```

<aside>
📌

- 네임스페이스 생성, 조회, 특정 네임스페이스 리소스 조회, 전체 네임스페이스 리소스 조회는 필수이다.
- 기본 컨텍스트 변경은 시간 절약에 도움이 된다.
</aside>

</details>

</details>

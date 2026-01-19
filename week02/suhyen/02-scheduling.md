# 📑 개념
> 참고 문서: https://github.com/kodekloudhub/certified-kubernetes-administrator-course/tree/master/docs/03-Scheduling

<details>
<summary>Manual Scheduling</summary>

### 쿠버네티스 수동 스케줄링(Manual Scheduling)

#### 1. 스케줄링의 기본 원리: `nodeName`

모든 파드(Pod) 설정에는 `nodeName`이라는 필드가 있습니다. 기본적으로 이 필드는 비어 있으며, 평소에는 쿠버네티스의 **스케줄러(Scheduler)**가 적절한 노드를 찾아 이 필드를 자동으로 채워줍니다.

#### 2. 수동 스케줄링 방법

클러스터에 스케줄러가 없거나 특정 노드에 파드를 직접 할당하고 싶을 때 두 가지 방법을 사용할 수 있습니다.

방법 1: 파드 생성 시 `nodeName` 지정
- 파드를 생성할 때 YAML 파일의 `spec` 섹션 아래에 `nodeName`을 직접 작성합니다.
- 동작 방식: 파드 생성 시점에 이미 노드가 지정되어 있으므로 스케줄러를 거치지 않고 해당 노드에 바로 배치됩니다.
- 예시:
    ```yaml
    spec:
      containers:
      - name: nginx
        image: nginx
      nodeName: node02  # 특정 노드 이름을 직접 명시
    ```

방법 2: Binding 오브젝트 사용 (이미 생성된 파드용)
- 파드가 이미 생성되었지만 `nodeName`이 지정되지 않아 'Pending' 상태인 경우, 파드 정의를 직접 수정할 수 없습니다. 이때는 **Binding(바인딩) 오브젝트**를 생성하여 API를 통해 노드에 할당해야 합니다.
- 동작 방식: JSON 형태로 바인딩 정보를 작성하여 파드의 바인딩 API에 POST 요청을 보냅니다.
- Binding 예시:
    ```yaml
    apiVersion: v1
    kind: Binding
    metadata:
      name: nginx
    target:
      apiVersion: v1
      kind: Node
      name: node02
    ```

### 3. 요약
- 스케줄러의 역할: 파드의 `nodeName` 필드를 업데이트하는 것
- 수동 스케줄링: 사용자가 직접 `nodeName`을 입력하거나 `Binding` 객체를 사용하여 노드를 강제 지정하는 것

</details>

<details>
<summary>Labels and Selectors</summary>

### 레이블과 셀렉터(Labels and Selectors), 어노테이션(Annotations)

#### 1. 레이블(Labels)과 셀렉터(Selectors)의 개념

쿠버네티스에서 수많은 오브젝트를 체계적으로 관리하기 위해 사용하는 표준 방법입니다.
- 레이블(Labels): 각 아이템에 부착되는 속성(키-값 쌍)입니다. (예: "이 앱은 프론트엔드다", "이 앱은 버전 1이다")
- 셀렉터(Selectors): 레이블을 이용해 특정 오브젝트들을 필터링하거나 그룹화할 때 사용합니다.

#### 2. 레이블 지정 및 조회 방법

YAML 파일의 `metadata` 섹션 내에 `labels`를 정의합니다.
- Pod 정의 예시:
    ```yaml
    metadata:
      name: simple-webapp
      labels:
        app: App1
        function: Front-end
    ```
- 조회 명령어: 특정 레이블을 가진 파드만 확인하고 싶을 때 사용합니다.
    ```bash
    kubectl get pods --selector app=App1
    ```

#### 3. 오브젝트 간의 연결 (연계 원리)

쿠버네티스는 레이블과 셀렉터를 사용하여 서로 다른 오브젝트들을 연결합니다.
- ReplicaSet/Deployment: `spec.selector.matchLabels`에 정의된 레이블이 파드 템플릿(`template.metadata.labels`)의 레이블과 일치해야 해당 파드들을 관리할 수 있습니다.
- Service: 서비스 역시 `spec.selector`를 통해 특정 레이블을 가진 파드들을 찾아 트래픽을 전달합니다.

#### 4. 레이블(Labels)과 어노테이션(Annotations) 비교

어노테이션은 레이블과 비슷해 보이지만 용도가 다릅니다.
| 구분 | 레이블 (Labels) | 어노테이션 (Annotations) |
| --- | --- | --- |
| 주요 용도 | 오브젝트 그룹화 및 선택 (필터링) | 상세 정보 및 메타데이터 기록 |
| 연결성 | 서비스나 레플리카셋이 파드를 찾을 때 사용 | 시스템이 직접적인 로직에 사용하지 않음 |
| 예시 | `app: App1`, `env: prod` | `buildversion: 1.34`, `maintained-by: admin` |

</details>

<details>
<summary>Taints and Tolerations</summary>

### 테인트와 톨러레이션(Taints and Tolerations)

#### 1. 기본 개념

테인트와 톨러레이션은 **노드가 특정 파드만 수용하도록 제한**을 거는 기능입니다.
- 테인트(Taints): 노드에 설정하며, "허용되지 않은 파드는 오지 마라"고 거부하는 역할을 합니다.
- 톨러레이션(Tolerations): 파드에 설정하며, 노드의 테인트를 "견딜 수 있는(용인하는) 능력"을 부여합니다.

#### 2. 노드에 테인트 설정 (Taints)

노드에 테인트를 부여할 때는 `kubectl taint` 명령어를 사용합니다.
- taint 생성: `kubectl taint nodes <노드이름> key=value:taint-effect`
- taint 조회: `kubectl describe node <노드이름> | grep Taint`
- 테인트 효과(Taint Effect) 3가지:
    1. NoSchedule: 톨러레이션이 없으면 절대 스케줄링되지 않음 (기존 파드는 유지)
    2. PreferNoSchedule: 가급적 피하되, 자원이 부족하면 스케줄링될 수도 있음
    3. NoExecute: 톨러레이션이 없으면 스케줄링되지 않을 뿐만 아니라, 이미 실행 중인 파드도 쫓겨남(Evict)
        | 효과 | 새 파드 배치? | 기존 파드 유지? |
        |------|-------------|---------------|
        | **NoSchedule** | ❌ 불가 | ✅ 유지 |
        | **PreferNoSchedule** | 🤔 되도록 회피 | ✅ 유지 |
        | **NoExecute** | ❌ 불가 | ❌ 퇴거 |

#### 3. 파드에 톨러레이션 설정 (Tolerations)

파드가 테인트가 걸린 노드에 들어가려면, YAML 파일 `spec` 섹션에 `tolerations`를 추가해야 합니다.
- 설정 예시:
    ```yaml
    spec:
      containers:
      - name: nginx-container
        image: nginx
      tolerations:
      - key: "app"
        operator: "Equal"
        value: "blue"
        effect: "NoSchedule"
    ```

#### 4. 주의할 점

테인트와 톨러레이션을 이해할 때 가장 중요한 포인트는 다음과 같습니다.
- "테인트와 톨러레이션은 파드를 특정 노드로 보내는 기능이 아닙니다."
- 이 기능은 노드가 파드를 밀어내는 역할을 할 뿐입니다. 즉, 톨러레이션이 있는 파드는 테인트가 있는 노드에 들어갈 수만 있을 뿐, 반드시 그 노드로 간다는 보장은 없습니다. (다른 테인트 없는 노드로 갈 수도 있습니다.)
- 만약 파드를 특정 노드에 반드시 배치하고 싶다면, 테인트가 아니라 노드 셀렉터나 노드 어피니티를 사용하는 것이 좋습니다.

</details>

<details>
<summary>Node Affinity</summary>

### 노드 셀렉터(Node Selectors), 노드 어피니티(Node Affinity)

#### 1. 노드 셀렉터 (Node Selectors)

가장 간단하게 파드를 특정 노드에 배치하는 방법입니다.
- 작동 방식: 노드에 설정된 레이블(Label)을 기반으로 파드를 배치합니다.
- 설정 방법:
    1. 노드에 레이블 부여: `kubectl label nodes <노드명> size=Large`
    2. 파드 설정: `spec.nodeSelector` 필드에 해당 레이블을 명시합니다.
- 한계: "이 레이블이 있는 노드"라는 단순한 조건만 가능하며, "이 레이블이 없거나 여러 개 중 하나인 노드" 같은 복잡한 조건은 설정할 수 없습니다.

#### 2. 노드 어피니티 (Node Affinity)

노드 셀렉터의 한계를 극복하기 위한 고급 스케줄링 기법입니다.
- `In`, `NotIn`, `Exists` 등의 연산자를 사용하여 훨씬 복잡한 조건을 설정할 수 있습니다.
    - In: 나열된 값 중 하나라도 일치하는 노드 (예: Large 또는 Medium)
    - NotIn: 나열된 값이 없는 노드 (예: Small이 아닌 노드)
    - Exists: 특정 키(Key) 레이블이 존재하기만 하면 됨

#### 3. 노드 어피니티의 유형 (Types)

스케줄링 시점과 실행 시점의 강제성 정도에 따라 나뉩니다.
| 유형 | 특징 |
| --- | --- |
| requiredDuringScheduling... | 필수 조건. 조건에 맞는 노드가 없으면 파드가 생성되지 않음. |
| preferredDuringScheduling... | 선호 조건. 조건에 맞는 노드를 찾되, 없으면 다른 노드에라도 배치함. |
| ...IgnoredDuringExecution | 이미 실행 중인 파드는 노드 레이블이 바뀌어도 영향을 받지 않음. |

> 참고: 현재는 실행 중인 파드를 쫓아내는 `RequiredDuringExecution` 기능은 계획 단계에 있습니다.

#### 4. 테인트/톨러레이션 vs 노드 어피니티

두 기능은 상호 보완적인 관계입니다.
- 테인트와 톨러레이션: 노드가 파드를 **밀어내는(Repel)** 성격. (원치 않는 파드가 오는 것을 방지)
- 노드 어피니티: 파드를 특정 노드로 **끌어당기는(Attract)** 성격. (원하는 노드에 배치)


</details>

<details>
<summary>Resource Limits</summary>

### 리소스 제한(Resource Limits)

#### 1. 리소스와 스케줄링의 관계

쿠버네티스 클러스터의 각 노드는 한정된 CPU, 메모리, 디스크 자원을 가지고 있습니다.
- 자원 부족 시: 파드를 배치할 노드에 충분한 자원이 없으면 파드는 Pending(대기) 상태로 남게 됩니다.
- 원인 확인: `kubectl describe pod` 명령어로 이벤트를 확인하면 "Insufficient CPU" 또는 "Insufficient memory"와 같은 사유를 볼 수 있습니다.

#### 2. 리소스 요청(Requests) vs 제한(Limits)

쿠버네티스에서는 컨테이너별로 자원 사용량을 두 가지 단계로 정의할 수 있습니다.

① 리소스 요청 (Resource Requests)
- 컨테이너가 실행되기 위해 최소한으로 필요한 자원 양입니다.
- 스케줄러의 역할: 스케줄러는 이 '요청' 수치를 보고 해당 자원이 남아있는 노드를 찾아 파드를 배치합니다.
- 기본값: 별도 설정이 없으면 일반적으로 컨테이너당 `0.5 CPU`, `256Mi 메모리`를 기본 요청량으로 간주합니다. (환경에 따라 다를 수 있음)

② 리소스 제한 (Resource Limits)
컨테이너가 사용할 수 있는 최대 자원 한도입니다.
- 설정 이유: 하나의 파드가 노드의 모든 자원을 독점하여 다른 파드에 피해를 주는 것을 방지합니다.
- 기본값: 설정하지 않으면 보통 `1 CPU`, `512Mi 메모리` 등의 제한이 걸릴 수 있습니다.

#### 3. 설정 예시 (YAML)

`spec.containers` 하위의 `resources` 섹션에서 설정합니다.

```yaml
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    resources:
      requests:   # 최소 보장량
        memory: "1Gi"
        cpu: "1"
      limits:     # 최대 한도
        memory: "2Gi"
        cpu: "2"

```

#### 4. 리소스 제한 초과 시 발생하는 일 (중요)

파드가 설정된 `limits`를 초과하려고 하면 다음과 같은 현상이 발생합니다.

- CPU 초과 시: 쿠버네티스는 CPU 사용량을 제한(Throttling)합니다. 즉, 파드가 죽지는 않지만 동작이 매우 느려집니다.
- 메모리 초과 시: 파드는 시스템에 의해 강제 종료(Terminated)됩니다. 이때 발생하는 에러가 그 유명한 OOMKilled(Out Of Memory Killed)입니다.

#### 5. 요약

1. Requests: 스케줄링의 기준 (최소 이만큼은 있어야 배치가 됨).
2. Limits: 안전장치 (최대 이만큼까지만 사용 가능).
3. 주의사항: 설정은 파드 단위가 아니라 파드 안의 각 컨테이너 단위로 이루어집니다.

</details>

<details>
<summary>DaemonSets</summary>

### 데몬셋(DaemonSets)

#### 1. 데몬셋(DaemonSets)이란?

데몬셋은 레플리카셋(ReplicaSet)과 유사하게 여러 개의 파드 인스턴스를 실행하지만, 핵심적인 차이점이 있습니다. 데몬셋은 클러스터 내의 모든 노드(또는 특정 노드들)에 파드 사본을 딱 하나씩만 실행하도록 보장하는 컨트롤러입니다.

#### 2. 주요 사용 사례 (Use Cases)

모든 노드에서 공통적으로 실행되어야 하는 백그라운드 작업에 주로 사용됩니다.
- 로그 수집기: 각 노드에서 로그를 수집하여 중앙으로 전송 (예: Fluentd, Logstash)
- 모니터링 에이전트: 각 노드의 상태를 감시 (예: Prometheus Node Exporter, Datadog 에이전트)
- 네트워킹 설정: 클러스터 네트워크 관련 구성 (예: kube-proxy, Calico/Weave 등 CNI 플러그인)

#### 3. 데몬셋 정의 (Definition)

데몬셋의 설정 파일 구조는 레플리카셋과 매우 비슷합니다. 차이점은 `kind` 필드를 `DaemonSet`으로 지정한다는 것입니다.
- apiVersion: `apps/v1`
- kind: `DaemonSet`
- spec.selector: 어떤 파드를 관리할지 결정하는 셀렉터
- spec.template: 실제 노드에 띄울 파드의 명세
- YAML 예시:
    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: monitoring-daemon
    spec:
      selector:
        matchLabels:
          app: monitoring-agent
      template:
        metadata:
          labels:
            app: monitoring-agent
        spec:
          containers:
          - name: monitoring-agent
            image: monitoring-agent
    ```

#### 4. 주요 명령어

- 생성: `kubectl create -f daemon-set-definition.yaml`
- 목록 조회: `kubectl get daemonsets`
- 상세 정보 확인: `kubectl describe daemonset <이름>`


#### 5. 데몬셋의 작동 원리

1. 자동 배포: 클러스터에 새로운 노드가 추가되면, 데몬셋 컨트롤러가 이를 감지하고 해당 노드에 자동으로 파드를 배치합니다.
2. 자동 삭제: 노드가 클러스터에서 제거되면, 해당 노드에 있던 파드도 함께 삭제됩니다.
3. 스케줄링: 과거에는 `nodeName`을 직접 지정했으나, 현재 버전의 쿠버네티스에서는 기본 스케줄러와 노드 어피니티(Node Affinity) 기능을 사용하여 노드에 배치됩니다.

#### 6. 레플리카셋(ReplicaSet) VS 데몬셋(DaemonSet)

- 레플리카셋: 전체 클러스터에 **필요한 개수만큼** 파드를 띄움. (한 노드에 여러 개가 뜰 수도 있고, 안 뜰 수도 있음)
- 데몬셋: 각 노드당 **무조건 하나씩** 파드를 띄움. (모든 노드에 고르게 배치됨)

</details>

<details>
<summary>Static PODs</summary>

### 정적 파드(Static Pods)

#### 1. 정적 파드(Static Pods)란?

쿠버네티스의 제어부(API 서버 등) 없이 **kubelet이 특정 노드에서 직접 관리하는 파드**입니다.

일반적인 파드는 API 서버가 스케줄러를 통해 노드에 할당하지만, 정적 파드는 kubelet이 지정된 로컬 디렉토리의 파일을 감시하다가 파일이 생기면 파드를 생성하고, 삭제되면 파드를 종료합니다.

#### 2. 설정 방법

kubelet이 어떤 디렉토리를 감시할지 알려주는 방법은 두 가지가 있습니다.
- 방법 1: 직접 옵션 사용 (`--pod-manifest-path`)
    - kubelet 서비스를 실행할 때 `--pod-manifest-path=/etc/kubernetes/manifests`와 같이 경로를 직접 지정합니다.
- 방법 2: 설정 파일 사용 (`--config`)
- `--config` 옵션으로 설정 파일(예: `kubeconfig.yaml`)을 전달하고, 해당 파일 내부에 `staticPodPath: /etc/kubernetes/manifests`라고 정의합니다.

#### 3. 정적 파드 확인 및 관리

- 확인 방법: API 서버가 구동 중이지 않을 때는 노드에서 직접 `docker ps` (또는 `crictl ps`) 명령어를 사용하여 실행 중인 컨테이너를 확인해야 합니다.
- API 서버와의 관계: API 서버가 있는 경우, kubelet은 각 정적 파드에 대해 미러 파드(Mirror Pod)를 API 서버에 생성하여 `kubectl`로 상태를 볼 수 있게 해줍니다. 하지만 `kubectl`로 삭제할 수는 없으며, 반드시 호스트의 파일을 삭제해야 합니다.

#### 4. 정적 파드 vs 데몬셋 (DaemonSets)

두 개념 모두 노드당 파드를 실행한다는 점이 비슷해 보이지만, 목적과 관리 주체가 다릅니다.

| 구분 | 정적 파드 (Static Pods) | 데몬셋 (DaemonSets) |
| --- | --- | --- |
| 관리 주체 | Kubelet (노드별 직접 관리) | DaemonSet Controller (API 서버) |
| 주요 용도 | 컨트롤 플레인 컴포넌트 실행 (API 서버, 스케줄러 등) | 모니터링, 로그 수집 등 백그라운드 서비스 |
| 의존성 | API 서버가 없어도 실행 가능 | API 서버가 반드시 필요함 |
| 생성 방식 | 특정 디렉토리에 YAML 파일 배치 | API 서버에 오브젝트 생성 요청 |

#### 5. 핵심 요약

- 정적 파드는 "API 서버 없이 실행되는 독립적인 파드"입니다.
- kubelet이 특정 폴더를 감시하여 파드를 관리합니다.
- 쿠버네티스 클러스터 자체를 구성하는 핵심 서비스(etcd, api-server 등)를 띄울 때 주로 사용합니다.

</details>

<details>
<summary>Priority Classes</summary>

### 우선순위 클래스(Priority Classes)

#### 1. 우선순위 클래스(Priority Class)란?

쿠버네티스 클러스터에서 실행되는 파드들 사이에 **상대적인 중요도**를 정의하는 방법입니다. 자원이 부족할 때 어떤 파드를 먼저 실행하고, 어떤 파드를 희생(종료)시킬지 결정하는 기준이 됩니다.
- 용도: 제어부(Control Plane) 컴포넌트, 핵심 데이터베이스 등 중요한 워크로드가 자원 부족으로 인해 중단되지 않도록 보호합니다.
- 범위: Non-namespaced 오브젝트로, 한 번 생성하면 클러스터 내 모든 네임스페이스의 파드에서 사용할 수 있습니다.

#### 2. 우선순위 값의 범위

우선순위는 숫자로 정의하며, 숫자가 클수록 우선순위가 높습니다.
- 일반 애플리케이션: 약 -20억에서 10억 사이의 값을 가집니다.
- 시스템 크리티컬 파드: 쿠버네티스 내부 컴포넌트용으로 10억에서 20억 사이의 아주 높은 값을 가집니다.
- 기본값: 별도 지정이 없으면 파드의 우선순위는 0입니다.

#### 3. 주요 설정 및 사용법

① PriorityClass 생성    
- `value` 필드에 우선순위 숫자를 지정합니다.
    ```yaml
    apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
    name: high-priority
    value: 1000000
    globalDefault: false # 모든 파드에 기본 적용할지 여부
    description: "이 클래스는 매우 중요한 서비스용입니다."
    ```

② 파드에 적용    
- 파드 정의서(`spec`)의 `priorityClassName` 필드에 생성한 클래스 이름을 적어줍니다.
    ```yaml
    spec:
    priorityClassName: high-priority
    containers:
    - name: nginx
        image: nginx
    ```

#### 4. 선점 정책 (Preemption Policy)

새로운 고우선순위 파드가 배치되어야 하는데 노드에 자원이 없을 경우의 동작을 정의합니다.

- PreemptLowerPriority (기본값): 자리를 만들기 위해 실행 중인 낮은 우선순위의 파드를 즉시 종료(Evict)시키고 그 자리를 차지합니다.
- Never: 낮은 우선순위 파드를 죽이지 않습니다. 대신 대기열(Queue)의 맨 앞으로 가서 자원이 생길 때까지 기다립니다. (다른 낮은 순위 파드들보다 먼저 스케줄링될 기회를 가집니다.)

#### 5. 요약

1. 우선순위 결정: 숫자가 높을수록 스케줄러가 먼저 처리합니다.
2. 자원 확보: 중요한 파드를 위해 덜 중요한 파드를 쫓아낼 수 있습니다(선점).
3. 시스템 보호: 쿠버네티스 핵심 컴포넌트는 가장 높은 순위를 가져 항상 실행을 보장받습니다.

</details>

<details>
<summary>Multiple Schedulers</summary>

### 다중 스케줄러(Multiple Schedulers)

#### 1. 다중 스케줄러란?

쿠버네티스 클러스터에서는 기본 스케줄러(`default-scheduler`) 외에도 **여러 개의 스케줄러를 동시에 실행**할 수 있습니다. 특정 애플리케이션에 특화된 사용자 정의(Custom) 스케줄링 로직이 필요한 경우에 사용합니다.

#### 2. 추가 스케줄러 배포 방법

사용자 정의 스케줄러는 바이너리 파일을 직접 다운로드하여 실행하거나, 쿠버네티스 내부의 파드(Pod) 또는 디플로이먼트(Deployment) 형태로 배포할 수 있습니다.
- 설정 시 주의사항: 각 스케줄러는 서로 구분되는 고유한 이름을 가져야 합니다. (예: `my-custom-scheduler`)
- kubeadm 환경: 보통 `/etc/kubernetes/manifests` 폴더에 스케줄러 정의 파일을 넣어 정적 파드로 실행합니다.

#### 3. 사용자 정의 스케줄러 사용법

특정 파드를 기본 스케줄러가 아닌 본인이 만든 스케줄러로 배치하고 싶다면, 파드 정의서(`spec`)에 `schedulerName` 필드를 추가하면 됩니다.
- 참고: 이 필드를 생략하면 쿠버네티스는 자동으로 `default-scheduler`를 사용합니다.
- 파드 YAML 예시:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
      schedulerName: my-custom-scheduler  # 사용할 스케줄러 이름 명시
    ```

#### 4. 모니터링 및 검증

스케줄러가 정상적으로 동작하는지 확인하는 방법은 다음과 같습니다.
- 이벤트 확인: 어떤 스케줄러가 파드를 스케줄링했는지 확인합니다.
    - `kubectl get events`
- 로그 확인: 스케줄러의 동작 과정이나 에러 로그를 확인합니다.
    - `kubectl logs my-custom-scheduler -n kube-system`

</details>

<details>
<summary>Admission Controllers</summary>

### 어드미션 컨트롤러 (Admission Controllers)

#### 1. 어드미션 컨트롤러란?

어드미션 컨트롤러는 쿠버네티스 API 서버에 요청이 전달되었을 때, 인증(Authentication)과 인가(Authorization) 단계를 거친 직후, 요청이 최종적으로 etcd에 저장되기 전에 해당 요청을 가로채서 검사하거나 수정하는 '게이트키퍼(Gatekeeper)' 역할을 합니다.

#### 2. 왜 필요한가요? (RBAC와의 차이)

단순히 "누가 이 행동을 할 수 있는가"를 정하는 것만으로는 부족한 보안 및 운영 정책을 강제하기 위해 사용합니다.
- RBAC (역할 기반 접근 제어): "사용자 A가 파드를 생성할 수 있는가?" (권한 유무 확인)
- 어드미션 컨트롤러: "생성되는 파드가 보안 정책에 적합한가?" (설정 내용 검증)
    - 예시 1: 특정 사내 레지스트리(Private Registry)가 아닌 외부 이미지(Docker Hub 등) 사용 금지
    - 예시 2: `latest` 태그 사용 금지 (버전 관리 목적)
    - 예시 3: 루트(`root`) 권한으로 실행되는 컨테이너 생성 차단

#### 3. 주요 기능 및 예시 

어드미션 컨트롤러는 단순히 요청을 거절(Validate)하는 것뿐만 아니라, 필요에 따라 요청 내용을 수정(Mutate)하기도 합니다.

| 컨트롤러 명칭 | 주요 역할 | 특징 |
| --- | --- | --- |
| AlwaysPullImages | 파드 생성 시 이미지를 항상 새로 내려받도록 강제 | 보안 강화 |
| DefaultStorageClass | PVC 생성 시 스토리지 클래스가 없으면 기본값으로 자동 추가 | 사용자 편의성(Mutating) |
| NamespaceExists | 존재하지 않는 네임스페이스에 리소스를 생성하려는 요청을 거부 | 검증(Validating) |
| NamespaceLifecycle | 삭제 중인 네임스페이스에 생성을 막고, `default` 등 핵심 네임스페이스 삭제를 방지 | 안정성 유지 |
| ResourceQuota | 특정 네임스페이스가 사용할 수 있는 자원 총량을 초과하는 요청 거부 | 자원 관리 |

#### 4. 설정 및 관리 방법

어드미션 컨트롤러는 API 서버의 설정 플래그를 통해 관리할 수 있습니다.
= 활성화된 플러그인 확인(kube-apiserver 설정 확인)
    - `kube-apiserver -h | grep enable-admission-plugins`
- 활성화 및 비활성화
    - `--enable-admission-plugins`: 사용할 플러그인 목록을 추가 (예: `NamespaceAutoProvision`)
    - `--disable-admission-plugins`: 기본으로 활성화된 플러그인 중 사용하지 않을 것을 지정
- kubeadm 환경: `/etc/kubernetes/manifests/kube-apiserver.yaml` 파일을 수정하여 반영

</details>

<details>
<summary>Validating and Mutating Admission Controllers</summary>

### 검증 및 변조 어드미션 컨트롤러(Validating and Mutating Admission Controllers)

#### 1. 어드미션 컨트롤러의 두 가지 유형

어드미션 컨트롤러는 인증과 권한 부여가 끝난 후, 요청이 API 서버에 저장되기 직전에 가로채서 검사하거나 수정하는 역할을 합니다.

| 유형 | 역할 | 특징 |
| --- | --- | --- |
| 변조(Mutating) | 요청 내용을 변경함 | 예: PVC 생성 시 지정이 없으면 기본 StorageClass를 자동으로 추가 |
| 검증(Validating) | 요청의 유효성 검사 | 예: 지정된 네임스페이스가 존재하는지 확인하고 없으면 요청 거부 |

- 실행 순서: 보통 변조(Mutating) 컨트롤러가 먼저 실행되고, 그 다음에 검증(Validating) 컨트롤러가 실행됩니다. 이는 변조 컨트롤러가 바꾼 내용까지 포함해서 최종적으로 검증하기 위함입니다.

#### 2. 어드미션 웹후크 (Admission Webhooks)

쿠버네티스에서 기본으로 제공하는 기능 외에 사용자 정의 로직을 추가하고 싶을 때 웹후크를 사용합니다.
- MutatingAdmissionWebhook: 사용자 정의 로직으로 요청 객체를 수정합니다.
- ValidatingAdmissionWebhook: 사용자 정의 로직으로 요청을 허용할지 거부할지 결정합니다.

작동 원리
1. 사용자가 API 요청을 보냅니다.
2. 쿠버네티스 API 서버가 설정된 외부 웹후크 서버로 `AdmissionReview` 객체(JSON 형식)를 보냅니다.
3. 웹후크 서버는 로직을 처리한 후 허용 여부(`allowed: true/false`)와 수정 내용(Patch)을 담아 응답합니다.

#### 3. 사용자 정의 웹후크 설정 단계

사용자 정의 어드미션 컨트롤러를 구축하는 과정은 크게 두 단계입니다.

① 웹후크 서버 배포
- API 서버의 요청을 받아 처리할 수 있는 서버를 만듭니다 (Go, Python 등 사용 가능).
    - 변조(Mutate) API: JSON Patch 방식(add, remove, replace 등)을 사용하여 객체를 수정하는 로직을 구현합니다.
    - 검증(Validate) API: 특정 조건에 따라 `true` 또는 `false`를 반환하는 로직을 구현합니다.
- 보안을 위해 반드시 TLS(HTTPS) 통신이 필요합니다.

② 웹후크 설정 오브젝트 생성
- 쿠버네티스에 언제 어떤 웹후크 서버를 호출할지 알려주는 설정을 생성합니다.
- 설정 예시 (ValidatingWebhookConfiguration):
    - clientConfig: 웹후크 서버의 주소(URL) 또는 클러스터 내부 서비스 이름과 CA 번들(TLS 인증서)을 지정합니다.
    - rules: 어떤 작업(CREATE, DELETE 등)이나 리소스(Pods, Deployments 등)에 대해 이 웹후크를 실행할지 정의합니다.

#### 4. 핵심 요약

- Mutating: 생성 전에 값을 "바꾸는" 컨트롤러 (먼저 실행)
- Validating: 생성해도 되는지 "검사하는" 컨트롤러 (나중에 실행)
- Webhook: 쿠버네티스 외부의 커스텀 로직 서버를 연결하여 기능을 확장하는 방법

</details>

# 🧪 실습
> Lab 문제를 풀면서 헷갈리는 개념 중심으로 정리

<details>
<summary>Manual Scheduling</summary>

#### 2. Why is the POD in a pending state? Inspect the environment for various kubernetes control plane components.

1. pod에 할당된 node가 없다: `Node: <none>`
```
controlplane ~ ➜ kubectl describe pod nginx
Name:             nginx
Namespace:        default
Priority:         0
Service Account:  default
Node:             <none> 
Labels:           <none>
Annotations:      <none>
Status:           Pending
IP:               
IPs:              <none>
Containers:
nginx:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:
    /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-l8rxn (ro)
Volumes:
kube-api-access-l8rxn:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                            node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
```

2. 현재 클러스터 내에 스케줄러(kube-scheduler) 파드가 실행 중이지 않음
```
controlplane ~ ➜ kubectl get pods -n kube-system | grep scheduler

(출력 없음)
```
=> 정답: No Scheduler Present

</details>

<details>
<summary>Labels and Selectors</summary>

> 2: 라벨 필터링 / 4; 여러 라벨 필터링

#### 2. How many PODs are in the finance business unit (bu)?

라벨 필터링: `-l` 옵션 사용

=> 정답: kubectl get pods -l bu=finance

#### 4. Identify the POD which is part of the prod environment, the finance BU and of frontend tier?

여러 라벨 필터링: `-l` 옵션 사용 + 쉼표간 띄어쓰기 없음 주의

=> 정답: kubectl get pod -l env=prod,bu=finance,tier=frontend

</details>

<details>
<summary>Taints and Tolerations</summary>

> 3: node에 taint 만들기 / 7: pod에 toleration 적용하기 / 10: node에 taint 없애기

#### 3. Create a taint on node01 with key of spray, value of mortein and effect of NoSchedule.

node에 taint 만들기: `kubectl taint nodes <노드이름> key=value:taint-effect`

=> 정답: kubectl taint node node01 spray=mortein:NoSchedule

#### 7. Create another pod named bee with the nginx image, which has a toleration set to the taint mortein.

pod에 toleration 적용하기

1) 파드 기본 yaml 만들고: `kubectl run bee --image nginx --dry-run=client -o yaml > bee.yaml`

2) bee.yaml에 toleration 추가하기: `vim bee.yaml`
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: bee
      name: bee
    spec:
      tolerations:
      - key: spray
        value: mortein
        effect: NoSchedule
      containers:
      - image: nginx
        name: bee
        resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
    status: {}
    ```

3) bee 파드 생성: `kubectl create -f bee.yaml`

#### 10. Remove the taint on controlplane, which currently has the taint effect of NoSchedule.

node에 taint 없애기: `kubectl taint nodes <노드이름> <키>=<값>:<효과>-`

=> 테인트를 설정할 때 사용했던 명령어 맨 끝에 마이너스(-) 기호만 붙여주면 된다.

=> 정답: kubectl taint node controlplane node-role.kubernetes.io/control-plane:NoSchedule-

</details>

<details>
<summary>Node Affinity</summary>

> 3: 노드에 라벨 붙이기 / 6: 노드에 노드 어피니티 추가 / 8: 시스템 노드에 노드 어피티니 추가

#### 3. Apply a label color=blue to node node01

노드에 라벨 붙이는 명령어: `kubectl label nodes <노드이름> <키>=<값>`

=> 정답: kubectl label node node01 color=blue


6. Set Node Affinity to the blue deployment to place the pods on node01 only. Ensure that node01 has the label color=blue.

> Requirements:   
> - Use requiredDuringSchedulingIgnoredDuringExecution node affinity    
> - Key: color    
> - Value: blue    
> If the label is not already set, apply it to node01 before updating the deployment.   

노드에 노드 어피니티 추가
1. kubectl edit deployment blue
2. blue yaml 수정: `template` 하단의 `spec`에 추가
    ```yaml
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: color
                operator: In
                values:
                - blue   
    ```

참고링크: https://kubernetes.io/ko/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/

#### 8. Create a new deployment named red with the nginx image and 2 replicas, and ensure it gets placed on the controlplane node only.

> Use the label key - node-role.kubernetes.io/control-plane - which is already set on the controlplane node.

시스템 노드에 노드 어피티니 추가: `operator: Exists` + `values: (없음)`
1. kubectl edit deployment red
2. red yaml 수정: `template` 하단의 `spec`에 추가
    ```yaml
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/control-plane
                operator: Exists
    ```

</details>

<details>
<summary>Resource Limits</summary>

#### 7. The elephant pod runs a process that consumes 15Mi of memory. Increase the limit of the elephant pod to 20Mi.

존재하는 pod의 설정정보를 yaml로 만들기: `kubectl describe pod elephant -o yaml > elephant.yaml`

</details>

<details>
<summary>DaemonSets</summary>

> 6: DeamonSet 생성

#### 6. Deploy a DaemonSet for FluentD Logging.

> Use the given specifications.   
> - Name: elasticsearch    
> - Namespace: kube-system   
> - Image: registry.k8s.io/fluentd-elasticsearch:1.20    

주의할 점
1. DaemonSets (X) / DaemonSet (O)
2. kubectl create으로 daemonset을 생성할 수 없다. 

DeamonSet 생성  
1. deployment yaml 만들어서 kind를 daemonset으로 변경
2. 불필요한 필드 삭제: replicas 삭제, {} 필드 삭제

</details>

<details>
<summary>Static PODs</summary>

> 1: static pod 이름 / 5: static pod yaml 위치 / 8: static pod 생성 / 10: static pod 삭제

#### 1. How many static pods exist in this cluster in all namespaces?

pod 이름 끝에 해시값이 안붙은게 static pod

#### 5. What is the path of the directory holding the static pod definition files?

static pod yaml 파일 기본 위치: `/etc/kubernetes/manifests`

#### 8. Create a static pod named static-busybox that uses the busybox image , run in the default namespace and the command sleep 1000.

> Note: Here we asked to create a static pod use the knowledge you gained in the previous questions

static pod 생성  
1. `cd /etc/kubernetes/manifests/` 로 이동
2. `kubectl run static-busybox --image=busybox --dry-run=client -o yaml --command -- sleep 1000 > static-busybox.yaml` (--command -- sleep 위치 주의)
3. static pod는 kubectl apply를 따로 할 필요가 없음. 파일이 폴더에 들어가는 순간, 노드의 kubelet이 이를 감지하고 즉시 파드를 생성함.

#### 10. We just created a new static pod named static-greenbox. Find it and delete it.

> This question is a bit tricky. But if you use the knowledge you gained in the previous questions in this lab, you should be able to find the answer to it.   
> While trying to SSH to node01, if you are prompted for a password, use newRootP@ssw0rd.

static pod 삭제
1. static-greenbox이 배포된 node 찾기: kubectl get all --all-namespaces
2. 해당 node로 접속: ssh node01
3. 기본 static pod 매니페스트 경로로 이동했지만 yaml 파일 존재 안함: cd /etc/kubernetes/manifests
4. 위치를 따로 찾아봐야 함: grep -i staticPodPath /var/lib/kubelet/config.yaml
5. 특정된 경로로 이동: cd /etc/just-to-mess-with-you
6. 해당 static pod yaml 파일 삭제: rm -rf greenbox.yaml

</details>

<details>
<summary>Priority Classes</summary>

> 추가 예정

</details>

<details>
<summary>Multiple Schedulers</summary>

> 추가 예정

</details>

<details>
<summary>Admission Controllers</summary>

> 추가 예정

</details>

<details>
<summary>Validating and Mutating Admission Controllers</summary>

> 추가 예정

</details>

# 🪜 추가

<details>
<summary>k8s에서의 메모리/CPU 단위</summary>

### k8s에서의 메모리 단위

쿠버네티스에서는 두 가지 메모리 단위 체계를 사용합니다.

#### 1. 이진법 기반 단위 (2의 거듭제곱) - 권장
- `Ki` (Kibibyte) = 1024 bytes = 2¹⁰
- `Mi` (Mebibyte) = 1024 Ki = 1,048,576 bytes = 2²⁰
- `Gi` (Gibibyte) = 1024 Mi = 1,073,741,824 bytes = 2³⁰
- `Ti` (Tebibyte) = 1024 Gi = 2⁴⁰

#### 2. 십진법 기반 단위 (10의 거듭제곱)
- `K` (Kilobyte) = 1000 bytes = 10³
- `M` (Megabyte) = 1000 K = 1,000,000 bytes = 10⁶
- `G` (Gigabyte) = 1000 M = 1,000,000,000 bytes = 10⁹
- `T` (Terabyte) = 1000 G = 10¹²

#### 실제 비교
```yaml
memory: "256Mi"   # 268,435,456 bytes (약 268 MB)
memory: "256M"    # 256,000,000 bytes (정확히 256 MB)
```

#### 왜 이렇게 나뉘어 있나요?
- 컴퓨터는 이진법: 실제 메모리는 2의 거듭제곱으로 작동합니다 (1024 기반)
- 사람은 십진법: 우리는 1000 단위가 익숙합니다
- 쿠버네티스 권장: 이진법 단위(`Mi`, `Gi`) 사용을 권장합니다

### k8s에서의 CPU 단위

CPU는 더 간단합니다.
- `1` = 1 CPU 코어 (또는 1 vCPU)
- `0.5` = 0.5 CPU 코어
- `500m` = 500 밀리코어 = 0.5 CPU 코어 (`m`은 milli, 1000m = 1 CPU)

#### 예시
```yaml
cpu: "1"      # 1 코어
cpu: "500m"   # 0.5 코어 (같은 의미: "0.5")
cpu: "100m"   # 0.1 코어
cpu: "2"      # 2 코어
```

### 정리
| 단위 | 의미 | 예시 |
|------|------|------|
| `Mi` | 메비바이트 (1024² bytes) | `256Mi` = 약 268 MB |
| `Gi` | 기비바이트 (1024³ bytes) | `1Gi` = 약 1.07 GB |
| `m` | 밀리코어 (1/1000 CPU) | `500m` = 0.5 CPU |
| `숫자` | CPU 코어 수 | `1` = 1 CPU 코어 |

</details>

<details>
<summary>--dry-run=client</summary>

예시: kubectl run bee --image nginx --dry-run=client -o yaml > bee.yaml

(1) dry 의미: 실제로 리소스를 생성하지 않고 시뮬레이션만 수행
(2) run 의미: pod를 실행(생성)하는 명령어
(3) client: dry-run을 클라이언트 측에서만 수행 (서버에 요청을 보내지 않고 로컬에서만 검증)

=> 예시 해석: nginx 이미지를 사용하는 bee라는 이름의 pod를 실제로 생성하지 않고, 클라이언트 측에서만 yaml 형식으로 출력하여 bee.yaml 파일로 저장하는 명령어. 이렇게 생성된 yaml 파일을 편집한 후 `kubectl create -f bee.yaml`로 실제 pod를 생성할 수 있다. 

</details>

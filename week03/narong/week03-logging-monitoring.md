# 1. Metrics Server& cAdvisor

쿠버네티스를 운영하다 보면 "지금 내 노드와 파드들이 CPU랑 메모리를 얼마나 쓰고 있지?" 노드의 디스크가 꽉 차지는 않았는지 특정 파드가 메모리를 과도하게 점유하고 있지는 않은지 확인해야 합니다. 

이를 위해 **Metrics Server**가 있습니다.

## 1.1. 모니터링의 역사: Heapster → Metrics Server로

**Heapster는 더 이상 사용되지 않습니다(Deprecated).**

힙스터를 대체하기 위해 더 가볍고 슬림한 버전으로 만들어진 것이 바로 **Metrics Server**입니다. 현재 쿠버네티스 생태계의 표준 내장 모니터링 솔루션이라고 보시면 됩니다.

## 1.2. Metrics Server의 특징

Metrics Server는 클러스터 전체의 리소스 사용 데이터를 집계하는 역할을 합니다.

### 1.3 특징

1. **1 Cluster = 1 Metrics Server:** 클러스터당 하나만 존재합니다.
2. **In-Memory 저장:** 수집한 데이터를 디스크에 저장하지 않고 **메모리에만** 저장합니다.
    - 즉, 과거 데이터(History)를 볼 수 없습니다.
    - "지난주 금요일의 CPU 사용량" 같은 데이터를 보려면 Prometheus 같은 별도의 DB 기반 솔루션을 구축해야 합니다.
    - Metrics Server는 오직 "실시간 현재 상태"를 확인하는 용도입니다.

## 1.3. cAdvisor

그렇다면 Metrics Server는 각 노드의 CPU/Memory 정보를 어떻게 알아낼까요?

1. **Kubelet:** 모든 노드에는 Kubelet이 실행 중입니다.
2. **cAdvisor (Container Advisor):** Kubelet 내부에는 cAdvisor라는 하위 컴포넌트가 포함되어 있습니다.
    - 이 친구가 실제로 파드와 컨테이너의 성능 지표(CPU, 메모리 등)를 수집합니다.
3. **집계:** cAdvisor가 수집한 정보를 Kubelet API를 통해 노출하면 **Metrics Server**가 주기적으로 이 데이터를 긁어가서(Poling) 집계합니다.

## 1.4. 설치 및 사용 방법 (Getting Started)

Metrics Server는 쿠버네티스 설치 시 기본으로 포함되지 않는 경우가 많아 별도로 활성화해야 합니다.

### 설치

**1) Minikube 환경**
애드온 명령어를 사용

```
minikube addons enable metrics-server

```

**2) 일반 클러스터 (Bare metal, Cloud)**
공식 GitHub 리포지토리에서 배포니 파일을 다운로드하여 생성합니다.

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

```

## 1.5. 메트릭 확인하기 (Viewing Metrics)

설치가 완료되었다면 `kubectl top` 명령어로 손쉽게 리소스 사용량을 조회할 수 있습니다.

### 노드 리소스 확인

```
kubectl top nodes

```

**출력 예시:**

```
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube   166m         8%     1234Mi          15%

```

- **CPU(cores):** `166m`은 166 millicores(약 0.16 코어)를 의미합니다.

### 파드 리소스 확인

```
kubectl top pods

```

**출력 예시:**

```
NAME      CPU(cores)   MEMORY(bytes)
nginx     10m          20Mi
db-pod    50m          100Mi

```

# 2. kubectl logs

쿠버네티스 환경에서 파드 내부 애플리케이션의 로그를 확인하는 방법은 도커(Docker)와 매우 유사하면서도 파드의 특성상 주의해야 할 점이 하나 있습니다.

 `kubectl logs` 명령어의 기본 사용법과 멀티 컨테이너 환경에서의 로그 확인법을 정리해 봅니다.

## 2.1. 도커(Docker)에서의 로그 확인

쿠버네티스를 이해하기 위해 먼저 도커에서의 동작 방식을 되짚어 봅시다.

보통 컨테이너화된 애플리케이션은 로그를 별도 파일로 저장하기보다 표준 출력(Standard Output, stdout)으로 내보내도록 설계됩니다. 예를 들어, 웹 서버를 시뮬레이션하는 `event-simulator`라는 컨테이너가 있다고 가정해 봅시다.

### 백그라운드 실행 시 문제점

```
docker run -d event-simulator
```

- `d` (Detached mode) 옵션으로 백그라운드에서 실행하면 터미널에는 아무런 로그도 출력되지 않습니다.

### 로그 확인 명령어

이때 사용하는 명령어가 `docker logs`입니다.

```
# 저장된 로그 확인
docker logs <container_id>

# 실시간 로그 스트리밍 (Follow)
docker logs -f <container_id>
```

- `f` 옵션을 붙이면 마치 `tail -f`를 쓰는 것처럼 생성되는 로그를 실시간으로 따라가며 볼 수 있습니다.

## 2.2. 쿠버네티스에서의 로그 확인

쿠버네티스에서도 기본 원리는 같습니다. 파드 내부의 컨테이너가 표준 출력으로 로그를 뱉어내면, 우리는 `kubectl`을 통해 이를 조회합니다.

### 기본 명령어

```
# 파드의 로그 확인
kubectl logs <pod-name>

# 실시간 로그 스트리밍
kubectl logs -f <pod-name>
```

도커 명령어와 거의 똑같죠? `docker` 대신 `kubectl`을 쓰고 컨테이너 ID 대신 파드 이름을 쓴다는 점만 다릅니다.

## 2.3. 멀티 컨테이너 파드의 경우 (주의!)

여기서 한 가지 헷갈리기 쉬운 상황이 있습니다.
**"하나의 파드 안에 여러 개의 컨테이너가 들어있다면, 누구의 로그를 보여줄까?"**

예를 들어, `event-simulator` 파드 안에 다음 두 개의 컨테이너가 함께 실행 중이라고 가정해 봅시다.

1. `event-simulator` (이벤트 생성기)
2. `image-processor` (이미지 처리기)

이때 단순히 파드 이름만으로 로그를 요청하면 에러가 발생합니다.

```
kubectl logs event-simulator-pod
# Error: a container name must be specified ...

```

쿠버네티스 입장에서는 "두 녀석 중에 누구 로그를 보여달라는 거야?"라고 되묻는 것이죠.

### 해결 방법: 컨테이너 이름 명시하기 (-c 옵션)

파드 내에 컨테이너가 여러 개일 때는 반드시 **`-c` 옵션**으로 컨테이너 이름을 명시해야 합니다.

event-simulator 컨테이너의 로그만 확인
```kubectl logs event-simulator-pod -c event-simulator```

image-processor 컨테이너의 로그만 확인
```kubectl logs event-simulator-pod -c image-processor```
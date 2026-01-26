# 1. Rolling Updates & Rollbacks

애플리케이션을 운영하다 보면 필연적으로 **버전 업데이트**를 해야 할 때가 옵니다.
쿠버네티스에서 `Deployment`를 사용하면, 서비스 중단 없이 안전하게 애플리케이션을 업데이트하고 문제 발생 시 손쉽게 이전 버전으로 되돌릴 수 있습니다.

쿠버네티스의 배포 전략(Strategy)과 롤아웃(Rollout), 롤백(Rollback) 과정을 정리해 봅니다.

## 1.1. 롤아웃(Rollout)과 버전 관리

Deployment를 생성하거나 이미지를 업데이트하면 **Rollout** 프로세스가 시작됩니다.
쿠버네티스는 배포가 발생할 때마다 새로운 리비전(Revision)을 생성하고 변경 사항을 기록합니다. 이 덕분에 우리는 언제든지 이전 버전으로 되돌아갈 수 있습니다.

- **동작 원리:** Deployment는 업데이트가 발생할 때마다 **새로운 ReplicaSet**을 생성하여 파드들을 점진적으로 이동시킵니다.
- **확인 명령어:**
    
    ```
    kubectl rollout status deployment/my-app
    kubectl rollout history deployment/my-app
    ```
    

## 1.2. 배포 전략 (Deployment Strategies)

쿠버네티스는 크게 두 가지 배포 전략을 제공합니다.

### 1) Recreate (재생성)

> "다 부수고 새로 짓자!"
> 
- **동작:** 기존 버전의 파드들을 모두 한꺼번에 종료(Kill)시킨 후, 새로운 버전의 파드들을 생성합니다.
- **단점:** 기존 파드가 다 죽고 새 파드가 뜰 때까지 서비스가 중단되는 다운타임(Downtime)이 발생합니다.
- **사용처:** 잠깐의 중단이 허용되는 개발 환경이나, 구/신버전이 동시에 실행되면 안 되는 경우.

### 2) Rolling Update (롤링 업데이트) - **Default**

> "하나씩 교체하자!"
> 
- **동작:** 기존 파드를 하나 줄이고, 새 파드를 하나 늘리는 식으로 **점진적으로 교체**합니다.
- **장점:** 배포 중에도 애플리케이션이 계속 실행되므로 **서비스 중단(Downtime)이 없습니다.**
- **특징:** 쿠버네티스 Deployment의 **기본 전략**입니다. 별도 설정을 안 하면 이 방식으로 동작합니다.

## 1.3. 업데이트 방법 (Commands)

이미지를 업데이트하는 방법은 크게 두 가지가 있습니다.

### 방법 1: 정의 파일 수정 후 적용 (권장)

YAML 파일을 수정한 뒤 `apply` 하는 방법입니다.

```
# deployment.yaml 파일에서 image 버전을 수정한 후
kubectl apply -f deployment.yaml
```

### 방법 2: 명령어로 직접 수정 (Imperative)

YAML 파일을 건드리지 않고 즉시 이미지를 교체할 때 사용합니다.

```
# kubectl set image deployment <배포이름> <컨테이너이름>=<새이미지>
kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
```

- **주의:** 이 방식은 편리하지만 로컬의 YAML 파일(정의 파일)과 클러스터의 상태가 달라질 수 있으므로 주의해서 사용해야 합니다!

## 1.4. 롤백 (Rollback)

"야심 차게 새 버전을 배포했는데 버그가 터졌다!"
이때 당황하지 않고 이전 버전으로 되돌리는 기능이 **Rollback**입니다.

### 롤백 명령어

```
# 바로 직전 버전으로 되돌리기
kubectl rollout undo deployment/myapp-deployment
```

이 명령어를 실행하면, 쿠버네티스는 현재의 활성 ReplicaSet을 줄이고 이전 버전의 ReplicaSet을 다시 활성화하여 파드들을 복구합니다.

# 2. CMD vs ENTRYPOINT

쿠버네티스에서 파드(Pod) 정의 파일을 작성하다 보면 `command`와 `args`라는 필드를 볼 수 있습니다. 그 뿌리가 되는 **도커(Docker)의 명령어 처리 방식**을 먼저 이해해야 합니다.

이번 글에서는 도커 이미지 빌드 시 사용되는 `CMD`와 `ENTRYPOINT`의 차이점과 동작 원리를 정리해 봅니다.

## 2.1. 도커 컨테이너의 동작 원리

도커 컨테이너는 **"특정 프로세스(작업)를 실행하고 그 작업이 끝나면 종료되도록"** 설계되었습니다.

- **예시 1: `docker run ubuntu`**
    - 우분투 이미지의 기본 커맨드는 `bash`입니다.
    - `bash`는 터미널이 연결되지 않으면 할 일이 없어 즉시 종료됩니다. 따라서 컨테이너도 실행되자마자 죽습니다.
- **예시 2: `docker run ubuntu sleep 5`**
    - 명령어 뒤에 `sleep 5`를 붙여주면, 우분투는 5초 동안 잠을 자는 작업을 수행하고 그 후에 종료됩니다.
    - 이렇게 실행 시점에 명령어를 전달하여 동작을 제어할 수 있습니다.

## 2.2. Dockerfile: CMD vs ENTRYPOINT

나만의 커스텀 이미지를 만들 때, 컨테이너 시작 시 실행될 명령어를 지정하는 방법은 두 가지가 있습니다.

### 1) CMD (Command)

- **역할:** 컨테이너 시작 시 실행할 기본 명령어(Default Command)나 인자를 정의합니다.
- **특징:** 사용자가 `docker run` 실행 시 명령어를 입력하면 CMD에 정의된 내용은 완전히 무시(Override)됩니다.

**Dockerfile 예시:**

```
FROM ubuntu
CMD ["sleep", "5"]

```

**실행 결과:**

- `docker run my-image` 👉 `sleep 5` 실행
- `docker run my-image sleep 10` 👉 `sleep 10` 실행 (CMD 무시됨)

### 2) ENTRYPOINT (진입점)

- **역할:** 컨테이너가 시작될 때 항상 실행되어야 하는 실행 파일(Executable)을 고정합니다.
- **특징:** 사용자가 `docker run` 뒤에 입력한 값은 ENTRYPOINT 명령어의 파라미터로 추가(Append)됩니다.

**Dockerfile 예시:**

```
FROM ubuntu
ENTRYPOINT ["sleep"]
```

**실행 결과:**

- `docker run my-image 10` 👉 `sleep 10` 실행 (`10`이 `sleep` 뒤에 붙음)
- `docker run my-image` 👉 에러 발생 (`sleep` 명령어에 시간 인자가 없어서)

## 2.3. CMD와 ENTRYPOINT 함께 사용하기 (Best Practice)

가장 유연하고 좋은 방법은 두 가지를 섞어 쓰는 것입니다.
**ENTRYPOINT**로 실행 파일(명령어)을 고정하고 **CMD**로 기본 인자값을 설정합니다.

**Dockerfile 예시:**

```
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

**실행 결과:**

1. **인자 없이 실행:** `docker run my-image`
👉 `sleep 5` 실행 (CMD의 `5`가 ENTRYPOINT 뒤에 붙음)
2. **인자 넣고 실행:** `docker run my-image 10`
👉 `sleep 10` 실행 (사용자가 입력한 `10`이 CMD의 `5`를 덮어씀)

## 2.4. ENTRYPOINT 덮어쓰기

ENTRYPOINT로 설정된 실행 파일 자체를 변경해야 할 때는 어떻게 할까요?
실행 시 `--entrypoint` 옵션을 사용하면 됩니다.

```
# sleep 대신 echo 명령어 실행
docker run --entrypoint echo my-image "Hello Kubernetes"
```

이 개념은 쿠버네티스 파드(Pod) 정의 파일에서 다음과 같이 매핑됩니다.

- Docker `ENTRYPOINT` ➡️ Kubernetes `command`
- Docker `CMD` ➡️ Kubernetes `args`

이 관계를 이해하면 파드 실행 명령어를 훨씬 자유롭게 제어할 수 있습니다!

# 3. 파드에 환경 변수(Environment Variables) 주입하기

쿠버네티스 파드(Pod) 정의 파일에서 컨테이너에 환경 변수를 전달하는 다양한 방법을 알아봅시다.

## 3.1. 기본 방법: 직접 값 지정하기 (Plain Key-Value)

가장 간단하고 직관적인 방법은 파드 정의 파일의 `env` 속성에 키(name)와 값(value)을 직접 적어주는 것입니다.

### YAML 예시

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    ports:
    - containerPort: 8080

    # 여기서 환경 변수를 설정합니다.
    env:
    - name: APP_COLOR    # 변수 이름
      value: pink        # 변수 값
    - name: APP_MODE
      value: prod

```

이렇게 설정하고 파드를 생성하면컨테이너 내부의 애플리케이션은 `APP_COLOR`라는 환경 변수를 통해 `pink`라는 값을 읽어올 수 있게 됩니다.

## 3.2. 고급 방법: ConfigMap을 활용한 중앙 집중형 관리

환경 변수가 많아지거나 여러 파드에서 공통된 설정을 공유해야 한다면 파드 정의 파일에 일일이 값을 적는 것은 비효율적입니다. 이때 사용하는 것이 **ConfigMap**입니다.

### 1단계: ConfigMap 생성하기 (Creating ConfigMap)

ConfigMap을 만드는 방법은 크게 두 가지가 있습니다.

### A. 명령형(Imperative) 방식

`kubectl create configmap` 명령어를 사용하여 터미널에서 직접 생성합니다. 빠르고 간편합니다.

```
# 형식: kubectl create configmap <이름> --from-literal=<키>=<값>
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod

```

### B. 선언형(Declarative) 방식

YAML 파일을 작성하여 관리합니다. 버전 관리와 재사용이 용이하여 권장되는 방식입니다.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod

```

- **확인하기:** `kubectl get configmaps`, `kubectl describe configmap app-config`

### 2단계: 파드에 주입하기 (Injecting into Pod)

생성된 ConfigMap의 데이터를 파드의 환경 변수로 가져오는 방법도 두 가지가 있습니다.

### A. 특정 키만 가져오기 (`valueFrom`)

ConfigMap의 특정 데이터 하나만 콕 집어서 환경 변수로 매핑할 때 사용합니다.

```
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: app-config  # 참조할 ConfigMap 이름
          key: APP_COLOR    # ConfigMap 안의 키 이름

```

### B. 통째로 가져오기 (`envFrom`)

ConfigMap에 정의된 모든 키-값 쌍을 한 번에 환경 변수로 주입합니다. 변수가 많을 때 유용합니다.

```
    envFrom:
    - configMapRef:
        name: app-config

```

## 3.3. 보안 정보 관리: Secrets

DB 비밀번호나 API 키 같은 민감한 정보는 ConfigMap에 저장하면 평문으로 노출될 위험이 있습니다. 이런 정보는 **Secret** 오브젝트를 사용해야 합니다.
Secret은 데이터를 **Base64로 인코딩**하여 저장하며 암호화된 형태로 관리됩니다.

### 1단계: Secret 생성하기

### A. 명령형(Imperative) 방식

`kubectl create secret generic` 명령어를 사용합니다.

```
kubectl create secret generic app-secret --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123

```

### B. 선언형(Declarative) 방식 (Base64 인코딩 필수)

YAML 파일로 정의할 때는 반드시 값을 **Base64로 인코딩**해서 넣어야 합니다.

1. **인코딩 방법 (Linux/Mac):**
    
    ```
    echo -n 'password123' | base64
    # 출력결과: cGFzc3dvcmQxMjM=
    
    ```
    
2. **YAML 작성:**
    
    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: app-secret
    data:
      DB_Host: c3FsMDE=      # Base64 encoded 'sql01'
      DB_User: cm9vdA==      # Base64 encoded 'root'
      DB_Password: cGFzc3dvcmQxMjM= # Base64 encoded 'password123'
    
    ```
    
- **확인하기:** `kubectl get secrets`, `kubectl describe secret app-secret` (실제 값은 숨겨져서 보입니다)

### 2단계: 파드에 주입하기

### A. 환경 변수로 주입 (`valueFrom`)

ConfigMap과 사용법이 매우 유사하지만 `secretKeyRef`를 사용합니다.

```
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret  # 참조할 Secret 이름
          key: DB_Password  # Secret 안의 키 이름

```

### B. 볼륨으로 마운트 (Volumes)

환경 변수 외에도, Secret을 **파일 형태**로 파드 내부에 마운트할 수도 있습니다. 애플리케이션이 파일에서 설정을 읽어오는 방식일 때 유용합니다.

```
    volumes:
    - name: secret-volume
      secret:
        secretName: app-secret

```

### 3단계: 확인 및 보안 주의사항 (Verification & Best Practices)

### 확인하기

파드 내부에 Secret이 환경 변수로 잘 주입되었는지 확인하려면 `kubectl exec` 명령어를 사용하여 파드 내부의 환경 변수 목록을 조회해 볼 수 있습니다.

```
# 파드 내부의 모든 환경 변수 출력
kubectl exec -it <파드이름> -- env

# DB 관련 변수만 필터링
kubectl exec -it <파드이름> -- env | grep DB_

```

### 로깅 주의 (Logging Practices)

애플리케이션이 정상적으로 동작하는지 확인하는 것은 중요하지만 **민감한 정보가 로그에 남지 않도록 주의**해야 합니다.
예를 들어, 코드 내에서 `print(os.environ['DB_PASSWORD'])`와 같이 비밀번호를 출력하게 되면 `kubectl logs` 명령어를 통해 누구나 해당 비밀번호를 볼 수 있게 됩니다.

**보안 수칙:** 소스 코드에 비밀번호를 하드코딩하지 말고, 반드시 **Secret**을 사용하세요! 또한, 주입된 Secret 값을 애플리케이션 로그에 출력하지 않도록 주의해야 합니다!!!

# 4. Multi-Container Pods

쿠버네티스의 파드(Pod)는 필요에 따라 **하나의 파드 안에 여러 개의 컨테이너**를 함께 실행해야 할 때가 있습니다.
이들은 서로 **네트워크(IP)와 스토리지(Volume)를 공유**하며 밀접하게 협력합니다.

이번 글에서는 멀티 컨테이너 파드의 주요 디자인 패턴 3가지를 정리해 봅니다.

## 4.1. Co-located Containers (공존 컨테이너)

가장 기본적인 형태의 멀티 컨테이너 패턴입니다. 두 개 이상의 컨테이너가 파드의 전체 수명 주기 동안 함께 실행됩니다.

- **특징:**
    - 서로 강하게 결합(Tightly coupled)된 서비스들에 사용됩니다.
    - 특별한 시작 순서가 정해져 있지 않습니다. (거의 동시에 시작됨)
- **예시:**
    - 웹 서버 컨테이너 + 로그 수집 에이전트 컨테이너
    - 웹 서버가 로그를 파일에 쓰면 로그 수집기가 그 파일을 읽어서 외부로 전송하는 식입니다.

## 4.2. Init Containers (초기화 컨테이너)

메인 애플리케이션이 실행되기 전 필요한 사전 작업을 수행하는 컨테이너입니다.

- **동작 방식:**
    1. 파드가 생성되면 **가장 먼저** 실행됩니다.
    2. 주어진 작업을 완료하고 반드시 성공적으로 종료되어야 합니다.
    3. Init Container가 실패하면 파드는 재시작되며 성공할 때까지 메인 컨테이너는 실행되지 않습니다.
    4. 여러 개의 Init Container가 있다면 순차적(Sequential)으로 하나씩 실행됩니다.
- **사용 사례:**
    - **의존성 확인:** "데이터베이스(DB)가 뜰 때까지 기다려라" (예: `nc -z db-service 3306`)
    - **권한 설정:** 공유 볼륨의 파일 권한(`chown`, `chmod`) 변경
    - **초기 데이터 생성:** 설정 파일이나 초기 데이터를 동적으로 생성하여 볼륨에 저장

### YAML 예시

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:  # 초기화 컨테이너 정의
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]

```

## 4.3. Sidecar Containers (사이드카 컨테이너)

메인 애플리케이션 컨테이너의 기능을 보조하기 위해 옆에 붙여서 실행하는 컨테이너입니다. 오토바이 옆에 붙어 있는 보조석(Sidecar)에서 유래한 이름입니다.

- **특징:**
    - 메인 애플리케이션과 수명 주기를 함께합니다. (함께 시작하고 함께 종료)
    - 최근 쿠버네티스 버전(v1.28+)에서는 `InitContainer`에 `restartPolicy: Always`를 적용하여, 메인 앱보다 **먼저 시작되지만 종료되지 않고 계속 실행되는** 네이티브 사이드카(Native Sidecar) 패턴을 지원합니다.
- **사용 사례:**
    - **로그 수집기 (Log Shipping):** 메인 앱이 남기는 로그 파일을 실시간으로 읽어서(Tailing) 중앙 로그 서버(Elasticsearch 등)로 전송. (예: Filebeat, Fluentd)
    - **프록시 (Proxy):** 서비스 매시(Istio, Linkerd)에서 사용하는 사이드카 프록시. 네트워크 트래픽을 가로채서 보안, 모니터링 기능을 수행.

이 패턴들을 적절히 활용하면 하나의 거대한 컨테이너에 모든 기능을 때려 넣는 대신 기능별로 컨테이너를 분리하여 깔끔하고 유연한 아키텍처를 설계할 수 있습니다.


# 5. 자동 스케일링 (Automated Scaling)

트래픽이나 부하에 따라 쿠버네티스가 알아서 조절하는 방식입니다.

### 5.1. HPA (Horizontal Pod Autoscaler)

- **대상:** 워크로드 (파드 개수)
- **역할:** 파드의 CPU 사용량 등이 기준치를 넘으면 파드 개수(Replicas)를 자동으로 늘립니다.
- **사용:** 가장 보편적으로 사용되는 방식입니다.

### 5.2. VPA (Vertical Pod Autoscaler)

- **대상:** 워크로드 (파드 리소스)
- **역할:** 파드가 자꾸 죽거나 리소스가 부족해 보이면 파드의 `request/limit` 설정을 자동으로 수정하여 재시작합니다.

### 5.3. CA (Cluster Autoscaler)

- **대상:** 인프라 (노드 개수)
- **역할:** 파드가 너무 많아져서 더 이상 배치할 노드가 없을 때(Pending 상태), 클라우드 공급자(AWS, GCP 등)와 연동하여 자동으로 새 노드를 추가합니다.

## 5.4. Horizontal Pod Autoscaler

HPA는 쿠버네티스 워크로드 관리의 핵심 도구입니다.

### 1) 왜 HPA가 필요한가? (Manual vs Automated)

기존의 수동 방식(`kubectl scale`)은 관리자가 `kubectl top pod` 명령어로 리소스 사용량을 계속 감시해야 합니다. 트래픽이 급증하는 새벽 시간이나 주말에는 즉각적으로 대응하기 어렵고 비효율적입니다.
HPA는 설정된 임계값(Threshold)을 기반으로 파드의 개수를 자동으로 조절하여 리소스 할당을 최적화합니다.

### 2) 사전 조건

HPA가 동작하려면 파드의 리소스 사용량(CPU, Memory 등)을 알 수 있어야 합니다. 따라서 클러스터에 **Metrics Server**가 반드시 설치되어 있어야 데이터 수집이 가능합니다.

### 3) 설정 방법

### A. 명령형 (Imperative)

CLI 명령어로 빠르게 설정할 때 사용합니다.

```
# CPU 사용량이 50%를 넘으면 파드를 늘려라. (최소 1개 ~ 최대 10개)
kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10
```

### B. 선언형 (Declarative) - 권장

YAML 파일로 관리하는 것이 정석입니다. `autoscaling/v2` API 버전을 사용합니다.

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:        # 대상 지정 (어떤 디플로이먼트를 조절할 것인가?)
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1         # 최소 파드 수
  maxReplicas: 10        # 최대 파드 수
  metrics:               # 감시할 지표 설정
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50

```

### 4) 상태 확인

HPA가 잘 동작하고 있는지 확인하려면 다음 명령어를 사용합니다.

```
kubectl get hpa
```

출력 결과의 `TARGETS` 컬럼에서 `현재사용량 / 목표사용량` (예: `30%/50%`)을 확인할 수 있습니다.

## 6.5. 파드 무중단 리소스 변경: In-place Pod Resizing

**In-place Pod Vertical Scaling** 기능을 사용하면 **파드를 재시작하지 않고도 리소스를 변경**할 수 있습니다.

### 주요 특징

1. **Feature Flag:** `InPlacePodVerticalScaling` 기능을 활성화해야 사용 가능합니다.
2. **resizePolicy:** 리소스 변경 시 재시작 여부를 정책으로 설정할 수 있습니다.
    - `RestartNotRequired`: 재시작 없이 즉시 반영 (CPU 등)
    - `RestartRequired`: 재시작 필요 (메모리 등 일부 경우)
3. **제약 사항 (Limitations):**
    - **CPU, Memory**만 변경 가능합니다.
    - **Init Container**는 변경할 수 없습니다.
    - 메모리 제한(Limit)을 현재 사용량보다 낮게 줄일 수는 없습니다.
    - Windows 파드는 지원하지 않습니다.


## 5.6. Vertical Pod Autoscaler (VPA)

VPA는 파드의 리소스(CPU, Memory) 요구량을 자동으로 분석하고 최적화해 주는 도구입니다.

### 1) VPA의 3가지 핵심 컴포넌트

VPA는 3단계 프로세스를 거칩니다.

1. **Recommender (추천기):** 과거 리소스 사용 이력과 현재 사용량을 분석하여 "적절한 값"을 추천합니다.
2. **Updater (업데이터):** 현재 실행 중인 파드가 추천값과 너무 다르면 파드를 종료(Evict)시켜 재시작을 유도합니다.
3. **Admission Controller:** 파드가 생성(또는 재시작)될 때, 추천된 리소스 값을 파드 사양에 자동으로 주입합니다.

### 2) VPA 동작 모드 (Update Policies)

VPA가 얼마나 적극적으로 개입할지 결정할 수 있습니다.

- **Off:** 추천값만 계산하고, 실제 파드에는 아무 짓도 안 합니다. (모니터링 용도)
- **Initial:** 파드가 **처음 생성될 때만** 리소스를 수정합니다. 실행 중에는 건드리지 않습니다.
- **Recreate:** 실행 중인 파드의 리소스가 부적절하면 강제로 종료(Evict)하고 재시작하여 값을 수정합니다. (중단 발생)
- **Auto:** 현재는 `Recreate`와 동일하게 동작하지만 향후 **In-place Pod Resizing** 기능을 사용하여 **무중단 업데이트**를 수행하는 것이 목표입니다.

### 3) VPA 설정 예시 (YAML)

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"   # Off, Initial, Recreate, Auto

```

> 주의: 일반적으로 HPA와 VPA를 동일한 지표(CPU/Memory)로 동시에 설정하면 서로 충돌할 수 있으므로 권장하지 않습니다. (예: VPA는 CPU가 높다고 리소스를 늘리려 하고 HPA는 CPU가 높다고 파드를 늘리려 함)

> 
일반적으로 쿠버네티스 운영 환경에서는 HPA(파드 수평 확장)와 Cluster Autoscaler(노드 수평 확장)를 조합하여 트래픽 변화에 유연하게 대응하는 전략을 주로 사용하며, **VPA**는 리소스 사용량을 예측하기 어려운 애플리케이션의 적정 사이즈를 찾는 튜닝 도구로 활용합니다.
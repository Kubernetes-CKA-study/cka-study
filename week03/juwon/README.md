# **CKA 강의 정리: Section 4, 5, 6 - 로깅 & 모니터링, 애플리케이션 생명주기 관리, 클러스터 유지보수**

# Section4. Logging & Monitoring

## 1. 모니터링 개념

- **노드 단위 메트릭 예시**
    - 노드 개수, 건강한 노드 개수
    - CPU / 메모리 / 네트워크 / 디스크 이용률
- **Pod 단위 메트릭 예시**
    - Pod 개수
    - Pod CPU / 메모리 사용량
- 위와 같은 메트릭을 **수집 · 저장 · 분석**해 주는 모니터링 솔루션이 필요함
- 쿠버네티스 자체에 강력한 빌트인 모니터링은 없고, **외부/오픈소스 솔루션을 조합해서 사용**
    - 예: Metrics Server, Prometheus, Elastic Stack, Datadog, Dynatrace

## 2. Metrics Server

- 과거 **Heapster**가 담당하던 기능의 경량 버전이 **Metrics Server**
    - Heapster는 모니터링·분석 가능했지만 현재는 **deprecated**
- 클러스터당 **Metrics Server 1개**를 두고 사용
- **In-memory 모니터링 솔루션**
    - 디스크에 메트릭을 장기 저장하지 않음
    - 그래서 `kubectl top` 같은 **실시간/단기 지표** 확인에 적합
- 수집 구조
    - 각 노드의 **kubelet 안에 cAdvisor**가 내장되어 있음
    - cAdvisor가 노드/파드/컨테이너의 성능 지표를 수집
    - kubelet API를 통해 Metrics Server로 전달 → Kubernetes API로 노출 → `kubectl top` 등에서 사용

## 3. Metrics Server 관련 명령어

### 1) 시작하기

```bash
# Minikube addon으로 설치
minikube addons enable metrics-server

# Manifest로 설치 (예시)
git clone https://github.com/.../metrics-server.git
kubectl create -f metrics-server/deploy/1.8+/
```

### 2) CPU, 메모리 사용량 조회

```bash
# 노드별 리소스 사용량
kubectl top node

# 파드별 리소스 사용량
kubectl top pod
```

## 4. 로그 관리

### 1) 도커(Docker) 로그

```bash
# 컨테이너 실행 시 이벤트 로그 확인
docker run [컨테이너]

# 백그라운드(detached) 실행
docker run -d [컨테이너]
# → 바로 로그는 안 보이고, 따로 logs 명령으로 확인해야 함

# 컨테이너 로그 실시간 확인
docker logs -f [컨테이너ID 또는 이름]
```

### 2) 쿠버네티스(Kubernetes) 로그

```bash
# 파드 로그 실시간 확인
kubectl logs -f [pod명]

# 파드 내 특정 컨테이너 로그 확인 (멀티 컨테이너 파드)
kubectl logs -f [pod명] [컨테이너명]
```

- 하나의 Pod에 여러 컨테이너가 있을 때는 **컨테이너 이름까지 명시**해야 정확한 로그를 볼 수 있음

# Section5. Application Lifecycle Management

## 1. Rolling Updates & Rollbacks

### 1) Rollout 개념
- 새로운 Deployment 배포는 **새로운 롤아웃과 revision**을 생성함
- Deployment 수준의 변화를 추적해서, **이전 버전으로 롤백**할 수 있음

```bash
# 현재 롤아웃 상태 확인
kubectl rollout status deployment/<deployment-name>

# 롤아웃 히스토리 확인
kubectl rollout history deployment/<deployment-name>

# 이전 버전으로 롤백
kubectl rollout undo deployment/<deployment-name>
```

### 2) Deployment 전략

#### Recreate 전략
- 기존 인스턴스를 모두 삭제한 뒤, 새 인스턴스를 한꺼번에 생성
- **이전 버전이 내려가고, 새 버전이 모두 올라오기 전까지 서비스 단절** 발생

#### RollingUpdate 전략
- 인스턴스를 **하나씩(or 일부씩) 교체**
- 서비스가 계속 살아 있으므로 **다운타임 없이(seamless)** 배포 가능

### 3) 롤링 업데이트 수행 방법

```bash
# 1) YAML을 수정한 뒤 재적용
kubectl apply -f <deployment-file>.yaml

# 2) set image로 직접 이미지 버전 변경
kubectl set image deployment/<deployment-name> <container-name>=<image:tag>
```
- `kubectl set image`는 **Deployment 객체만 수정**하고, 로컬 YAML 파일은 자동으로 바뀌지 않으므로 주의

## 2. Commands & Arguments Configuration

### 1) Docker 관점
- 컨테이너는 **특정 태스크/프로세스**를 실행하고, 태스크가 끝나면 종료(exit)
- Dockerfile에서 기본 명령을 `CMD ["명령어"]` 형태로 정의
	- 예: `CMD ["bash"]` → 터미널 입력을 기다리는 쉘, 입력이 없으면 종료

```bash
# 이미지에 정의된 기본 CMD로 실행
docker run ubuntu

# 기본 CMD를 일시적으로 override
docker run ubuntu sleep 5
```

- Dockerfile 예시
```dockerfile
CMD ["sleep", "5"]          # 또는
CMD sleep 5

ENTRYPOINT ["sleep"]
# → docker run <image> 10  ==  sleep 10
```
- `--entrypoint` 옵션으로 ENTRYPOINT도 override 가능

### 2) Pod YAML에서 command/args 매핑

```yaml
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    command: ["sleep2.0"]   # Docker ENTRYPOINT에 해당
    args: ["10"]            # Docker CMD에 해당
```

`kubectl run`으로도 command/args 지정 가능:

```bash
# 인자만 넘기기
kubectl run <pod-name> --image=<image> -- <arg1> <arg2> ...

# command + args 지정
kubectl run <pod-name> --image=<image> --command -- <cmd> <arg1> <arg2> ...
```

### 3) YAML에서 리스트 작성 패턴

```yaml
# 한 줄 리스트
command: ["sleep", "5000"]

# 대시로 나열
command:
- "sleep"
- "5000"   # 숫자라도 문자열로 적는 게 안전

# command / args 분리
command: ["sleep"]
args: ["5000"]
```

## 3. Environment Variables & ConfigMaps

### 1) Docker에서 환경변수
```bash
docker run -e APP_COLOR=blue <image>
```

### 2) Pod YAML에서 env 설정

```yaml
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: APP_COLOR
      value: "blue"
```

### 3) ConfigMap 생성

**Imperative 방식 (명령형)**

```bash
# literal로 key=value 지정
kubectl create configmap <config-name> --from-literal=APP_COLOR=blue

# 파일로부터 생성
kubectl create configmap <config-name> --from-file=app-config.properties
```

**Declarative 방식 (정의 파일)**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config-name>
data:
  APP_COLOR: blue
```

```bash
kubectl create -f <config-file>.yaml

# 조회
kubectl get configmaps
kubectl describe configmap <config-name>
```

### 4) Pod에 ConfigMap 연결

#### (1) 여러 ENV 한 번에 주입 (envFrom)

```yaml
spec:
  containers:
  - name: app
    image: myapp
    envFrom:
    - configMapRef:
        name: <config-name>
```

#### (2) 개별 ENV 매핑

```yaml
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: <config-name>
          key: APP_COLOR
```

#### (3) Volume으로 마운트

```yaml
spec:
  volumes:
  - name: app-config-vol
    configMap:
      name: <config-name>
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: app-config-vol
      mountPath: /config
```

## 4. Secrets & Encryption at Rest

### 1) Secret 생성

**Imperative 방식**

```bash
kubectl create secret generic <secret-name> \
  --from-literal=DB_PASSWORD=passw0rd

kubectl create secret generic <secret-name> \
  --from-file=./secret-file.txt
```

**Declarative 방식**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
data:
  DB_PASSWORD: <base64-encoded-value>
```

```bash
# 인코딩 / 디코딩 예시
echo -n 'passw0rd' | base64
echo -n '<encoded-value>' | base64 --decode
```

```bash
kubectl get secrets
kubectl describe secret <secret-name>
```

### 2) Pod에 Secret 연결

#### (1) envFrom으로 전체 주입

```yaml
spec:
  containers:
  - name: app
    image: myapp
    envFrom:
    - secretRef:
        name: <secret-name>
```

#### (2) 개별 ENV 매핑

```yaml
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: <secret-name>
          key: DB_PASSWORD
```

#### (3) Volume으로 마운트

```yaml
spec:
  volumes:
  - name: app-secret-vol
    secret:
      secretName: <secret-name>
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: app-secret-vol
      mountPath: /etc/app-secret
```

> Secret의 각 key가 **파일 하나씩**으로 마운트되고, 파일 내용이 value가 됨

### 3) Encryption at Rest 

- 기본: Secret은 etcd에 **평문(단순 인코딩)** 으로 저장
- `EncryptionConfiguration`을 설정하면 etcd에 저장될 때 **암호화** 가능
- kube-apiserver에 `--encryption-provider-config` 옵션 + 해당 파일을 hostPath로 마운트해서 사용

## 5. Multi-Container Pods 개요

- 서로 **강하게 연관된 두 서비스**를 한 Pod 안에 함께 배치하는 패턴
  - 예: 웹 서버 + 로그 수집 에이전트
- 각 컨테이너는 **같은 네트워크 네임스페이스와 볼륨**을 공유
- 대표 패턴: **Sidecar, Adapter, Ambassador** (CKAD 범위)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: app
    image: myapp
    ports:
    - containerPort: 8080
  - name: log-agent
    image: log-collector
    volumeMounts:
    - name: logs
      mountPath: /log
  volumes:
  - name: logs
    emptyDir: {}
```

```bash
# 컨테이너 안에서 직접 파일 확인
kubectl -n <namespace> exec -it <pod> -c log-agent -- cat /log/app.log

# Pod 로그 바로 보기
kubectl logs <pod> -n <namespace>
```

# Section6. Cluster Maintenance

<details>
<summary><strong>1. 운영체제 업그레이드</strong></summary>

### 1) Node 장애와 Pod 처리

- 노드가 다운되면, 해당 노드의 Pod에 접근 불가
- 노드가 온라인으로 다시 돌아온다면 kubelet 프로세스가 실행됨
- **pod-eviction-timeout** 동안 돌아오지 않는다면 Pod가 중지되고, ReplicaSet에 속한 Pod라면 다른 노드에 생성됨
- **pod-eviction-timeout**이 5분으로 기본 설정되어 있음 (`kube-controller-manager`)
- pod-eviction-timeout이 지나고 노드가 온라인으로 돌아온다면 빈 노드로 생성됨
- ReplicaSet에 속하지 않고 해당 노드에만 속한 Pod가 있었다면 문제가 됨

### 2) 유지보수 시 필수 명령어

- `kubectl drain [노드명]`
    - 노드의 Pod들이 중지되고 다른 노드에 생성됨
    - 노드는 unschedulable로 마킹됨 (어떠한 Pod도 스케줄링될 수 없음)
    - `kubectl drain [노드명] --ignore-daemonsets` (DaemonSet Pod 무시)
- `kubectl cordon [노드명]`
    - 노드를 unschedulable로 마킹
    - 기존 Pod는 영향을 받지 않음
- `kubectl uncordon [노드명]`
    - 노드에 Pod들이 스케줄링될 수 있도록 마킹 변경
    - 노드를 reboot한 다음에 사용할 수 있는 명령어
    - 다른 노드로 보낸 Pod가 해당 노드로 돌아오지는 않음
    - 새로운 Pod가 스케줄링될 때 해당 노드에 스케줄링될 수 있음

> `drain`은 Pod를 다른 노드로 "이동"하는 것이 아니라, **Pod를 삭제**하고 ReplicaSet 등의 컨트롤러가 **다른 노드에 새 Pod를 생성**하게 만드는 동작입니다.

</details>

<details>
<summary><strong>2. 쿠버네티스 소프트웨어 버전</strong></summary>

### 1) 버전 형식

- **major.minor.patch** 순서 (예: `v1.11.3`)
- **minor**
    - 새로운 특징(feature)과 기능(functionality) 포함
    - 몇 달에 한번 주기
- **patch**
    - 버그 수정
    - minor보다 빈번
- **alpha, beta 버전**
    - 안정적인 버전을 출시(release)하기 이전

### 2) 버전 관리

- 같은 버전으로 관리될 수 있는 프로젝트
    - 예: `kube-apiserver`, `controller-manager`, `kube-scheduler`, `kubelet`, `kube-proxy`, `kubectl`
- ETCD cluster, CoreDNS는 다른 프로젝트이기 때문에 다른 버전으로 관리됨

</details>

<details>
<summary><strong>3. 클러스터 업그레이드</strong></summary>

### 1) 버전 정책

- 쿠버네티스는 **가장 최신의 3개 버전만 지원**함
- 요소(component)들이 모두 반드시 같은 버전으로만 사용되어야 하는 것은 아님
- **kube-apiserver가 X 버전** (예: v1.10)이라면
    - `controller-manager`, `kube-scheduler`는 **X-1 버전** (예: v1.9, v1.10)까지
    - `kubelet`, `kube-proxy`는 **X-2 버전** (예: v1.8, v1.9, v1.10)까지 가능
    - kube-apiserver보다 높은 버전을 가질 수는 없음
- 이러한 특성 덕분에 **live upgrade**가 가능
- 요소(component)별로 업그레이드 가능
- 여러 버전을 한 번에 건너뛰어서 업그레이드하기 보다는, **한 버전씩 순차적으로 업그레이드**하는 것이 권장됨

</details>

<details>
<summary><strong>4. 업그레이드 방법</strong></summary>

### 1) 구글 쿠버네티스 엔진 사용 시
- 버튼으로 업그레이드 가능

### 2) kubeadm 사용해 클러스터 구성 시
- `kubeadm upgrade plan`
- `kubeadm upgrade apply`

### 3) 처음부터 클러스터 구성 시 (from scratch)
- 구성한 클러스터에 맞게 업그레이드

</details>

<details>
<summary><strong>5. 업그레이드 순서 (kubeadm)</strong></summary>

### 1) 마스터 노드 업그레이드

- controlplane 요소들 (예: API server, controller-manager, scheduler)이 잠시 다운됨
- 워커 노드의 워크로드들은 평소처럼 서비스됨
- 관리 기능들만 영향을 받음
    - 예: `kubectl` 혹은 쿠버네티스 API로 클러스터 접근 불가
    - 예: 새로운 프로그램을 배포하거나 삭제, 수정 불가
    - 예: Pod 중단 시 새로운 Pod 생성 불가 (스케줄링 불가)

### 2) 워커 노드 업그레이드

#### 2-1. 워커 노드 전체를 다운시키고 업그레이드
- 서비스가 중단되는 다운타임 필요

#### 2-2. 워커 노드를 하나씩 업그레이드
- 하나의 노드가 업그레이드되는 동안 해당 노드에 있던 Pod는 다른 노드로 이동

#### 2-3. 새로운 소프트웨어 버전을 가진 노드를 새로 추가
- 워크로드를 새로운 노드에 옮기고 기존 노드를 다운

</details>

<details>
<summary><strong>6. 명령어 (kubeadm)</strong></summary>

### 1) 업그레이드 계획 확인

```bash
kubeadm upgrade plan
```
- 현재 클러스터 버전, 최신 버전 등의 정보 제공

### 2) 제어 플레인(마스터 노드) 업그레이드

**[마스터 노드에서 실행]**

```bash
# 1. kubeadm 패키지 업그레이드
apt-get upgrade -y kubeadm=1.12.0-00
# 또는 apt update 후
apt-get install kubeadm=1.12.0-00

# 2. 노드 내 클러스터 요소들을 업그레이드
kubeadm upgrade apply v1.12.0

# 3. kubelet, kubectl 패키지 업그레이드
apt-get upgrade -y kubelet=1.12.0-00
# 또는
apt-get install kubelet=1.12.0-00

# 4. kubelet 재시작 (필요시 daemon-reload)
systemctl restart kubelet
systemctl daemon-reload  # 필요할 수 있음

# 5. 버전 확인 (kubectl get nodes는 각 노드의 kubelet 버전 정보를 의미)
kubectl get nodes
```

### 3) 워커 노드 업그레이드 (각 노드에 대해 해당 순서 진행)

**[마스터 노드에서 실행]**

```bash
# 1. 노드의 Pod를 다른 노드로 옮기고 unschedulable 처리
kubectl drain [노드명] --ignore-daemonsets --delete-emptydir-data
```

**[해당 워커 노드에 SSH 접속 후 실행]**

```bash
# 2. kubeadm 패키지 업그레이드
apt-get upgrade -y kubeadm=1.12.0-00
# 또는
apt-get install kubeadm=1.12.0-00

# 3. kubelet 패키지 업그레이드
apt-get upgrade -y kubelet=1.12.0-00
# dependency 이슈 발생 시
apt-get install kubelet=1.12.0-00

# 4. 노드 설정 업그레이드
kubeadm upgrade node config --kubelet-version v1.12.0
# 또는
kubeadm upgrade node

# 5. kubelet 재시작
systemctl restart kubelet
```

**[마스터 노드에서 실행]**

```bash
# 6. 노드를 다시 schedulable로 변경
kubectl uncordon [노드명]
```

</details>

<details>
<summary><strong>7. 백업 및 복구</strong></summary>

### 1) 리소스 환경설정 (Resource Configuration)

#### 리소스 생성 방법

**Imperative 방식 (명령형)**
- 명령어를 통해 생성
- 예: `kubectl create namespace [네임스페이스]`
- 예: `kubectl create secret`
- 예: `kubectl create configmap`

**Declarative 방식 (선언형)**
- Pod YAML 파일을 통해 생성
- Imperative에 비해서 백업에 있어서 더욱 바람직 (GitHub 등에 업로드 가능)
- `kubectl apply -f [파일명].yaml`

#### 리소스 백업

**kube-apiserver 활용**
- 모든 객체에 대한 복사본을 만들어 백업 가능
```bash
kubectl get all --all-namespaces -o yaml > [파일명].yaml
```

**외부 솔루션 활용**
- Velero (HeptIO)

### 2) ETCD Cluster

#### 들어가며

**etcd 명령어를 사용할 때 필요한 항목들**

- etcd 클러스터의 엔드포인트
    - `--endpoints=[ETCD pod 내 listen-client-urls 혹은 advertise-client-urls]`
- 인증을 위한 CA 인증서
    - `--cacert=[ETCD pod 내 trusted-ca-file]`
- etcd 서버의 인증서
    - `--cert=[ETCD pod 내 cert-file]`
- etcd 서버의 key
    - `--key=[ETCD pod 내 key-file]`

**ETCDCTL_API 환경변수 설정**
```bash
export ETCDCTL_API=3
```
- etcdctl 사용 전 미리 환경변수로 설정할 수 있음

#### ETCD 백업

- ETCD 서버 자체를 백업할 수 있음
- 빌트인 스냅샷 솔루션을 활용해 백업 가능

```bash
# 스냅샷 저장
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-client.crt \
  --key=/etc/etcd/etcd-client.key

# 스냅샷 상태 확인
ETCDCTL_API=3 etcdctl snapshot status snapshot.db
```

- 위에서 언급한 etcd 명령어를 사용할 때 필요한 항목들 (예: endpoint)을 같이 기재해줘야 함
- 마스터 노드에 ETCD 위치
    - `etcd.service` 파일에서 `--data-dir=` 옵션을 통해 경로 확인 가능

#### ETCD 복원 (Restore)

**Case 1. ETCD가 static pod로 배포된 경우 (Stacked ETCD)**

1. `service kube-apiserver stop`
    - ETCD에 접속하는 API 서버를 먼저 내림
2. 스냅샷 복원 실행
    ```bash
    ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir [새로운 데이터 디렉토리 경로]
    ```
    - 새로운 클러스터 환경설정을 사용, etcd의 멤버로 하여금 새로운 클러스터의 새로운 멤버가 되도록 함
    - 우연히 기존에 존재하는 클러스터에 합류되는 일을 막기 위함
    - restore 시에는 endpoint 등을 추가로 기재해줄 필요 없음
3. `/etc/kubernetes/manifests/etcd.yaml`에서 `volumes: hostPath: path`를 새로운 디렉토리로 수정
    - `volumes: hostPath`는 controlplane 내의 위치
    - `volumeMounts: mountPath`는 컨테이너 내의 위치
    - volumeMounts와 `--data-dir`의 경로는 같아야 하지만, volumes과 volumeMounts는 이름으로 연결되어 있기 때문에 경로가 같을 필요는 없음
4. etcd pod가 자동으로 재시작 (kube-controller-manager, kube-scheduler도 마찬가지)
    - 만약 자동으로 재시작되지 않는다면: `kubectl delete pod -n kube-system [etcd pod명]`
5. `service kube-apiserver start`

**Case 2. ETCD가 외부 서버에서 배포된 경우 (External ETCD)**

1. `service kube-apiserver stop`
2. 스냅샷 복원 실행
    ```bash
    ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir [새로운 데이터 디렉토리 경로]
    ```
3. 외부 ETCD 서버에 SSH 접속
    ```bash
    ssh etcd-server
    ```
4. 복구 시 새로운 데이터 디렉토리의 권한 설정
    ```bash
    chown -R etcd:etcd [데이터 디렉토리명/]
    ```
    - 복구 시 새로운 데이터 디렉토리의 권한이 root:root으로 설정되어 있을 경우 사용
    - 기존 데이터 디렉토리의 권한과 동일하게 설정
5. ETCD 서비스 파일 수정
    ```bash
    vi /var/lib/etc/systemd/system/etcd.service
    # --data-dir 경로를 새로운 폴더로 변경
    ```
6. ETCD 서비스 재시작
    ```bash
    systemctl daemon-reload && systemctl restart etcd
    exit  # 기존 노드로 돌아옴 (ssh 연결 해제)
    ```
7. controlplane 요소들 재시작
    ```bash
    kubectl delete pods [controller명] [scheduler명] -n kube-system
    ```
8. kubelet 재시작
    ```bash
    ssh [controlplane 노드명] && systemctl restart kubelet
    ```
9. `service kube-apiserver start`

</details>

<details>
<summary><strong>8. 참고</strong></summary>

### 1) 기타 명령어

```bash
# 명령어 사용 시 편한 가명(alias) 지정 가능
alias k=kubectl

# 모든 노드들에 대해 taint 옵션이 있는지 확인 가능
k describe node | grep Taints

# 해당 노드로 접근 가능 (ssh 접속)
ssh [노드명/IP]

# 파일을 앞의 경로에서 뒤의 경로로 복사
scp [보낼 파일의 경로] [받을 경로]

# 클러스터를 포함한 다양한 정보 조회
kubectl config view

# 클러스터 전환
kubectl config use-context [클러스터명]

# ETCD 클러스터 내에 몇 개의 노드가 있는지 확인
ETCDCTL_API=3 etcdctl member list \
  --endpoints=[URL] \
  --cacert=[파일경로] \
  --cert=[파일경로] \
  --key=[파일경로]
# ETCDCTL_API=3의 경우 환경변수 미리 설정해주었다면 생략 가능
```

### 2) Stacked ETCD와 External ETCD

#### Stacked ETCD

- controlplane에 pod 형태로 배포
- `kubectl get pods -n kube-system`을 통해 조회 가능
- 확인 방법:
    - `kubectl describe pod [kube-apiserver pod명]`에서 `--etcd-servers=[경로]`에서 localhost IP 사용 여부로 확인 가능
    - `/etc/kubernetes/manifests/`에 yaml 파일 있는지 여부로 확인 가능 (static pod)

#### External ETCD

- pod 형태로 배포되지 않음
- 확인 방법:
    - `kubectl describe pod [kube-apiserver pod명]`에서 `--etcd-servers=[경로]`에서 외부(별도) IP 사용 여부로 확인 가능

#### ETCD Configuration 조회

```bash
# 프로세스에서 ETCD 정보 (예: endpoints) 조회
ps -ef | grep -i etcd
```

</details>

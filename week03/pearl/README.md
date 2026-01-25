# Logging & Monitoring
- 노드, 파드, 컨테이너의 상태 로그를 찍고 모니터링하여 문제가 있는지 찾기 위해서 알아두어야 한다.

## Monitoring Components
- 쿠버네티스에서는 모든 데이터를 모니터링할 수 있는 내장 모니터링 솔루션이 제공되지 않는다.
- 따라서 Metrics-Server, Prometheus, Elastic Stack 등의 오픈소스 솔루션이나, 독점 라이센스를 가진 Datadog 혹은 Dynatrace 같은 솔루션을 사용할 수 있다.

## Metrics-Server
- 쿠버네티스 클러스터 당 하나의 메트릭 서버를 가질 수 있다.
- 메트릭 서버는 각 노드와 파드로부터 모니터링 데이터를 검색한 뒤, 이를 종합하여 메모리에 저장한다.
- 메트릭 서버는 인메모리 모니터링 솔루션이기 때문에 측정 항목을 디스크에 저장하지 않는다.
- Lab Session 에서는 표준으로 공식 매니페스트를 적용한다.


```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl get pods -n kube-system | grep metrics # 메트릭 서버 확인

kubectl get top node # 노드 관련 메트릭 조회
```

## Application Logs
- 도커 컨테이너의 로깅처럼 생각하면 된다.
    - `kubectl logs -f event-simulator.yaml`: 특정 파드에 대한 로그 확인하는 명령어
- 다중 컨테이너에서 로그를 확인하려면, 파드 중 어떤 컨테이너의 로그를 확인할 것인지 한 번 더 명시해줘야 한다.
    - `kubectl logs -f event-simulator-pod event-simulator`: <특정 파드> <특정 컨테이너> 순으로 명시


---
# Application Lifecycle Management

## Rolling Updates and Rollbacks
- 디플로이먼트가 처음 생성되면, 롤아웃이 트리거된다.
- 새로운 롤아웃은 새로운 디플로이먼트의 리비전을 생성한다.
- 리비전은 디플로이먼트가 변경사항을 추적하고자 할 때와 이전 버전으로 롤백하고자 할 때 도움을 준다.
- 두 개의 리비전이 유지되고 있기 때문에 새로운 버전으로 롤아웃/이전 버전으로 롤백하는 것이 가능하다


## 배포 전략

### 1. Recreate

- 기존 배포된 애플리케이션을 모두 삭제하고, 새 버전의 애플리케이션을 다시 생성
- 삭제 후 새 버전을 생성하는 사이에 다운타임이 발생함
- 쿠버네티스의 기본 배포 전략 아님


### 2. Rolling Update
- 이전 버전을 하나씩 종료하고, 새로운 버전을 배포하는 방식
- 어플리케이션 다운타임이 발생하지 않고 원활하게 새 버전을 배포 가능
- 쿠버네티스의 기본 배포 전략이다.

## Updates
- 매니페스트 파일의 변경 이후, kubectl apply 명령어를 통해 업데이트를 적용할 수 있다. 
- 이를 통해 디플로이먼트의 새로운 리비전이 생성된다.
- `kubectl set image` 명령어를 통해 애플리케이션 이미지 업데이트를 진행할 수 있다. 
- 이 방식은 yaml 파일 내 애플리케이션 이미지 버전을 변경하지는 않는다.
- `kubectl set image deployment frontend simple-webapp=kodecloud/webapp-color:v2`


## Commands
관련 명령어들을 알아보자.


```bash
kubectl rollout status deployment/myapp-deploy # 롤아웃 상태 확인
kubectl rollout history deployment/myapp-deploy # 롤아웃 기록 확인
kubectl rollout undo deployment/myapp-deploy # 이전 버전으로 롤백
```



## Commands and Arguments
Dockerfile의 ENTRYPOINT는 쿠버네티스의 command 필드와 대응되며, CMD는 쿠버네티스의 args 필드와 대응된다.
```bash
FROM Ubuntu

ENTRYPOINT [ "sleep" ]

CMD [ "5" ]
```


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu-sleeper
      command: [ "sleep2.0" ]
      args: [ "10" ]
```


## Configure Environment Variables in Applications
- 쿠버네티스에서 환경변수를 설정하는 방법 (기본)
```yaml
env:
  - name: APP_COLOR
    value: pink

```


- 컨피그맵을 통해 환경변수를 설정하는 방법
```yaml
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
```


- 시크릿을 통해 환경변수를 설정하는 방법
```yaml
env:
  - name: APP_COLOR
    valueFrom:
      secretKeyRef:
```

### ConfigMaps 사용
- 환경변수를 파드 매니페스트 파일에 정의할 수도 있으나, 이렇게 하면 환경변수를 각 파드별로 관리하게 된다.
- configmap을 사용하면 하나의 파일에서 모든 환경변수를 관리하여 더 효율적이다.


### Create ConfigMaps
`--from-literal` 옵션을 사용하여 명령형 방식으로 ConfigMap을 생성할 수 있다.
```bash
kubectl create configmap \
  app-config --from-literal=APP_COLOR=blue \
             --from-literal=APP_MODE=prod
```


또는 `--from-file` 옵션을 통해 해당 파일을 읽어 ConfigMap을 생성할 수 있다.
```bash
kubectl create configmap \
  app-config --from-file=app_config.properties
```


아래와 같이 선언형으로도 ConfigMap을 생성할 수 있다.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: "blue"
  APP_MODE: "prod"
```

### View ConfigMaps
아래 명령어를 사용해서 ConfigMap의 조회, 정보를 확인할 수 있다.
```bash
kubectl get configmaps # 조회
kubectl describe configmaps # 정보 확인
```


### ConfigMap in Pods
아래와 같이 `envFrom.configMapRef` 필드에 생성된 ConfigMap 이름을 지정할 수 있다.
```yaml
envFrom:
  - configMapRef:
    name: app-config
```


단일 환경변수로 주입하려면 `env.valueFrom.configMapKeyRef` 필드에 생성된 컨피그맵 이름을 지정하고, 환경변수 이름을 key로 지정한다.

```yaml
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_COLOR

# 또는 볼륨에 마운트하는 것도 가능하다.
volumes:
  - name: app-config-volume
    configMap:
      name: app-config
```


### Configure Secrets in Applications
- ConfigMap은 문자열을 그대로 저장하기 때문에 민감한 정보가 있을 때는 위험하다.
- 시크릿을 사용하면 해당 정보가 인코딩 or 해시 상태로 저장되어 보안을 강화할 수 있다.

### Create Secrets
`--from-literal` 옵션을 사용하여 명령형 방식으로 시크릿을 생성할 수 있다.
```bash
kubectl create secret generic \
  app-secret --from-literal=DB_HOST=mysql \
             --from-literal=DB_USER=root \
             --from-literal=DB_PASSWORD=paswrd
```


또는 `--from-file` 옵션을 통해 해당 파일을 읽어 시크릿을 생성할 수 있다.
```bash
kubectl create secret generic \
  app-config --from-file=app_secret.properties
```


아래와 같이 선언형으로도 시크릿을 생성할 수 있다. 이 때, 인코딩된 형식으로 데이터를 지정해야 한다. (e.g. base64 인코딩 등)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: "bXlzcWw="
  DB_USER: "cm9vda=="
  DB_PASSWORD: "cGFzd3Jk"
```



### View Secrets
시크릿 관련하여 다양한 것을 확인하는 커맨드 목록

```bash
kubectl get secrets # 생성된 시크릿 조회
kubectl describe secrets # 생성된 시크릿 정보 조회
kubectl get secret app-secret -o yaml # 시크릿 값을 조회
echo -n 'bXlzcWw=' | base64 --decode # base64 인코딩된 데이터를 디코딩하는 예시 커맨드
```


### Secrets in Pods
아래와 같이 envFrom.secretRef 필드에 생성된 시크릿 이름을 지정할 수 있다.
```yaml
envFrom:
  - secretRef:
    name: app-secret
```


단일 환경변수로 주입하려면 `env.valueFrom.secretKeyRef` 필드에 생성된 시크릿 이름을 지정하고, 환경변수 이름을 key로 지정한다.

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_PASSWORD
# 또는 볼륨에 시크릿을 마운트할 수도 있음
volumes:
  - name: app-secret-volume
    secret:
      secretName: app-secret
```


## Multi Container Pods
- 마이크로서비스 아키텍쳐가 등장하면서 각 서비스를 스케일 업/다운하는 작업이 필요하다.
- 하지만 마이크로서비스임에도 불구하고, 같이 써야 효과적인 서비스들도 있다.(e.g. 애플리케이션과 로깅 서비스)



### Multi Container Pods
- 멀티 컨테이너 파드는 같은 네트워크를 사용하여 서로 로컬호스트처럼 접근이 가능하다.
- 또한, 동일한 스토리지 볼륨을 공유한다.


### Create
`spec.containers` 필드에 여러 컨테이너를 명시하면 된다.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers: # 컨테이너 밑에 웹애플리케이션과 로깅 서비스를 같이 올리기
    - name: simple-webapp
      image: simple-webapp
      ports:
        - containerPort: 8080
    - name: log-agent
      image: log-agent
```


### Init Containers
- 파드 컨테이너가 실행될 때, 메인 프로세스 컨테이너 실행 전에 선행 작업이 필요한 경우가 있다.
- e.g. 원격 저장소로부터 소스코드나 바이너리 파일을 받아 메인 애플리케이션에 사용하고자 하는 경우
- 이럴 때는 init Container를 사용한다.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
    - name: myapp-container
      image: busybox:1.28
      command: [ 'sh', '-c', 'echo The app is running! && sleep 3600' ]
  initContainers: # git clone하여 초기화하고 시작
    - name: init-myservice
      image: busybox
      command: [ 'sh', '-c', 'git clone <some-repository-that-will-be-used-by-application>; done;' ]
```



- 이렇게 yaml을 설정하면 파드가 실행될 때 초기화 컨테이너가 먼저 실행된다.
- 애플리케이션 컨테이너가 실행되기 전에 초기화 컨테이너의 실행이 완료되어야 한다.
- 초기화 컨테이너는 여러 개 정의할 수 있고, 순차적으로 실행된다.
- 초기화 컨테이너가 실패하면 성공할 때까지 파드를 재실행한다.



---
# Cluster Maintainance
쿠버네티스에서 클러스터를 운영할 때, 여러 작업이나 업데이트가 일어나도 클러스터를 안정적으로 유지하는 것이 중요하다.
이런 관점에서, 여러 상황에서 클러스터 유지가 필요한 경우를 다룬다.


## OS Upgrades

### Pod Eviction Timeout

- 파드가 다운되었을 때, Kubelet은 파드를 다시 활성 상태로 만든다.
- 하지만 파드가 5분 이상 다운 상태가 되면 쿠버네티스는 그 파드가 죽은 것으로 간주하여 해당 파드는 노드로부터 종료된다.
- 만약 파드가 레플리카셋의 일부인 경우에는 다른 노드에서 재생성된다.
- 파드가 다시 온라인 상태가 될 때까지 기다리는 시간을 파드 제거 시간 초과라고 하며, 아래 명령어를 통해 설정할 수 있다.
  ```bash
  kube-controller-manager --pod-eviction-timeout=5m0s
  ```
- 기본 설정은 5분으로, 마스터 노드는 파드가 죽은 것으로 간주하기 전에 최대 5분동안 기다리는 것이다.
- 파드 제거 시간 초과 이후 노드가 다시 활성화되면 예약된 파드 없이 빈 노드 상태로 활성화된다.


### Drain

- 안전하게 노드를 작업하기 위해, 노드를 비워(drain) 모든 워크로드를 다른 노드로 옮길 수 있다. 이때 데몬셋을 제외하고 적용하려면 `--ignore-daemonsets` 옵션을 사용한다.
  ```bash
  kubectl drain node-1
  ```
- 기존 노드에서 파드가 종료되고, 다른 노드에서 재생성된다.
- 노드가 통제(cordon)되거나 스케줄 불가능 상태가 된다. 이는 특별한 제한이 제거될 때까지 해당 노드에 스케줄된 파드가 없음을 의미한다.
- 파드가 다른 노드에서 정상적으로 구동되면, 노드를 재시동할 수 있다. 해당 노드가 다시 활성화되어도 여전히 스케줄 불가능 상태이다.
- 노드의 통제를 해제해야만 다시 파드가 스케줄된다.
  ```bash
  kubectl uncordon node-1
  ```
- 노드를 스케줄 불가능하게 하지만 노드를 비우는 것(drain)과 다르게 파드를 종료/이동시키지 않고 싶다면 통제(cordon)를 적용한다. 이는 새로운 파드가 더 이상 스케줄링되지 않도록 처리한다.
  ```bash
  kubectl cordon node-1
  ```

## Cluster Upgrade Process

- 쿠버네티스의 구성 요소들의 버전이 모두 같을 필요는 없다.
- 하지만 kube-apiserver는 컨트롤 플레인의 핵심 구성 요소이기 때문에 다른 구성 요소들이 kube-apiserver보다 버전이 높으면 안된다.
  - kube-controller-manager, kube-scheduler는 한 버전 더 낮을 수 있다.
  - kubelet, kube-proxy는 두 버전까지 낮을 수 있다.
  - kubectl은 1 버전 높거나, 같거나, 1 버전 낮을 수 있다.
- 쿠버네티스는 오직 최근 세 개의 버전까지만 지원한다.
- 추천되는 버전 업그레이드에 대한 접근은 마이너 버전을 하나씩 올리는 것이다.
- 버전 업그레이드는 클러스터를 어떤 방식으로 구성했는지에 따라 다르다.
  - 클라우드 제공자: 제공되는 업그레이드 기능을 사용한다.
  - kubeadm: kubeadm을 통해 업그레이드한다.
  - scratch: 수동으로 업그레이드한다.

### 어떻게 업그레이드할까?

**마스터 노드 업그레이드**

- 마스터 노드가 업그레이드할 동안 kube-apiserver, kube-controller-manager, kube-scheduler 등의 컨트롤 플레인 구성 요소들이 다운된다.
- 마스터 노드가 다운된다고 해서 워커 노드가 영향을 받는 것은 아니다.
- kube-apiserver가 다운되어 있기 때문에, 이와 관련된 리소스에는 접근할 수 없으며, 새로운 애플리케이션을 배포하거나 삭제하거나 수정할 수 없다.
- 만약 파드가 죽으면, 새로운 파드가 자동으로 재생성되지 않는다.
- 하지만 노드와 파드가 구동 중일 때는 영향을 받지 않는다.

**워커 노드 업그레이드**

1. 워커 노드를 한 번에 업그레이드한다: 업그레이드 동안 애플리케이션에 접근이 불가능하다.
2. 워커 노드 하나씩 업그레이드한다: 워커 노드마다 하나씩 업그레이드할 노드의 파드를 다른 실행 중인 노드에 옮기고 업그레이드를 한다. 완료 후 다시 옮기는 과정을 반복한다.
3. 버전 업그레이드한 새로운 노드를 클러스터에 추가하고, 기존 노드를 축출한다: 클라우드 제공자 환경에서 제공되는 기능이다.

### Kubeadm을 통한 버전 업그레이드

- 아래 명령어를 통해 업그레이드할 버전에 대한 정보를 알 수 있다.
  ```bash
  kubeadm upgrade plan
  ```
- 아래 명령어를 통해 업그레이드를 적용할 수 있다.
  ```bash
  kubeadm upgrade apply <version>
  ```
- kubeadm은 kubelet을 업그레이드하지 않는다. 수동으로 업그레이드해야 한다.
- kubeadm은 쿠버네티스 버전에 맞게 업그레이드되기 때문에 v1.11에서 바로 v1.13으로 업그레이드된다. 하지만 한 번에 한 버전씩 업그레이드하는 방법이 추천되기 때문에 아래와 같은 버전으로 버전을 한
  단계씩 올리도록 하자.
  ```bash
  apt-get upgrade -y kubeadm=1.12.0-00
  kubeadm upgrade apply v1.12.0
  ```

**마스터 노드 업그레이드**

- `kubectl get nodes` 명령어를 통해 각 노드의 버전을 확인해도 버전이 변하지 않은 것을 확인할 수 있다.
- 이는 kube-apiserver의 버전이 아닌, kube-apiserver에 등록된 kubelet의 버전이기 때문이다.
- 마스터 노드의 kubelet도 수동으로 업그레이드해야 한다.
- 아래 명령어를 통해 kubelet을 업그레이드할 수 있다.
  ```bash
  apt-get upgrade -y kubelet-1.12.0-00
  systemctl restart kubelet
  ```

**워커 노드 업그레이드**

- node-1, node-2, node-3의 세 개의 노드가 실행되고 있다고 가정한다.
- 아래 명령어를 통해 먼저 node-1을 비운다.
  ```bash
  kubectl drain node-1
  ```
- 이후 ssh를 통해 노드에 접속하여 버전 업그레이드를 진행한다.
  ```bash
  apt-get upgrade -y kubeadm=1.12.0-00
  apt-get upgrade -y kubelet=1.12.0-00
  kubeadm upgrade node config --kubelet-version v1.12.0
  systemctl restart kubelet
  ```
- 업그레이드 이후, 통제(cordon)를 해제하여 스케줄링이 가능하도록 변경한다. 기존의 파드가 다시 해당 노드에 스케줄 되진 않지만 새로운 파드는 스케줄 될 것이다.
  ```bash
  kubectl uncordon node-1
  ```
- 위 과정을 각 워커 노드마다 반복한다.
- 아래 명령어를 통해 업그레이드가 적용되었는지 확인할 수 있다.
  ```bash
  kubectl get nodes
  ```

## Backup and Restore Methods

- 백업은 중요하다. 깃허브와 같은 코드 레포지토리에 저장을 하면 재사용에도, 누군가와 공유하기도 편하기 때문에 큰 문제가 되지 않는다.
- 하지만 누군가가 명령형 방식으로 리소스를 생성하였고, 이에 대한 문서도 전혀 남아 있지 않다면 어떻게 백업을 해야 할까?
  - 모든 리소스를 파일로 저장하는 방법이 있다. 이는 일부 리소스 그룹에만 해당된다.
    ```bash
    kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml
    ```

### ETCD Backup

- etcd 클러스터는 클러스터의 상태에 대해 모든 정보를 저장한다.
- 모든 리소스를 백업하는 대신, etcd 서버 자체를 백업할 수 있다.
- etcd 클러스터는 마스터 노드에 호스팅되며, `--data-dir` 옵션을 통해 모든 데이터가 저장되는 위치를 확인할 수 있다.

### ETCD Snapshot

- etcd에는 스냅샷 솔루션이 내장되어 있다.
- 아래 명령어를 통해 snapshot.db라는 스냅샷을 생성할 수 있다.
  ```bash
  ETCDCTL_API=3 etcdctl --endpoints 127.0.0.1:2379 \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    snapshot save snapshot.db
  ```
- 아래 명령어를 통해 저장된 스냅샷 파일을 확인할 수 있다.
  ```bash
  ETCDCTL_API=3 etcdctl snapshot status snapshot.db
  ```

### ETCD Restore

- 아래 명령어를 통해 kube-apiserver를 중지시킨다.
  ```bash
  service kube-apiserver stop
  ```
- 아래 명령어를 통해 etcd 클러스터를 복구한다. `--data-dir` 옵션에 명시한 경로에 새로운 데이터 디렉토리가 생성된다.
  ```bash
  ETCDCTL_API=3 etcdctl \
    snapshot restore snapshot.db \
    --data-dir /var/lib/etcd-from-backup
  ```
- etcd.service의 `--data-dir` 옵션을 수정하여 새로운 데이터 디렉토리를 사용하도록 한다.
- 이후 etcd를 재시동하고 kube-apiserver를 실행한다.
  ```bash
  systemctl daemon-reload
  service etcd restart
  service kube-apiserver start
  ```

### ETCD가 스태틱 파드인 경우 복구하기

- 아래 명령어를 통해 etcd 클러스터를 복구한다. `--data-dir` 옵션에 명시한 경로에 새로운 데이터 디렉토리가 생성된다.

  ```bash
  ETCDCTL_API=3 etcdctl \
    snapshot restore snapshot.db \
    --data-dir /var/lib/etcd-from-backup
  ```

- /etc/kubernetes/manifests/etcd.yaml에서 hostPath.path 필드를 수정하여 새로운 데이터 디렉토리를 사용하도록 한다.
  ```yaml
  - hostPath:
      path: /var/lib/etcd-from-backup
      type: DirectoryOrCreate
    name: etcd-data
  ```
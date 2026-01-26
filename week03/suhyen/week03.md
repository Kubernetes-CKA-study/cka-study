# CKA 직전대비 ... (응급용)

26/2/2이라는 마감기한이 믿겨지지 않아...    
일단 시험을 봐보기라도 해보자 라는 마인드로...    
응급대비용 공부를 해보았습니다...     
못 푼 문제 많음 주의...  

# 1. CKA 시험 후기

### 범위 바뀐 후 후기
https://imprint.tistory.com/573

https://velog.io/@z1z0b4v/3%EA%B0%9C%EC%9B%94-%EC%A7%80%EB%82%98%EC%84%9C-%EC%A0%81%EB%8A%94-CKA-%ED%95%A9%EA%B2%A9%ED%9B%84%EA%B8%B0%ED%8C%81

https://sunrise-min.tistory.com/entry/2025-CKA-%ED%95%A9%EA%B2%A9-%ED%9B%84%EA%B8%B0-%EC%9C%A0%ED%98%95-%EB%B3%80%EA%B2%BD-%EB%8C%80%EC%9D%91%EB%B2%95-%EB%B0%8F-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EB%AC%B8%EC%A0%9C-%EA%B2%BD%ED%97%98-%EA%B3%B5%EC%9C%A0

### 범위 바뀌기 전 후기

https://jiheon95.tistory.com/71

https://cumulus.tistory.com/search/CKA%20%EC%A4%80%EB%B9%84

https://peterica.tistory.com/540

### 시험 환경 후기
https://devops-james.tistory.com/97

# 2. 이번주 Lab 오답노트

### Logging & Monitoring

1. Monitor Cluster Components
    - kubectl top nodes: node별 CPU, 메모리 사용량 확인
    - kubectl top pods: pod별 CPU, 메모리 사용량 확인

2. Managing Application Logs
    - kubectl logs {pod 이름}: pod 내 컨테이너들 로그 확인 

### Application Lifecycle Management

1. Rolling Updates and Rollbacks
    - kubectl set image deployment/{deployment 이름} {container 이름}={image 이름}: deployment 이미지 바꾸기 

2. Commands and Arguments
    - 우선순위: Docker ENTRYPOINT/CMD < Kubernetes command/args 
        - Kubernetes의 command는 Docker의 ENTRYPOINT를 덮어쓴다
    - args 명령으로 줄 때 `--` 사용
        - 예시: kubectl run webapp-green --image=kodekloud/ webapp-color -- --color=green

3. Env Variables
    - kubectl get configmaps: configmap 목록 확인 

4. Secrets
> 추가 예정

5. Multi Container Pods
> 추가 예정

6. Init Containers
> 추가 예정

7. Manual Scaling
> 추가 예정

8. HPA
> 추가 예정

9. Install VPA
> 추가 예정

10. Modifying CPU resources in VPA 
> 추가 예정

### Cluster Maintenance

1. OS Upgrades
    - drain, uncordon, cordon
        - kubectl drain node01 --ignore-daemonsets: 노드에서 pod를 제거하고 노드를 스케줄링 불가능 상태로 만든다 
            - daemonosets은 노드를 위한 pod이기때문에 drain을 할 때는 서비스를 위한 pod만 적용하기 위해 해당 옵션을 붙여줘야 한다.
            - node drain할 때 pod를 지워도 컨트롤러(Deployment/ReplicaSet/Job 등)가 다시 만들어줄 수 있을 때만 지울 수 있다.
        - kubectl uncordon node01: 노드를 다시 스케줄링 가능하게 만든다 (cordon/drain으로 비활성화된 노드를 다시 활성화)
        - kubectl cordon node01: node01에 새롭게 올라가는 pod만 지운다
    - k8s에서 application 단위는 deployment로 간주
    - control-plane 노드에는 기본적으로 taint가 걸려 있어서 -> 일반 pod가 못 올라가지만 -> 현재 올라간 것으로 보아 해당 노드에 걸려진 taint가 없음을 알 수 있음

2. Cluster Upgrade Process
    - kubectl version: Server 버전이 클러스터 버전
    - control-plane 노드는 기본적으로 워크로드가 못 올라감
        - control-plane 노드는 기본적으로 taint가 있기 때문에, 일반 Pod는 스케줄되지 않음
    - ⭐ Kubernetes v1.33 → v1.34 업그레이드 설명 (controlplane 기준)
        1. controlplane node 비우기
            - kubectl drain controlplane --ignore-daemonsets
        2. repo를 v1.34로 변경 
            - vim /etc/apt/sources.list.d/kubernetes.list => v1.33 -> v1.34
            - apt update
            - apt-cache madison kubeadm
            - apt-get install kubeadm=1.34.0-1.1
        3. kubeadm 업그레이드
            - kubeadm upgrade plan v1.34.0
        4. kubeadm upgrade apply 
            - kubeadm upgrade apply v1.34.0
        5. kubelet 업그레이드
            - apt-get install kubelet=1.34.0-1.1
            - systemctl daemon-reload
            - systemctl restart kubelet
    - ⭐ Worker node 업그레이드 절차 (node01 기준)
        1. node01 노드 비우기
            - kubectl drain node01 --ignore-daemonsets
        2. contorlplane에서 worker(node01)로 이동 
            - ssh node01
        3. Kubernetes repo 및 패키지 준비
            - Kubernetes repo를 v1.34로 변경: vim /etc/apt/sources.list.d/kubernetes.list
            - 패키지 목록 갱신: apt update -> (확인용) apt-cache madison kubelet
            - kubelet hold 해제: apt-mark unhold kubelet -> (확인용) apt-mark showhold
        4. kubelet 업그레이드
            - kubelet 업그레이드(worker는 kubeadm X): apt-get install -y kubelet=1.34.0-1.1 
        5. kubelet 서비스 재시작
            - sudo systemctl daemon-reload
            - sudo systemctl restart kubelet
        6. controlplane으로 돌아가서 스케줄링 다시 허용
            - node01 ssh 세션 종료하고 controlplane으로 이동: exit 
            - kubectl uncordon node01

3. Backup and Restore Methods
    - etcd version 확인 
        1. kubectl get all -n kube-system
        2. kubectl describe pod etcd-controlplane -n kube-system -> image에 적힌 버전 체크
> 추가 예정

# 3. CKA YT Dump 1 정리

### Practice Question #1 ArgoCD

<details>
<summary>문제</summary>

Install Argo CD in a Kubernetes cluster using Helm while ensuring that CRDs are not installed (as they are pre-installed). Follow the steps below: 

Requirements:

Add the official Argo CD Helm repository with the name argo.

Generate a Helm template from the Argo CD chart version 7.7.3 for the argocd namespace.

Ensure that CRDs are not installed by configuring the chart accordingly.

Save the generated YAML manifest to /home/argo/argo-helm.yaml.

---

CRD(Custom Resource Definitions)가 이미 설치되어 있는 환경에서, Helm을 사용하여 쿠버네티스 클러스터에 Argo CD를 설치(준비)하십시오. 아래의 지침을 따르십시오.

요구 사항:

1. 공식 Argo CD Helm 저장소를 argo라는 이름으로 추가하십시오.
2. argocd 네임스페이스를 대상으로, Argo CD 차트 7.7.3 버전을 사용하여 Helm 템플릿을 생성하십시오.
3. 차트 설정을 적절히 구성하여 CRD가 설치되지 않도록 하십시오 (CRD가 이미 존재하기 때문입니다).
4. 생성된 YAML 매니페스트 파일을 /home/argo/argo-helm.yaml 경로에 저장하십시오.

</details>

<details>
<summary>풀이</summary>

> 1번: ArgoCD Helm 설치 및 매니페스트 추출     
> 평가 내용: Helm을 사용하여 클러스터에 직접 배포하는 대신, 특정 조건이 반영된 YAML 파일을 생성하는 능력     

### 1. 이 문제를 풀이하기 위해 필요한 개념

#### Helm Repository & Chart

Helm은 쿠버네티스의 패키지 매니저입니다. 애플리케이션 설치에 필요한 리소스 정의서들의 묶음인 차트(Chart)를 저장소(Repository)에서 받아와 사용합니다.

#### Helm Template

`helm install`이 클러스터에 즉시 배포한다면, `helm template`은 차트의 변수들을 실제 값으로 치환하여 **배포될 최종 YAML 파일 내용만 확인**하거나 파일로 저장할 때 사용합니다.

#### CRD (Custom Resource Definition) 제어

ArgoCD 같은 복잡한 애플리케이션은 자신만의 리소스 타입(CRD)을 정의합니다. 문제에서 "CRD는 이미 설치되어 있으니 제외하라"는 조건이 붙을 경우, `--skip-crds` 옵션을 사용하여 순수 리소스들만 매니페스트에 포함시켜야 합니다.

### 2. 이 문제를 풀이하는 방법

시험 환경의 터미널에서 다음 순서대로 명령어를 입력하세요.

#### Step 1: Argo CD 레포지토리 추가 및 업데이트

문제에서 요구한 이름(`argo`)으로 공식 저장소를 추가하고 최신 정보를 가져옵니다.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

#### Step 2: 조건에 맞는 YAML 템플릿 생성 및 저장

요구사항(버전 7.7.3, 네임스페이스 argocd, CRD 제외, 특정 경로 저장)을 모두 포함하여 명령어를 실행합니다.

```bash
helm template argocd argo/argo-cd \
  --version 7.7.3 \
  --namespace argocd \
  --skip-crds \
  > /home/argo/argo-helm.yaml
```

#### 추가: 옵션 상세 설명

- `argocd`: 릴리스 이름입니다.
- `argo/argo-cd`: `레포지토리이름/차트이름` 형식입니다.
- `--version 7.7.3`: 문제에서 지정한 정확한 버전입니다.
- `--namespace argocd`: 생성될 리소스들의 네임스페이스를 지정합니다.
- `--skip-crds`: CRD 설치 파일을 결과물에서 제외합니다.
- `> /home/argo/argo-helm.yaml`: 생성된 내용을 지정된 경로에 파일로 저장합니다.

### 3. 요약

| 요구사항 | 해결 방법 | 주의 사항 |
| --- | --- | --- |
| Repo 이름 | `argo` | 이름 오타 주의 |
| 차트 버전 | `--version 7.7.3` | 버전 번호 정확히 기재 |
| 네임스페이스 | `--namespace argocd` | 해당 네임스페이스 정보가 YAML에 포함됨 |
| CRD 제외 | `--skip-crds` | 문제의 핵심 제약 조건 |
| 결과 저장 | `> [파일경로]` | 경로 오타 방지를 위해 지문 복사 추천 |

### 4. 관련 문서 링크

- Helm Repo 관리: [Helm Docs: helm repo add](https://helm.sh/docs/helm/helm_repo_add/)
- Helm Template 가이드: [Helm Docs: helm template](https://helm.sh/docs/helm/helm_template/)
- CRD 처리 방식: [Helm Docs: CRD handling in Helm](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/)

</details>

### Practice Question #2 SideCar

<details>
<summary>문제</summary>

Update the existing deployment wordpress, adding a sidecar container named sidecar using the busybox:stable image to the exising pod.

The new sidecar container has to run the following command: “bin/sh -c “tail -f /var/log/workdpress.log” use a volume mounted at /var/log to make the log file wordpress.log available to the co-located container.

---

기존의 wordpress Deployment를 업데이트하여, sidecar라는 이름의 사이드카 컨테이너를 기존 파드(Pod)에 추가하십시오.

1. 이미지: busybox:stable을 사용하십시오.
2. 실행 명령: 사이드카 컨테이너는 반드시 /bin/sh -c "tail -f /var/log/wordpress.log" 명령을 실행해야 합니다.
3. 로그 공유: /var/log 경로에 마운트된 볼륨(Volume)을 사용하여, 같은 파드 내에 있는 컨테이너가 wordpress.log 파일에 접근할 수 있도록 설정하십시오.

</details>

<details>
<summary>풀이</summary>

> 2번: Sidecar(사이드카) 컨테이너
> 평가 내용: 하나의 파드 안에서 두 개의 컨테이너가 볼륨을 통해 어떻게 데이터를 주고받는지 이해하는 능력

### 1. 이 문제를 풀이하기 위해 필요한 개념

#### Sidecar 패턴

애플리케이션 컨테이너(주 컨테이너)를 돕는 보조 컨테이너를 같은 파드 안에 배치하는 디자인 패턴입니다. 주로 로그 수집, 모니터링, 프록시 역할을 수행합니다.

#### 공유 볼륨 (Shared Volume)

파드 내의 컨테이너들은 네트워크 네임스페이스뿐만 아니라 **스토리지 볼륨**도 공유할 수 있습니다.

- 한 컨테이너가 볼륨에 파일을 쓰면, 다른 컨테이너는 즉시 그 파일을 읽을 수 있습니다.
- 이 문제에서는 `emptyDir` 볼륨을 사용하여 `/var/log` 경로를 서로 연결하는 것이 핵심입니다.

### 2. 이 문제를 풀이하는 방법

시험장에서는 기존 리소스를 안전하게 수정하기 위해 YAML 파일로 추출한 뒤 작업하는 방식을 추천합니다.

#### Step 1: 기존 Deployment를 YAML 파일로 저장

```bash
kubectl get deployment wordpress -o yaml > wordpress-sidecar.yaml
```

#### Step 2: YAML 파일 수정

`vi wordpress-sidecar.yaml` 명령어로 파일을 열어 다음 세 가지 요소를 추가/수정합니다. (문제의 오타인 `workdpress.log`는 실제 환경에 맞춰 `wordpress.log`로 교정하여 풀이합니다.)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  template:
    spec:
      containers:
      - name: wordpress        # 1. 기존 메인 컨테이너에 마운트 추가
        image: wordpress
        volumeMounts:
        - name: log-storage
          mountPath: /var/log
      - name: sidecar          # 2. 사이드카 컨테이너 정의 추가
        image: busybox:stable
        command: ["/bin/sh", "-c", "tail -f /var/log/wordpress.log"]
        volumeMounts:
        - name: log-storage
          mountPath: /var/log
      volumes:                 # 3. 컨테이너들이 공유할 볼륨 정의
      - name: log-storage
        emptyDir: {}
```

#### Step 3: 수정된 설정 적용

```bash
kubectl apply -f wordpress-sidecar.yaml
```

### 3. 요약

| 작업 항목 | 위치 | 설정 내용 |
| --- | --- | --- |
| 공유 볼륨 정의 | `spec.template.spec.volumes` | `name` 지정 및 `emptyDir: {}` 설정 |
| **메인 컨테이너 설정 | `spec.template.spec.containers[0]` | `/var/log` 경로에 볼륨 마운트 |
| 사이드카 추가 | `spec.template.spec.containers[1]` | 이름, 이미지, 실행 명령어, 볼륨 마운트 설정 |
| 마운트 경로 | `volumeMounts.mountPath` | 두 컨테이너 모두 `/var/log`로 일치 |

### 4. 관련 문서 링크

- 파드 내 컨테이너 간 통신 (볼륨 공유): [Kubernetes Docs: Communicate Between Containers in the Same Pod](https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/)
- 볼륨 (emptyDir): [Kubernetes Docs: Volumes - emptyDir](https://www.google.com/search?q=https://kubernetes.io/docs/concepts/storage/volumes/%23emptydir)
- 컨테이너 실행 명령어 설정: [Kubernetes Docs: Define a Command and Arguments for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)

</details>

### Practice Question #3 GatewayAPI

<details>
<summary>문제</summary>

You have an existing web application deployed in a Kubernetes cluster using an Ingress resource named web. You must migrate the existing Ingress configuration to the new Kubernetes Gateway API, maintaining the existing HTTPS access configuration.

Tasks:

Create a Gateway resource named web-gateway with hostname gateway.web.k8s.local that maintains the existing TLS and listener configuration from the existing Ingress resource named web.

Create an HTTPRoute resource named web-route with hostname gateway.web.k8s.local that maintains the existing routing rules from the current Ingress resource named web.

Note: A GatewayClass named nginx-class is already installed in the cluster.

---

귀하는 현재 쿠버네티스 클러스터에서 web이라는 이름의 Ingress 리소스를 사용하여 배포된 웹 애플리케이션을 관리하고 있습니다. 기존의 HTTPS 접속 설정을 그대로 유지하면서, 현재의 Ingress 구성을 새로운 Kubernetes Gateway API로 마이그레이션해야 합니다.

작업 지침:

1. Gateway 리소스 생성: web Ingress 리소스의 기존 TLS 및 리스너(listener) 설정을 유지하는 web-gateway라는 이름의 Gateway 리소스를 생성하십시오. 호스트네임은 gateway.web.k8s.local로 설정해야 합니다.
2. HTTPRoute 리소스 생성: 현재 web Ingress의 라우팅 규칙을 유지하는 web-route라는 이름의 HTTPRoute 리소스를 생성하십시오. 호스트네임은 gateway.web.k8s.local로 설정해야 합니다.

참고: 클러스터에는 nginx-class라는 이름의 GatewayClass가 이미 설치되어 있습니다.

</details>

<details>
<summary>풀이</summary>

> 3번: Gateway API    
> 평가 내용: 최신 Kubernetes 트렌드인 Gateway API를 사용하여 기존 Ingress 설정을 마이그레이션하는 능력    

### 1. 이 문제를 풀이하기 위해 필요한 개념

#### Gateway API

Ingress의 한계를 극복하기 위해 등장한 차세대 표준입니다. 기존 Ingress가 하나의 리소스에 모든 설정을 담았다면, Gateway API는 이를 세분화하여 관리합니다.

- GatewayClass: 인프라 제공자가 설정한 템플릿입니다 (문제의 `nginx-class`).
- Gateway: 네트워크의 진입점(Entry point)입니다. 포트, 프로토콜, TLS 설정을 관리합니다.
- HTTPRoute: 트래픽을 어떤 서비스로 보낼지 결정하는 라우팅 규칙입니다. `parentRefs`를 통해 특정 Gateway와 연결됩니다.

#### 마이그레이션 핵심

- 기존 Ingress의 TLS 설정은 `Gateway` 리소스로 옮깁니다.
- 기존 Ingress의 Path 및 Backend 서비스 설정은 `HTTPRoute` 리소스로 옮깁니다.

#### 참고

- Evolving Kubernetes networking with the Gateway API
    ![03-k8s-gateway-1](./img/03-k8s-gateway-1.png)
    - GatewayClass는 클러스터 스코프
    - Gateway/Route는 네임스페이스 스코프

- Kubernetes Ingress Vs Gateway API 
    ![03-k8s-gateway-2](./img/03-k8s-gateway-2.jpeg)

### 2. 이 문제를 풀이하는 방법

Gateway API는 리소스 구조가 복잡하므로, 시험장에서는 공식 문서의 예시를 활용해 YAML 파일을 만드는 것이 가장 정확합니다.

#### Step 1: Gateway 생성 (`web-gateway`)

TLS와 호스트네임 설정을 포함하는 Gateway 리소스를 작성합니다.

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1  # 시험 환경의 apiVersion 확인 필수
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: nginx-class
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: gateway.web.k8s.local
    tls:
      mode: Terminate
      certificateRefs:
      - name: web-tls-secret  # 기존 Ingress에서 사용하던 Secret 이름
    allowedRoutes:
      namespaces:
        from: Same
```

### Step 2: HTTPRoute 생성 (`web-route`)

호스트네임과 서비스 라우팅 규칙을 정의하고 위에서 만든 Gateway에 연결합니다.

```yaml
# route.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - gateway.web.k8s.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service      # 실제 트래픽을 받을 서비스 이름
      port: 80
```

### Step 3: 리소스 적용

```bash
kubectl apply -f gateway.yaml
kubectl apply -f route.yaml
```

### 3. 요약

| 항목 | 리소스 | 주요 설정 필드 |
| --- | --- | --- |
| 진입점 & TLS | `Gateway` | `gatewayClassName`, `listeners.tls`, `hostname` |
| 라우팅 규칙 | `HTTPRoute` | `parentRefs` (연결할 Gateway명), `rules.backendRefs` |
| 호스트네임 | 둘 다 설정 | `gateway.web.k8s.local` (문제 지정 값) |

> **⚠️ 실전 팁:** 시험 환경에 따라 `apiVersion`이 `v1beta1`일 수도 있습니다. `kubectl explain gateway` 명령어로 현재 클러스터에서 지원하는 버전을 반드시 확인하세요.

### 4. 관련 문서 링크

- Gateway API 개요: [Kubernetes Docs: Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- Gateway 설정 가이드: [Gateway API 공식 문서: Gateway](https://www.google.com/search?q=https://gateway-api.sigs.k8s.io/guides/gateway/)
- HTTPRoute 설정 가이드: [Gateway API 공식 문서: HTTPRoute](https://www.google.com/search?q=https://gateway-api.sigs.k8s.io/guides/httproute/)

</details>

### Practice Question #4 Resource Request

<details>
<summary>문제</summary>

You are managing a WordPress application running in a Kubernetes cluster.

Your task is to adjust the Pod resouce requests and limits to ensure stable operation. Follow the instructions below:

1. Scale down the wordpress Deployment to 0 replicas.
2. Edit the Deployment and divide node resources evenly across all 3 Pods.
3. Assign fair and equal CPU and memory requests to each Pod.
4. Add sufficient overhead to avoid node instability.

Ensure that both the init containers and main containers use exactly the same resource requests and limits.

After making the changes, scale the Deployment back to 3 replicas.

---

귀하는 쿠버네티스 클러스터에서 실행 중인 WordPress 애플리케이션을 관리하고 있습니다.

귀하의 임무는 안정적인 운영을 위해 Pod의 리소스 요청(Requests) 및 제한(Limits)을 조정하는 것입니다. 다음 지침을 따르십시오.

1. wordpress Deployment의 복제본(replicas) 수를 0으로 줄이십시오.
2. Deployment를 편집하여 3개의 모든 Pod에 노드 리소스를 균등하게 분배하십시오.
3. 각 Pod에 공정하고 동일한 CPU 및 메모리 요청량을 할당하십시오.
4. 노드 불안정성을 방지하기 위해 충분한 오버헤드(여유 자원)를 추가하십시오.

초기화 컨테이너(init containers)와 메인 컨테이너가 정확히 동일한 리소스 요청량과 제한량을 사용하도록 설정해야 합니다.

변경 사항을 모두 적용한 후, Deployment를 다시 3개의 복제본으로 확장하십시오.

</details>

<details>
<summary>풀이</summary>

> 4번: 리소스 할당(Resource Requests & Limits)
> 평가 내용: 노드의 가용 자원을 파악하고, 여러 컨테이너(Init 포함)에 자원을 배분하는 능력

### 1. 이 문제를 풀이하기 위해 필요한 개념

#### Resource Requests vs Limits

- Requests: 컨테이너가 실행되기 위해 보장받아야 하는 최소 자원량입니다. 스케줄러는 이 값을 기준으로 파드를 배치할 노드를 결정합니다.
- Limits: 컨테이너가 사용할 수 있는 최대 자원량입니다. 이 수치를 넘어가면 CPU는 제한(Throttling)되고, 메모리는 종료(OOM Kill)될 수 있습니다.

#### Init Container 리소스 관리

파드 내에 Init Container와 메인 컨테이너가 모두 있을 때, K8s는 다음과 같이 자원을 계산합니다.

- Requests/Limits 합산 방식: `max(모든 Init 컨테이너의 리소스 합, 모든 메인 컨테이너의 리소스 합)`
- 하지만 이 문제에서는 "Init과 메인 컨테이너에 동일한 리소스를 할당하라"고 명시했으므로, 계산된 값을 모든 컨테이너 항목에 동일하게 작성하면 됩니다.

#### 노드 자원 확인

문제에서 "노드 자원을 균등하게 배분"하라고 했으므로, 현재 파드가 실행될 노드의 전체 사양을 먼저 확인해야 합니다.

### 2. 이 문제를 풀이하는 방법

#### Step 1: 현재 배포된 Deployment 스케일 다운

작업 전 상태를 정리하기 위해 복제본을 0으로 만듭니다.

```bash
kubectl scale deployment wordpress --replicas=0
```

#### Step 2: 노드 리소스 용량 확인

노드의 전체 CPU와 Memory 용량을 확인합니다. (시험 환경의 노드 이름을 확인하세요.)

```bash
kubectl describe node <node-name> | grep -A 5 Capacity
```

> **예시 상황:** 노드 용량이 CPU 4000m(4 core), Memory 8Gi라고 가정하고, "충분한 오버헤드"를 고려해 실제 할당 가능 자원을 CPU 3000m, Memory 6Gi로 잡는다면:
> - 3개의 Pod에 배분 시: **Pod당 CPU 1000m, Memory 2Gi**

#### Step 3: Deployment 수정 (Resources 설정)

`kubectl edit deployment wordpress` 명령어로 파일을 열어 `template.spec` 부분을 수정합니다. **Init Container가 있다면 해당 부분도 똑같이 수정해야 함에 주의하세요.**

```yaml
spec:
  replicas: 0 # 이미 0으로 바뀐 상태
  template:
    spec:
      initContainers:      # 1. Init 컨테이너 설정
      - name: install
        image: busybox
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
      containers:          # 2. 메인 컨테이너 설정
      - name: wordpress
        image: wordpress
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
```

#### Step 4: Deployment 스케일 업

수정이 끝난 후 다시 3개로 늘려줍니다.

```bash
kubectl scale deployment wordpress --replicas=3
```

### 3. 요약

| 작업 순서 | 명령어 / 주의 사항 |
| --- | --- |
| 1. 스케일 다운 | `kubectl scale deployment [이름] --replicas=0` |
| 2. 자원 확인 | `kubectl describe node`로 Capacity 확인 |
| 3. 리소스 계산 | (전체 - 오버헤드) / 3 = Pod당 할당량 |
| 4. 동일 설정 | Init 컨테이너와 Main 컨테이너 수치를 반드시 동일하게 작성 |
| 5. 스케일 업 | `kubectl scale deployment [이름] --replicas=3` |

### 4. 관련 문서 링크

- 컨테이너 리소스 관리: [Kubernetes Docs: Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- Init 컨테이너 리소스 계산법: [Kubernetes Docs: Init Containers - Resources](https://www.google.com/search?q=https://kubernetes.io/docs/concepts/workloads/pods/init-containers/%23resources)
- Deployment 스케일링: [Kubernetes Docs: Scaling a Deployment](https://www.google.com/search?q=https://kubernetes.io/docs/concepts/workloads/controllers/deployment/%23scaling-a-deployment)

</details>

### Practice Question #5 StorageClass

<details>
<summary>문제</summary>

Create a new StorageClass named local-kiddle with the provisioner rancher.io/local-path.

Set the volumeBindingMode to WaitForFirstConsumer.

Configure the StorageClass as the default StorageClass.

Do not modify any existing Deployments or PersistentVolumeClaims.

---

다음 조건에 따라 새로운 StorageClass를 생성하십시오.

1. 이름: local-kiddle
2. 프로비저너(Provisioner): rancher.io/local-path
3. 볼륨 바인딩 모드(volumeBindingMode): WaitForFirstConsumer로 설정하십시오.
4. 기본 클래스 설정: 이 StorageClass를 클러스터의 기본(default) StorageClass로 구성하십시오.

주의 사항: 기존에 존재하는 어떤 Deployment나 PersistentVolumeClaim(PVC)도 수정하지 마십시오.

</details>

<details>
<summary>풀이</summary>

> 5번: StorageClass(SC) 설정에 관한 문제   
> 평가 내용: 특정 프로비저너를 사용하는 저장소 클래스를 만들고, 이를 클러스터의 기본(Default) 클래스로 지정하는 능력

### 1. 이 문제를 풀이하기 위해 필요한 개념

#### StorageClass (SC)

StorageClass는 관리자가 제공하는 스토리지의 "추상화된 유형"을 정의합니다. PVC(PersistentVolumeClaim)가 요청을 보내면 SC에 정의된 설정에 따라 실제 볼륨(PV)이 자동으로 생성됩니다.

#### volumeBindingMode: WaitForFirstConsumer

- Immediate (기본값): PVC가 생성되는 즉시 볼륨을 바인딩하고 프로비저닝합니다.
- WaitForFirstConsumer: PVC를 사용하는 파드가 실제로 스케줄링될 때까지 볼륨 생성을 지연합니다. 로컬 스토리지처럼 특정 노드에 종속적인 저장소를 사용할 때 필수적인 설정입니다.

#### Default StorageClass

클러스터에 여러 SC가 있을 때, 사용자가 특정 SC를 지정하지 않고 PVC를 생성하면 자동으로 할당되는 클래스입니다. 이를 설정하려면 메타데이터에 특정 Annotation을 추가해야 합니다.

#### 참고

![03-k8s-volume](./img/03-k8s-volume.jpg)

### 2. 이 문제를 풀이하는 방법

StorageClass 리소스는 `kubectl create`로 세부 옵션을 모두 지정하기 어려우므로, YAML 파일을 작성하여 실행하는 것이 가장 정확합니다.

#### Step 1: StorageClass YAML 파일 작성

`local-sc.yaml` 파일을 생성하고 아래 내용을 작성합니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-kiddle
  annotations:
    # 기본 StorageClass로 설정하는 핵심 어노테이션
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
```
#### Step 2: 리소스 생성

작성한 파일을 클러스터에 적용합니다.

```bash
kubectl apply -f local-sc.yaml
```

#### Step 3: 설정 결과 확인

생성된 StorageClass 목록을 확인합니다. 이름 옆에 `(default)` 표시가 있는지 확인하는 것이 핵심입니다.

```bash
kubectl get sc
```

출력 예시:  

```text
NAME                     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-kiddle (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  10s
```

### 3. 요약

| 설정 항목 | 설정 값 / 명령어 | 비고 |
| --- | --- | --- |
| 이름 | `local-kiddle` | `metadata.name` |
| 프로비저너 | `rancher.io/local-path` | `provisioner` |
| 바인딩 모드 | `WaitForFirstConsumer` | 로컬 스토리지 필수 옵션 |
| 기본 클래스 설정 | `storageclass.kubernetes.io/is-default-class: "true"` | `metadata.annotations`에 추가 |

> **⚠️ 주의사항:** 이미 다른 StorageClass가 `default`로 설정되어 있다면, 기존 클래스의 어노테이션을 `false`로 바꾸거나 삭제해야 충돌이 발생하지 않습니다. (문제 조건에 "기존 배포를 수정하지 말 것"이라고 했으므로, 중복 여부만 체크하세요.)

### 4. 관련 문서 링크

- StorageClass 개념 및 설정: [Kubernetes Docs: Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- 기본 StorageClass 변경하기: [Kubernetes Docs: Change the default StorageClass](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)
- 볼륨 바인딩 모드 설명: [Kubernetes Docs: Volume Binding Mode](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)

</details>

### Practice Question #6 PriorityClass

<details>
<summary>문제</summary>

You're working in a Kubernetes cluster with an existing Deployment named busybox-logger running in a namespace called priority.

The cluster already has at least one user-defined Priority Class

Perform the following tasks:

1. Create a new Priority Class named high-priority for user workloads. The value of this Priority Class should be exactly one less than the highest existing user-defined Priority Class value.
2. Patch the existing Deployment busybox-logger in the priority namespace to use the newly created high-priority Priority Class.

---

현재 귀하는 priority라는 네임스페이스에서 실행 중인 busybox-logger라는 기존 Deployment가 포함된 Kubernetes 클러스터에서 작업하고 있습니다.

해당 클러스터에는 이미 하나 이상의 사용자 정의 Priority Class가 존재합니다.

다음 작업을 수행하십시오:

1. 사용자 워크로드를 위한 high-priority라는 이름의 새로운 Priority Class를 생성하십시오. 이 Priority Class의 값(value)은 기존에 정의된 사용자 정의 Priority Class 값 중 가장 높은 값보다 정확히 1이 작아야 합니다.
2. priority 네임스페이스에 있는 기존 Deployment인 busybox-logger가 새로 생성한 high-priority Priority Class를 사용하도록 패치(Patch)하십시오.

</details>

<details>
<summary>풀이</summary>

> 6번: Priority(파드 우선순위)에 대한 문제    
> 평가 내용: 클러스터 내의 자원이 부족할 때 어떤 파드를 먼저 실행하거나 유지할지 결정하는 기준을 설정하고 적용하는 능력

### 1. 이 문제를 풀이하기 위해 필요한 개념

#### PriorityClass란?
PriorityClass는 파드의 상대적인 **중요도**를 나타내는 값(Integer)입니다.
- 값이 높을수록 우선순위가 높습니다.
- 자원이 부족해지면, 스케줄러는 우선순위가 낮은 파드를 퇴거(Eviction)시키고 우선순위가 높은 파드를 그 자리에 배치합니다.

#### 최고 값 찾기 및 계산
문제에서 "기존 사용자 정의 PriorityClass 중 가장 높은 값보다 딱 1 작은 값"을 설정하라고 했습니다. 따라서 먼저 시스템 기본값을 제외한 사용자 정의 클래스들의 값을 조회하고 비교하는 과정이 필요합니다.

### 2. 이 문제를 풀이하는 방법

#### Step 1: 기존 PriorityClass 확인 및 최고값 계산

시스템 리소스(`system-node-critical` 등)를 제외하고 사용자가 만든 클래스들의 값을 확인합니다.

```
kubectl get pc
```

출력 결과가 다음과 같다고 가정해 봅시다.
- `low-priority`: 1000
- `mid-priority`: 5000
- `system-cluster-critical`: 2000000000 (시스템 기본값은 보통 제외)

여기서 가장 높은 사용자 정의 값은 **5000**입니다. 따라서 우리가 만들 클래스의 값은 **5000 -1 = 4999**가 됩니다.

#### Step 2: 새로운 PriorityClass 생성 (`high-priority`)

`kubectl create` 명령어를 사용하거나 YAML을 작성합니다.

- 명령어 방식
```bash
kubectl create priorityclass high-priority --value=4999 --description="Priority Class for user workloads"
```
- YAML 방식
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 4999
globalDefault: false
description: "Priority Class for user workloads"
```

#### Step 3: 기존 Deployment에 패치 적용

`priority` 네임스페이스에 있는 `busybox-logger` Deployment가 새 PriorityClass를 사용하도록 수정합니다. `kubectl patch`를 사용하면 가장 빠릅니다.

```bash
kubectl patch deployment busybox-logger -n priority -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority"}}}}'
```

파일 수정이 편하다면 `kubectl edit deployment busybox-logger -n priority`를 실행하여 `spec.template.spec.priorityClassName` 항목을 직접 추가/수정해도 됩니다.

### 3. 요약

| 단계 | 실행 명령어 / 확인 사항 | 주의 사항 |
| --- | --- | --- |
| 1. 값 조회 | `kubectl get pc` | `VALUE` 열에서 가장 큰 숫자를 확인 |
| 2. PC 생성 | `kubectl create pc [이름] --value=[계산값]` | 문제에서 지정한 이름 오타 주의 |
| 3. 적용(Patch) | `kubectl patch deployment [이름] ...` | 네임스페이스(`-n priority`) 지정 필수 |
| 4. 확인 | `kubectl get deploy [이름] -n priority -o yaml` | `priorityClassName`이 반영되었는지 확인 |

### 4. 관련 문서 링크

- Pod 우선순위 및 선점: [Kubernetes Docs: Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- PriorityClass 리소스: [Kubernetes Docs: PriorityClass](https://www.google.com/search?q=https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/%23priorityclass)
- kubectl patch 사용법: [Kubernetes Docs: Update API Objects in Place Using kubectl patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)

</details>

### Practice Question #7 Ingress
> 추가 예정

<details>
<summary>문제</summary>

Create a new ingress resource named echo in echo-sound namespace

With the following tasks:

1. Expose the deployment with a service named echo-service on http://example.org/echo using Service port 8080 type=NodePort.
2. The availability of Service echo-service can be checked using the following command which should return 200:
    
curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #8 CRD’s
> 추가 예정

<details>
<summary>문제</summary>

Task:

1. Create a list of all cert-manager [CRDs] and save it to ~/resources.yaml
    
    Make sure kubectl’s use default output format and use kubectl to list CRD's
    
2. Using kubectl, extract the documentation for the subject specification field of the Certificate Custom Resource and save it to ~/subject.yaml

You may use any output format that kubectl supports.

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #9 NetworkPolicy
> 추가 예정

<details>
<summary>문제</summary>

There are 2 deployments, Frontend and Backend

Frontend will be in frontend namespace and Backend will be in backend namespace.

Task:

Create a network policy to have interaction between frontend and backend deployment. The network policy has to be least permissive.

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #10 HPA
> 추가 예정

<details>
<summary>문제</summary>

Create a new HorizontalPodAutoScaler [HPA] named apache-server in the autoscale namespace.

Tasks:

1. This HPA must target the existing deployment called apache-deployment in the autoscale namespace.
2. Set the HPA to target for 50% CPU usage per Pod.
3. Configure the HPA to have a minimum of 1 pod and maximum of 4 pods. Also, we have to set the downscale stabilization window to 30 seconds.

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #11 CNI
> 추가 예정

<details>
<summary>문제</summary>

Install and configure a CNI of your choice that meet the specified requirements,

choose one of the following;

Flannel (v0.26.1) using the manifest: [kube-flannel.yml
(https://github.com/flannel-io/flannel/releases/download/v0.26.1/kube-flannel.yml)

Calico (v3.28.2) using the manifest: [tigera-operator.yaml
(https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml)

The CNI you choose must:

1. Let pods communicate with each other
2. Support network policy enforcement
3. Install from manifest

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #12 PVC
> 추가 예정

<details>
<summary>문제</summary>

A Persistent Volume already exists and is retained for reuse.

Create a PersistentVolumeClaim named MariaDB in the mariadb namespace as follows:

1. Access mode ReadWriteOnce
2. Storage capacity 250Mi

Edit the maria-deployment in the file located at maria-deploy.yaml to use the newly created PVC.

Verify that the deployment is running and is stable.

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #13 CRI-Dockered
> 추가 예정

<details>
<summary>문제</summary>

Set up cri-dockerd

Install the Debian package ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb using dpkg.

Enable and start the cri-docker service

Configure these system parameters:

1. Set net.bridge.bridge-nf-call-iptables to 1
2. Set net.ipv6.conf.all.forwarding to 1
3. Set net.ipv4.ip_forward to 1
4. Set net.netfilter.nf_conntrack_max to 131072

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #14 Troubleshoot
> 추가 예정

<details>
<summary>문제</summary>

After a cluster migration, the controlplane kube-apiserver is not coming up.

Before migration:

etcd was external and in HA

After migration, kube-apiserver was pointing to etcd peer port 2380 instead of 2379

Fix it.

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #15 Scheduling
> 추가 예정

<details>
<summary>문제</summary>

Task:

1. Add a taint to node01 so that no normal pods can be scheduled in the node.
2. Key=IT Value=Kiddie Type=NoSchedule
3. Schedule a Pod on node01 adding the correct toleration to the spec and ensure that it lands on the correct node.

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #16 NodePort
> 추가 예정

<details>
<summary>문제</summary>

There is a deployment named nodeport-deployment in the relative namespace.

Tasks:

1. Configure the deployment so it can be exposed using port 80 and protocol TCP name http.
2. Create a new Service named nodeport-service exposing the container port 80 and TCP.
3. Configure the new Service to also expose the individual pods using NodePort.

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

### Practice Question #17 TLS
> 추가 예정

<details>
<summary>문제</summary>

There is an existing deployment called nginx-static in the nginx-static namespace.

The deployment contains a ConfigMap that supports TLSv1.2 and TLSv1.3, and a Secret for TLS.

There is a service called nginx-static in the nginx-static namespace that is currently exposing the deployment.

Tasks:

1. Configure the configmap to only support TLSv1.3
2. Add the IP address of the service in /etc/hosts and name ITKiddie.k8s.local
3. Verify that everything is working using the following command:
    
    curl --tls-max 1.2 https://ITKiddie.k8s.local -k   (TLSv1.2 should not work)
    curl --tlsv1.3 https://ITKiddie.k8s.local -k

</details>

<details>
<summary>풀이</summary>

> 추가 예정

</details>

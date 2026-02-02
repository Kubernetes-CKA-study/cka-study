# Workload Security
- 사용자 접근, 인증/인가 권한 뿐만 아니라 컨테이너 워크로드의 보안도 중요하다. 
- 컨테이너는 결국 클러스터의 `node` 위에서 실행되는 하나의 프로세스이므로, 컨테이너 이미지나 런타임 상황에서 취약점이 발견되면 클러스터 전체의 보안이 위험해질 수 있다.
- 아래와 같은 두 단계의 보안이 중요하다.

  - **이미지 보안:** 컨테이너를 실행하기 전, 안전하고 검증된 이미지를 사용하는 단계
  - **런타임 보안:** 컨테이너가 실행되는 중, 최소한의 권한으로 안전하게 동작하도록 강제하는 단계 → **`Security Contexts`**


## 이미지 보안 (Image Security)
신뢰할 수 없는 이미지를 사용하면 클러스터 전체가 위험에 처할 수 있기 때문에, 아래 세 가지 원칙이 중요하다.


1.  **신뢰할 수 있는 Base Image 사용**

      - 불필요한 라이브러리나 셸이 포함되지 않은 최소한의 이미지를 사용하여 공격 표면(Attack Surface)을 줄이는 것
      - 예: `distroless`, `alpine`

2.  **Private Registry 사용**

      - Docker Hub 같은 Public Registry 대신, 접근 제어가 가능한 Private Registry(Harbor, Docker Registry, GCR, ECR 등)를 사용하여 허가되고 검증된 이미지만을 클러스터 내에서 사용하도록 강제하기

3.  **이미지 취약점 스캔**

      - 이미지에 알려진 보안 취약점(CVE)이 있는지 지속적으로 스캔하고 관리해야 합니다.
      - 예: `Trivy`, `Clair`, `Snyk` 같은 오픈 소스 도구들을 CI/CD 파이프라인에 통합하여 자동 스캔을 수행할 수 있다.
      - 데브옵스가 활성화된 기업들은 이런 오픈소스들을 적극적으로 도입하기도 한다.

### 실습 포인트 - 1 (Private Registry 이미지 사용)

  - **시나리오:** 인증이 필요한 Private Registry에 있는 이미지를 Pod에서 사용하기

<!-- end list -->

1.  **`imagePullSecrets` 생성**

      - `docker-registry` 타입의 시크릿을 생성하여 Private Registry의 인증 정보를 저장합니다.

    <!-- end list -->

    ```bash
    kubectl create secret docker-registry private-reg-cred \
        --docker-server=<your-registry-server> \
        --docker-username=<your-username> \
        --docker-password=<your-password> \
        --docker-email=<your-email>
    ```

2.  **Pod 명세에 `imagePullSecrets` 지정**

      - Pod를 생성할 때, `spec.imagePullSecrets` 필드에 위에서 생성한 시크릿의 이름을 명시합니다.

    <!-- end list -->

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: private-pod
    spec:
      containers:
      - name: private-container
        image: <your-registry-server>/<your-private-image>:tag
      imagePullSecrets:
      - name: private-reg-cred
    ```

## 런타임 보안 - Security Contexts
`Pod` 또는 `Container` 수준에서 실행 권한과 관련된 보안 설정을 명시적으로 지정하는 것이다.


컨테이너가 필요 이상의 권한을 갖지 못하도록 제한하여, 해킹되더라도 피해를 최소화할 수 있다.


### 실습 포인트 - 2 (Security Context 적용)

  - **시나리오:** 컨테이너를 `root`가 아닌 일반 유저(UID 1000)로 실행하고, 권한 상승을 막으며, 루트 파일시스템을 읽기 전용으로 설정하기

<!-- end list -->

1.  **`securityContext`가 적용된 Pod YAML 작성**
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: secure-pod
    spec:
      securityContext:
        runAsUser: 1000       # Pod 내 모든 컨테이너는 UID 1000으로 실행
        runAsGroup: 3000      # Pod 내 모든 컨테이너는 GID 3000으로 실행
        fsGroup: 2000         # Volume 마운트 시 소유 그룹을 GID 2000으로 설정
      containers:
      - name: secure-container
        image: busybox:1.28
        command: [ "sh", "-c", "sleep 1h" ]
        securityContext:
          # Pod 수준의 설정을 덮어쓸 수 있음
          allowPrivilegeEscalation: false # 컨테이너가 부모 프로세스보다 더 많은 권한을 획득하는 것을 방지
          readOnlyRootFilesystem: true  # 컨테이너의 루트 파일시스템을 읽기 전용으로 설정
          capabilities:
            drop: ["ALL"] # 모든 Linux Capability를 제거
    ```
2.  **Pod 배포 및 보안 설정 확인**
    ```bash
    # Pod 배포
    kubectl apply -f secure-pod.yaml

    # Pod에 접속하여 현재 사용자 확인
    kubectl exec -it secure-pod -- whoami
    # 예상 출력: 1000 (또는 사용자 이름이 없는 경우 에러)

    # 루트 파일시스템에 파일 생성을 시도 (실패해야 함)
    kubectl exec -it secure-pod -- touch /testfile
    # 예상 출력: touch: /testfile: Read-only file system
    ```



---


# Network Security
- 쿠버네티스는 기본적으로 모든 Pod들이 서로 자유롭게 대화(통신)할 수 있는 '열린 광장'과도 같다. 
- 이 구조는 매우 편리하지만, 만약 하나의 Pod가 외부 공격에 노출된다면 해당 Pod를 발판 삼아 내부망에만 있어야 할 핵심 Pod(e.g. DB나 결제 로직을 담당하는 Pod)에 직접 접근할 수도 있다.
- 이를 막고자 Pod들 사이에 보이지 않는 '방화벽'을 세워 꼭 필요한 통신만 허용하는 것이 중요하고, 이것을 네트워크 정책(Network Policy)이 담당한다.


> **(중요) 사전 지식:** `NetworkPolicy` 리소스는 쿠버네티스 자체 기능이지만, 이를 실제로 구현하고 적용하는 것은 CNI(Container Network Interface) 플러그인의 역할이다. 
>
> 따라서 Calico, Cilium, Weave Net 등 `NetworkPolicy`를 지원하는 CNI가 클러스터에 설치되어 있어야 한다.


## Network Policy
기본적으로 'All Allow' 정책인 쿠버네티스 클러스터에서, 특정 Pod 그룹에 대한 Ingress(수신)와 Egress(송신) 트래픽 규칙을 정의하는 방화벽 역할을 한다.

### Network Policy 주요 구성요소
  - `podSelector`: 이 정책을 **어떤 Pod들에게 적용할지** 라벨로 선택
  - `policyTypes`: `Ingress`(수신), `Egress`(송신) 규칙 중 어떤 종류의 정책을 적용할지 지정
  - `ingress` / `egress` 규칙:
      - `from` (Ingress) / `to` (Egress): 트래픽을 허용할 **출처/목적지**를 정의
      - `podSelector`: 특정 라벨을 가진 Pod로부터의/로의 트래픽을 허용
      - `namespaceSelector`: 특정 라벨을 가진 네임스페이스의 모든 Pod로부터의/로의 트래픽을 허용
      - `ipBlock`: 특정 IP 대역(CIDR)으로부터의/로의 트래픽을 허용
      - `ports`: 허용할 포트와 프로토콜을 지정

### 실습 포인트 - 3 (Network Policy 적용하기)

  - **시나리오:** `role=db` 라벨을 가진 DB 파드는 `app=api` 라벨을 가진 API 파드로부터의 `TCP 6379` 포트 트래픽만 허용하고, 그 외의 모든 수신(Ingress) 트래픽은 차단하기

<!-- end list -->

1.  **테스트용 파드 배포**
    ```bash
    # DB 파드
    kubectl run db-pod --image=redis --labels=role=db --expose --port=6379
    # API 파드
    kubectl run api-pod --image=busybox:1.28 --labels=app=api -- sleep 3600
    # 외부 테스트용 파드
    kubectl run external-pod --image=busybox:1.28 --labels=app=external -- sleep 3600
    ```
2.  **(초기 상태) 통신 테스트**
      - `api-pod`와 `external-pod` 모두 `db-pod`에 접속이 가능해야 한다.
    <!-- end list -->
    ```bash
    # api-pod에서 db-pod로 접속 테스트 (성공해야 함)
    kubectl exec api-pod -- nc -zv db-pod 6379

    # external-pod에서 db-pod로 접속 테스트 (성공해야 함)
    kubectl exec external-pod -- nc -zv db-pod 6379
    ```
3.  **기본 Ingress Deny 정책 적용**
      - `role=db` 파드를 선택하고, `ingress` 규칙을 빈 값(`{}`)으로 두면 모든 수신 트래픽이 차단된다.
    <!-- end list -->
    ```yaml
    # db-deny-policy.yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: db-deny-ingress
    spec:
      podSelector:
        matchLabels:
          role: db
      policyTypes:
      - Ingress
      ingress: [] # 빈 Ingress 규칙은 모든 수신을 차단
    ```
    ```bash
    kubectl apply -f db-deny-policy.yaml

    # 이제 모든 접속이 실패해야 함
    kubectl exec api-pod -- nc -zv db-pod 6379
    kubectl exec external-pod -- nc -zv db-pod 6379
    ```
4.  **특정 트래픽만 허용하는 정책 적용**
      - 이제 `app=api` 파드로부터의 접속만 허용하는 규칙을 추가한다.
    <!-- end list -->
    ```yaml
    # api-allow-policy.yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: api-allow-to-db
    spec:
      podSelector:
        matchLabels:
          role: db # 이 정책은 db-pod에 적용
      policyTypes:
      - Ingress
      ingress:
      - from:
        - podSelector:
            matchLabels:
              app: api # app=api 라벨을 가진 파드로부터의 트래픽 허용
        ports:
        - protocol: TCP
          port: 6379 # 6379 TCP 포트로 오는 트래픽만 허용
    ```
    ```bash
    kubectl apply -f api-allow-policy.yaml

    # 최종 통신 테스트
    # api-pod에서 db-pod로 접속 테스트 (다시 성공해야 함)
    kubectl exec api-pod -- nc -zv db-pod 6379

    # external-pod에서 db-pod로 접속 테스트 (여전히 실패해야 함)
    kubectl exec external-pod -- nc -zv db-pod 6379
    ```
      - 위 테스트를 통해 `NetworkPolicy`가 의도대로 동작함을 확인할 수 있다.
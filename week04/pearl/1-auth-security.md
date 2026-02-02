# Chapter 07. Security

## Security Primitives

  - 호스트에 대한 모든 액세스는 보안을 거쳐야 한다.
      - 루트 액세스는 비활성화되어야 하며, 암호 기반 인증도 비활성화되어야 한다.
      - 오직 SSH Key를 기반으로 한 인증만 사용할 수 있다.
  - `kube-apiserver`는 쿠버네티스의 모든 동작의 중심으로, API에 직접 접근하거나 `kubectl`을 통해 상호작용하게 된다.
  - 즉, 첫 번째 방어선인 `kube-apiserver`에 대한 접근을 잘 제어해야 한다.
  - 아래 두 가지 질문에 대한 답이 쿠버네티스 보안의 시작이라고 할 수 있다.
      - **누가 클러스터에 접근할 수 있는가? (Who can access?) → `인증(Authentication)`**
      - **무엇을 할 수 있을까? (What can they do?) → `인가(Authorization)`**


### Who can access?
인증 메커니즘에 의해 결정된다. 아래에 해당하는 여러 가지 방법이 존재한다.

  - Files - Username and Passwords
  - Files - Username and Tokens
  - **Certificates (인증서):** 관리자, 노드(Kubelet) 등 핵심 컴포넌트와 사용자가 사용하는 가장 일반적인 방식.
  - External Authentication providers - LDAP, SAML 등
  - **Service Accounts:** 파드(Pod) 내부의 애플리케이션이 API 서버에 접근할 때 사용하는 방식.


### What can they do?

  - **RBAC Authorization (Role Based Access Control):** 현재 쿠버네티스 인가의 표준 방식.
  - ABAC Authorization (Attribute Based Access Control): RBAC 이전에 사용되던 방식으로, 정책 관리가 복잡하여 잘 사용되지 않음.
  - Node Authorization: 각 노드의 Kubelet이 수행할 수 있는 API 요청을 제한하는 특수 목적의 인가 방식.
  - Webhook Mode: 외부 서비스에 인가 결정을 위임하는 방식.
  - etcd 클러스터, kube-controller-manager, kube-scheduler, kube-apiserver 와 같은 다양한 구성 요소 사이의 모든 통신은 **TLS Encryption** 에 의해 보호된다.
  - 클러스터 내의 애플리케이션 간의 통신은 **Network Policy**에 의해 제한된다.


## 1\. API 접근 제어 (인증 -\> 인가 -\> Service Account)

### 인증 (Authentication)

  - 쿠버네티스는 자체적으로 사용자 정보를 관리하지 않는다. 사용자를 인증하는 다양한 “방법”만 제공한다.
  - 위에서 설명했던 Static File 종류들은 이제 deprecated 되어 잘 사용되지 않는다.
  - 주요 인증 방식은 클라이언트 인증서(Client Certificate)나 토큰(Token)을 이용한 방식이다.
      - **사용자 계정(User Account):** 주로 관리자나 개발자 같은 '사람'을 위한 계정. 외부 인증 시스템(LDAP, Keystone)이나 인증서를 통해 관리된다.
      - **서비스 어카운트(Service Account):** '애플리케이션(Pod)'을 위한 계정. 쿠버네티스 API가 직접 관리하며, 토큰을 사용한다.

### TLS Certification

  - 쿠버네티스 클러스터는 마스터 노드와 워커 노드로 구성되어 있다.
  - 노드들 간의 모든 통신은 보안이 필요하고 반드시 암호화되어야 하기 때문에 TLS 인증서가 중요하다.
  - **주요 TLS 암호화 통신 구간**
      - `kube-apiserver` ↔ `etcd` 클러스터
      - `kube-apiserver` ↔ `kube-controller-manager` / `kube-scheduler`
      - `kube-apiserver` ↔ `kubelet` (워커 노드)
      - 사용자(`kubectl`) ↔ `kube-apiserver`

### 실습 포인트 - 1 (Certificate Creation)

  - 인증서를 생성할 때, `openssl`을 사용하여 인증서 서명 요청(CSR)을 생성하고 CA(Certificate Authority)로 서명할 수 있다.
  - **시나리오:** 새로운 개발자 `dev-user`를 `development` 그룹에 소속시켜 인증서 발급하기

<!-- end list -->

1.  **개발자 개인 키(Private Key) 생성**
    ```bash
    openssl genrsa -out dev-user.key 2048
    ```
2.  **인증서 서명 요청(CSR) 생성**
      - `CN`: 사용자 이름 (Common Name)
      - `O`: 소속 그룹 (Organization) - 이 그룹 정보는 RBAC에서 사용
    <!-- end list -->
    ```bash
    openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user/O=development"
    ```
3.  **클러스터의 CA를 이용해 서명하여 인증서 발급** (관리자 권한 필요)
      - 클러스터의 CA는 보통 `/etc/kubernetes/pki/ca.crt` 와 `/etc/kubernetes/pki/ca.key` 에 위치
    <!-- end list -->
    ```bash
    openssl x509 -req -in dev-user.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out dev-user.crt -days 365
    ```

### 실습 포인트 - 2 (Certificate API)

  - 위와 같은 수동 방식 대신, 쿠버네티스 API를 통해 서명 요청을 보내고, 관리자가 이를 승인하는 자동화된 방식이다.
  - **시나리오:** `dev-user`가 생성한 CSR 파일을 관리자에게 전달하여 쿠버네티스를 통해 승인받기

<!-- end list -->

1.  **관리자가 `CertificateSigningRequest` 오브젝트 생성**
      - `dev-user.csr` 파일 내용을 `base64`로 인코딩하여 YAML 파일에 넣는다.
    <!-- end list -->
    ```bash
    cat dev-user.csr | base64 | tr -d "\n"
    ```
    ```yaml
    # dev-user-csr.yaml
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
      name: dev-user-csr
    spec:
      request: # 위에서 base64로 인코딩한 값을 여기에 붙여넣기
      signerName: kubernetes.io/kube-apiserver-client
      usages:
      - client auth
    ```
2.  **CSR 제출 및 승인**
    ```bash
    # CSR 제출
    kubectl apply -f dev-user-csr.yaml

    # 관리자가 CSR 목록 확인 (상태: Pending)
    kubectl get csr

    # 관리자가 CSR 승인
    kubectl certificate approve dev-user-csr

    # CSR 상태 확인 (상태: Approved, Issued)
    kubectl get csr
    ```

### KubeConfig 파일

![kubeconfig](/chapter-07/img/kubeconfig.png)

  - `kube-apiserver`에서 클라이언트가 자신을 증명하는 방법을 매번 명령줄에 입력하는 대신, 파일에 저장하여 편리하게 사용하는 방법이다.
  - **구조 (`clusters`, `users`, `contexts`)**
      - **`clusters`**: 접속하려는 쿠버네티스 클러스터의 API 서버 주소와 서버의 신뢰성을 검증하기 위한 CA 인증서 정보가 담겨있다.
      - **`users`**: 해당 클러스터에 접근하는 사용자 계정 정보. 위에서 만든 `dev-user.crt` (인증서)와 `dev-user.key` (개인키) 정보가 여기에 담긴다.
      - **`contexts`**: **어떤 `user`가 어떤 `cluster`에 접속할지**를 묶어놓은 '맥락' 정보. 네임스페이스를 지정할 수도 있다.
  - **관련 명령어**
      - `kubectl config view`: kubeconfig 구성 살펴보는 명령어
      - `kubectl config use-context <context-name>`: 기본으로 사용할 context를 변경하는 명령어
      - `kubectl config get-contexts`: 사용 가능한 모든 context 목록을 보는 명령어

### 인가 (Authorization)

  - 인증된 사용자가 클러스터에서 무엇을 할 수 있는가?
  - 위 인증 단계를 거쳐 사용자 신원(`dev-user`)이 확인된 후, 이 사용자가 어떤 리소스(Pod, Service 등)에 대해 어떤 작업(get, list, create, delete)을 수행할 수 있는지 허용(Allow) 또는 거부(Deny)하는 단계이다.

### 주요 인가 방식

  - **RBAC (Role-Based Access Control)**
  - **RBAC의 4가지 핵심 오브젝트**
      - **`Role`**: 특정 네임스페이스 내에서만 유효한 '권한 목록' (예: `development` 네임스페이스의 `pods`를 `get`, `list` 할 수 있는 권한)
      - **`ClusterRole`**: 클러스터 전체에 유효한 '권한 목록' (예: 모든 네임스페이스의 `pods`를 조회하거나, `nodes` 같은 클러스터 전역 리소스를 다룰 권한)
      - **`RoleBinding`**: `Role`을 특정 사용자/그룹/서비스어카운트에게 부여하는 '임명장'. 네임스페이스 내에서만 유효하다.
      - **`ClusterRoleBinding`**: `ClusterRole`을 특정 대상에게 부여한다. 클러스터 전체에 영향을 미친다.





### 실습 포인트 (RBAC)

  - **시나리오:** `development` 그룹에 속한 `dev-user`에게 `development` 네임스페이스에서 `Pod`는 모든 작업을 할 수 있지만, `Secret`은 조회만 가능하도록 권한 부여하기

  ![apigroups](/chapter-07/img/apigroups.png)
  

  - 위와 같은 Api Groups가 존재하는데, 각 파일에 어떤 api가 접근 가능한지 설정하는 것이라고 이해하면 된다.


<!-- end list -->

1.  **`Role` 생성: 필요한 권한 목록 정의**
    <!-- end list -->
    ```bash
    kubectl create role pod-secret-manager --verb=get,list,watch,create,delete --resource=pods -n development
    kubectl create role secret-viewer --verb=get,list --resource=secrets -n development

    # 위 두 role을 합친 role을 만들어도 된다.
    kubectl create role dev-role --verb=get,list,watch,create,delete --resource=pods -n development
    kubectl label role dev-role part-of-project=rbac-demo -n development # label 추가
    kubectl annotate role dev-role description="Developer role for pods and secrets" -n development # annotate 추가

    # 기존 Role에 규칙 추가
    kubectl edit role dev-role -n development
    ```
2.  **`RoleBinding` 생성: 사용자와 Role 연결**
    ```bash
    # `development` 그룹 전체에 `dev-role` 권한 부여
    kubectl create rolebinding dev-group-binding --role=dev-role --group=development -n development
    ```
3.  **권한 확인: `auth can-i` 사용**
      - `--as` 플래그를 사용하여 특정 사용자의 입장에서 권한을 테스트할 수 있다.
    <!-- end list -->
    ```bash
    # 허용된 작업: 성공 (yes)
    kubectl auth can-i create pods --as dev-user --as-group=development -n development

    # 거부된 작업: 실패 (no)
    kubectl auth can-i create secrets --as dev-user --as-group=development -n development

    # 허용된 작업: 성공 (yes)
    kubectl auth can-i get secrets --as dev-user --as-group=development -n development
    ```

### 서비스 어카운트 (Service Account)

  - **Pod를 위한 신분증**
  - 쿠버네티스 클러스터 내부에서 실행되는 애플리케이션(Pod)이 API 서버와 통신할 때 사용하는 ID이다.
  - **핵심:** 서비스 어카운트 역시 RBAC의 대상(Subject)이 될 수 있다. 
  - 즉, 우리가 만든 `Role`을 서비스 어카운트에 `RoleBinding`하여 Pod가 수행할 수 있는 작업을 정교하게 제어할 수 있는 것이다.
  - 아래는 파드 목록을 조회하는 권한을 서비스 어카운트로 제어하는 시나리오에 대한 흐름과 설명이다.

<!-- end list -->

1.  **서비스 어카운트 생성**
    ```bash
    kubectl create serviceaccount dashboard-sa -n monitoring
    ```
2.  **`ClusterRole` 생성 (모든 네임스페이스의 Pod 조회 권한)**
      - Pod는 네임스페이스에 속한 리소스지만, 모든 네임스페이스의 Pod를 보려면 `ClusterRole`이 필요하다.
    <!-- end list -->
    ```bash
    kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods
    ```
3.  **`ClusterRoleBinding` 생성: 서비스 어카운트와 ClusterRole 연결**
    ```bash
    # `monitoring` 네임스페이스의 `dashboard-sa` 서비스 어카운트에게 `pod-reader` 권한 부여
    kubectl create clusterrolebinding read-pods-global --clusterrole=pod-reader --serviceaccount=monitoring:dashboard-sa
    ```
4.  **Pod에서 서비스 어카운트 사용**
      - Pod 명세 파일에 `serviceAccountName`을 지정한다.
    <!-- end list -->
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: dashboard-pod
      namespace: monitoring
    spec:
      serviceAccountName: dashboard-sa
      containers:
      - name: dashboard
        image: some-dashboard-image
    ```
      - 이렇게 배포된 파드는 이제 클러스터의 모든 파드 목록을 조회할 권한을 가진다.
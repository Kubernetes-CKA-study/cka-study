# 1. Kubernetes Security Primitives

쿠버네티스는 기본적으로 개방형 구조를 가지고 있기 때문에 운영 환경에서는 반드시 보안 설정을 해야합니다. 쿠버네티스 보안의 핵심이 되는 5가지 영역을 정리해 보겠습니다. 

## 1. 호스트 보안 (Host Security)

가장 밑단인 물리 서버(또는 VM) 자체가 뚫리면 위의 모든 보안은 무용지물입니다.

- **루트 접근 금지 (No Root Access):** 모든 노드에 대한 Root 계정 접근을 비활성화해야 합니다.
- **패스워드 인증 금지:** SSH 접속 시 비밀번호 기반 인증을 막습니다.
- **SSH 키 사용:** 오직 **SSH Key** 기반의 인증만 허용하여 무차별 대입 공격(Brute Force Attack)을 방지해야 합니다.

## 2. API 서버 보안 (Authentication)

모든 명령의 관문인 `kube-apiserver`에 누가 접근할 수 있는지 제어하는 단계입니다. "당신은 누구입니까?"를 묻습니다.

- **사용자 인증 (User Auth):** 관리자나 개발자가 `kubectl`을 사용할 때.
    - 주로 **인증서(Certificates)** 방식 사용
    - 외부 인증 제공자와 연동합니다.
- **머신 인증 (Machine Auth):** 파드 내부의 애플리케이션이 API 서버에 접근할 때
    - Service Account를 사용하여 인증.

## 3. 권한 부여 (Authorization)

무엇을 할 수 있는지

- **RBAC (Role-Based Access Control)**
    - 사용자를 특정 그룹에 넣고 그 그룹에 특정 권한(Role)을 부여하는 방식입니다.
    - 예: "개발팀 그룹은 `default` 네임스페이스의 파드만 조회(List)할 수 있다."
- 이 외에도 ABAC, Webhook 등 다양한 모듈이 있음

## 4. 통신 보안 (Communication Security)

클러스터 내부의 컴포넌트들(Scheduler, Controller Manager, Kubelet 등)끼리 주고받는 데이터는 어떻게 보호될까요?

- **TLS 암호화:** 모든 통신은 **TLS(SSL)** 인증서를 통해 암호화되어야 합니다.
- etcd, API Server, Kubelet 등 모든 구성 요소 간의 트래픽은 평문으로 전송되지 않습니다.

## 5. 네트워크 보안 (Network Security)

기본적으로 쿠버네티스 클러스터 내의 모든 파드는 서로 통신이 가능(All Allow)하도록 설계되어 있습니다. 이는 보안상 큰 위협이 될 수 있습니다.

- **네트워크 정책 (Network Policies):** 파드 간의 방화벽 역할을 합니다.
- 특정 파드가 다른 특정 파드하고만 통신하도록 제한할 수 있습니다.
    - 예: "DB 파드는 오직 백엔드 API 파드에서 오는 트래픽만 허용한다."

# 2. Authentication

인증(Authentication)은 "당신은 누구입니까?"를 확인하는 과정입니다.

## 1. 쿠버네티스의 사용자(User) 구분

쿠버네티스 클러스터에 접근하는 주체는 크게 두 가지로 나뉩니다.

### 1) 일반 사용자 (Normal Users)

- **대상:** 관리자(Admin), 개발자(Developer) 등 **사람**.
- **특징:** 쿠버네티스는 **일반 사용자를 관리하는 별도의 오브젝트(User Object)가 없습니다.**
    - 외부의 관리 체계(Google 계정, LDAP, 인증서 등)에 의존합니다.
    - 즉, 쿠버네티스는 "이 인증서를 가진 사람은 'dev-user'야"라고 인식만 할 뿐 직접 계정을 생성하거나 삭제하지 않습니다.

### 2) 서비스 어카운트 (Service Accounts)

- **대상:** 파드(Pod) 내부의 애플리케이션, 봇(Bot) 등 **기계(Machine)**.
- **특징:** 쿠버네티스가 **직접 관리**하는 사용자입니다.
    - `kubectl create serviceaccount` 명령어로 생성할 수 있습니다.
    - 주로 파드가 API 서버와 통신할 때 사용됩니다.

## 2. 인증 메커니즘 (Authentication Mechanisms)

쿠버네티스 API 서버는 요청이 들어오면 다양한 방법으로 "이 요청을 보낸 사람이 누구인지" 확인합니다.

### 1) 정적 패스워드 파일 (Static Password File)

가장 단순하고 원시적인 방법입니다. 사용자 정보를 담은 CSV 파일을 API 서버에 등록합니다.

- **파일 형식 (`user-details.csv`):**
    
    ```
    password123,user1,u0001,group1
    # (비밀번호, 사용자명, 사용자ID, 그룹)
    ```
    
- **설정:** `kube-apiserver` 실행 옵션에 `-basic-auth-file=user-details.csv`를 추가하고 재시작합니다.
- **단점:** 비밀번호가 평문(Clear Text)으로 저장되므로 보안에 매우 취약합니다.

### 2) 정적 토큰 파일 (Static Token File)

패스워드 대신 토큰을 사용하는 방식입니다.

- **파일 형식 (`user-tokens.csv`):**
    
    ```
    K8sToken123,user1,u0001,group1
    # (토큰, 사용자명, 사용자ID, 그룹)
    ```
    
- **설정:** `kube-apiserver` 실행 옵션에 `-token-auth-file=user-tokens.csv`를 추가합니다.
- **단점:** 역시 토큰 정보가 파일에 평문으로 저장되므로 보안상 권장되지 않습니다.

### 3) 인증서 (Certificates) - ★ 권장 !

운영 환경에서 가장 많이 사용되는 방식입니다.
사용자는 신뢰할 수 있는 CA(Certificate Authority)가 서명한 클라이언트 인증서(Client Certificate)를 제출하여 신원을 증명합니다.

- HTTPS 보안 통신과 유사한 방식입니다.
- 쿠버네티스 클러스터 구축 시 기본적으로 설정되는 방식입니다. (`kubeconfig` 파일 안에 인증서 정보가 들어있습니다.)

### 4) 외부 인증 제공자 (Identity Providers)

LDAP, Kerberos, OpenID Connect(Google, AWS IAM 등)와 같은 제3자 인증 시스템과 연동합니다. 

# 3. TLS 인증 기초

본격적으로 쿠버네티스 인증서를 생성하기 전에 **TLS의 기본 개념**을 정리해보겠습니다. 

## 1. TLS란 무엇인가?

TLS(Transport Layer Security)는 인터넷상에서 정보를 안전하게 주고받기 위한 **암호화 통신 프로토콜**입니다. 쿠버네티스 클러스터 내부의 모든 컴포넌트(API Server, Kubelet, Scheduler 등)는 서로 통신할 때 그냥 데이터를 보내지 않고 **반드시 TLS로 암호화된 통신**을 합니다.

## 2. 암호화의 두 가지 방식

TLS를 이해하려면 두 가지 암호화 방식을 알아야 합니다.

### 1) 대칭키 암호화 (Symmetric Encryption)

- **개념:** 암호화할 때 쓰는 키와 복호화할 때 쓰는 키가 **같습니다**.
- **비유:** 우리 집에 들어올 수 있는 '열쇠'를 복사해서 가족들에게 나눠주는 것.
- **문제점:** 이 열쇠를 도둑(해커)에게 뺏기면 끝장납니다. 인터넷상에서 키를 안전하게 전달하기 어렵습니다.

### 2) 비대칭키 암호화 (Asymmetric Encryption) - ★

- **개념:** 키가 두 개입니다. 개인키(Private Key)와 **공개키(Public Key)**.
    - **공개키(Public Key):** 자물쇠입니다. 누구나 가질 수 있습니다. 이걸로 데이터를 잠금(암호화)니다.
    - **개인키(Private Key):** 열쇠입니다. 나만 가지고 있습니다. 이걸로 잠긴 데이터를 엽(복호화)니다.
- **장점:** 공개키는 남들에게 줘도 상관없습니다. 내 개인키만 잘 지키면 됩니다.

> 쿠버네티스는 이 비대칭키 방식을 사용하여 안전한 통신 채널을 만듭니다.
> 

## 3. 인증서(Certificate)와 CA(Certificate Authority)

키만 있다고 끝이 아닙니다. "이 공개키가 진짜 네 것이 맞느냐?"를 보증해 줄 신뢰할 수 있는 제3자가 필요합니다.

### 1) 인증서 (Certificate) = 신분증

- 내 공개키와 내 정보(이름, 도메인 등)가 적혀 있는 디지털 문서입니다.
- 예: `server.crt`, `client.crt`

### 2) CA (Certificate Authority) = 발급기관

- 신분증(인증서)이 진짜라고 도장을 찍어주는(서명하는) 기관입니다.
- 쿠버네티스 클러스터에는 자체적인 **Root CA**가 존재합니다.

## 4. 쿠버네티스에서의 서버 vs 클라이언트 인증서

쿠버네티스 보안이 복잡한 이유는 **하나의 컴포넌트가 때로는 서버가 되고 때로는 클라이언트가 되기 때문**입니다. 이를 위해 **서버 인증서**와 **클라이언트 인증서**를 구분해서 이해해야 합니다.

### 1) 서버 인증서 (Server Certificates)

> "나는 서버야. 안심하고 접속해." (접속을 받는 입장)
> 
- **API Server:** 클러스터의 중심입니다. 관리자(kubectl), 스케줄러, Kubelet 등의 접속을 받기 위해 서버 인증서가 필요합니다. (`apiserver.crt`)
- **etcd Server:** API 서버로부터 데이터를 저장하라는 요청을 받기 위해 서버 인증서가 필요합니다. (`etcdserver.crt`)
- **Kubelet:** API 서버가 파드 로그를 보거나 상태를 체크하러 들어올 때 자신의 신원을 증명하기 위해 서버 인증서가 필요합니다. (`kubelet.crt`)

### 2) 클라이언트 인증서 (Client Certificates)

> "나는 허가받은 사용자(클라이언트)야. 문 열어줘." (접속을 하는 입장)
> 
- **Admin (kubectl):** API 서버에 명령을 내리기 위해 클라이언트 인증서를 제출합니다. (`admin.crt`)
- **Scheduler & Controller Manager:** API 서버와 통신하여 파드를 스케줄링하거나 관리하기 위해 클라이언트 인증서를 사용합니다. (`scheduler.crt`, `controller-manager.crt`)
- **Kubelet:** 노드 상태를 API 서버에 보고(Report)할 때는 클라이언트 입장이므로 클라이언트 인증서가 필요합니다. (Kubelet은 서버/클라이언트 인증서가 둘 다 필요합니다!)
- **API Server:** etcd에 데이터를 쓸 때는 API 서버가 '클라이언트'가 됩니다. 이때 etcd에게 제출할 클라이언트 인증서가 필요합니다. (`apiserver-etcd-client.crt`)

## 5. 컴포넌트별 흐름

모든 통신은 **신뢰할 수 있는 CA**가 서명한 인증서를 기반으로 이루어집니다.

1. **kubectl (Client)** ➡ **API Server (Server)**
2. **API Server (Client)** ➡ **etcd (Server)**
3. **API Server (Client)** ↔ **Kubelet (Server)**
4. **Scheduler (Client)** ➡ **API Server (Server)**

# 4. TLS 인증서 생성 및 관리

**OpenSSL** 도구를 사용하여 쿠버네티스 클러스터 구축에 필요한 인증서들을 직접 생성해 보겠습니다. 이 과정은 "누가(Client/Server) 쓸 것인가?"에 따라 인증서 종류가 달라지며 "누가(CA) 서명할 것인가?"가 핵심입니다.

## 1. 인증서 생성 3단계

모든 인증서 생성 과정은 다음 3단계를 따릅니다.

1. **개인키 생성 (Generate Key):** `.key` 파일 생성
    - 나만 가지고 있어야 하는 비밀 열쇠입니다.
2. **인증서 서명 요청 (Create CSR - Certificate Signing Request):** `.csr` 파일 생성
    - 내 정보(이름, IP, 도메인 등)가 담긴 신청서입니다.
    - 여기에 개인키가 사용됩니다.
3. **인증서 발급 (Sign Certificate):** `.crt` 파일 생성
    - CA(인증 기관)가 CSR을 검토하고 자신의 개인키로 서명(Sign)하여 최종 신분증을 발급합니다.

## 2. CA (Certificate Authority) 만들기

클러스터 내의 모든 인증서를 보증해 줄 '인증 기관'부터 만들어야 합니다.

### 1) CA 개인키 생성

```
openssl genrsa -out ca.key 2048
```

### 2) CA 인증서 생성 (Self-Signed)

CA는 자기 자신을 보증하므로 CSR 생성과 서명을 한 번에 합니다.

```
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

- `ca.key` (절대 유출 금지), `ca.crt` (모든 컴포넌트에 배포)

## 3. 클라이언트 인증서 만들기: 관리자(Admin)용

관리자가 `kubectl`을 사용하기 위한 인증서를 만들어 봅시다.

### 1) Admin 개인키 생성

```
openssl genrsa -out admin.key 2048
```

### 2) Admin CSR 생성 (중요!)

여기서 **OU(Group)** 설정이 매우 중요합니다. 쿠버네티스 권한 관리(RBAC)와 연결되기 때문입니다.

```
# CN: 사용자 이름 (admin)
# O: 그룹 (system:masters -> 관리자 그룹)
openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
```

### 3) CA로 서명하여 인증서 발급

```
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out admin.crt -days 365
```

- **결과물:** `admin.crt`, `admin.key`

## 4. 클라이언트 인증서 만들기: **시스템 컴포넌트용**

쿠버네티스의 시스템 컴포넌트들은 **반드시 `system:` 접두어가 붙은 약속된 이름**을 사용해야 올바른 권한을 가질 수 있습니다.

### 주요 시스템 컴포넌트 이름 규칙

이 규칙을 지켜서 CSR을 생성해야 합니다.

1. **Kube-Scheduler**
    - **CN (Common Name):** `system:kube-scheduler`
    - **예시:** `openssl req ... -subj "/CN=system:kube-scheduler"`
2. **Kube-Controller-Manager**
    - **CN:** `system:kube-controller-manager`
    - **예시:** `openssl req ... -subj "/CN=system:kube-controller-manager"`
3. **Kube-Proxy**
    - **CN:** `system:kube-proxy`
    - **O (Organization/Group):** `system:node-proxies`
    - **예시:** `openssl req ... -subj "/CN=system:kube-proxy/O=system:node-proxies"`
4. **Kubelet (각 노드별 생성)**
    - **CN:** `system:node:<node-name>` (예: `system:node:node01`)
    - **O (Group):** `system:nodes`
    - **생성 예시:** `openssl req ... -subj "/CN=system:node:node01/O=system:nodes"`

> system: 키워드를 빼먹으면 단순 사용자로 인식되어 핵심 기능을 수행할 권한이 부여되지 않습니다.
> 

## 5. etcd 인증서 만들기 (Server & Peer)

etcd는 클러스터의 핵심 저장소. API 서버와 통신하거나 etcd 노드끼리 통신할 때 인증서가 필요합니다. (고가용성 클러스터의 경우)

### 1) etcd 설정 파일 (openssl-etcd.cnf) 준비

etcd도 서버 역할을 하므로 SAN(Subject Alternative Names) 설정이 필요합니다. etcd가 실행될 호스트의 IP를 포함해야 합니다.

```
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = 127.0.0.1
IP.2 = 10.96.0.100  # etcd가 실행되는 서버 IP
DNS.1 = localhost
```

### 2) etcd 서버 인증서 생성

```
# etcd-server.key 생성
openssl genrsa -out etcd-server.key 2048

# etcd-server.csr 생성
openssl req -new -key etcd-server.key -subj "/CN=etcd-server" -config openssl-etcd.cnf -out etcd-server.csr

# etcd-server.crt 발급
openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out etcd-server.crt -extensions v3_req -extfile openssl-etcd.cnf -days 365
```

### 3) etcd Peer 인증서 (멀티 노드 구성 시)

etcd 노드가 여러 개인 경우 노드끼리 통신하기 위한 **Peer 인증서**도 유사한 방식으로 생성해야 합니다.(`etcd-peer.key`, `etcd-peer.crt`)

## 6. Kube-API Server용

API 서버는 클러스터의 중심으로 다양한 이름(IP, 도메인)으로 불립니다. 따라서 인증서에 SAN(Subject Alternative Names)을 반드시 설정해야 합니다.

### 1) OpenSSL 설정 파일 (openssl.cnf) 준비

```
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```

### 2) 키 및 CSR 생성

```
openssl genrsa -out apiserver.key 2048
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -config openssl.cnf -out apiserver.csr
```

### 3) CA로 서명

```
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out apiserver.crt -extensions v3_req -extfile openssl.cnf -days 365
```

## 7. 인증서 적용 및 확인 (Configuration & Check)

### 인증서 내용 확인

인증서가 제대로 만들어졌는지 만료일은 언제인지, SAN은 잘 들어갔는지 확인할 때 사용합니다.

```
openssl x509 -in apiserver.crt -text -noout
```

### 컴포넌트에 적용

생성된 인증서 파일들을 각 컴포넌트의 실행 옵션이나 설정 파일에 등록해야 합니다.

**예: kube-apiserver 실행 옵션**

```
--client-ca-file=/var/lib/kubernetes/ca.crt
--tls-cert-file=/var/lib/kubernetes/apiserver.crt
--tls-private-key-file=/var/lib/kubernetes/apiserver.key
```

## 8. Kubelet

Kubelet은 API 서버로부터 들어오는 요청(로그 확인, exec 등)을 처리하기 위해 **서버로서의 역할**도 수행합니다. 따라서 HTTPS 통신을 위한 서버 인증서가 필요합니다.

> ⚠️ (Naming Convention):
Kubelet 인증서를 생성할 때 가장 중요한 것은 이름 규칙입니다. Kubelet이 올바른 권한을 얻으려면 CN(Common Name)은 반드시 system:node:<node-name> 형식이어야 하고, O(Group)는 system:nodes여야 합니다. 이 규칙을 지키지 않으면 API 서버가 해당 요청을 노드에서 온 것으로 인식하지 못해 권한 에러가 발생합니다.
> 

### 1) Kubelet 서버 설정 파일 (openssl-kubelet.cnf) 준비

각 워커 노드의 IP와 호스트 이름을 SAN에 포함해야 합니다.

```
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = node01      # 노드 호스트 이름
IP.1 = 10.96.0.101  # 노드 IP 주소
```

### 2) Kubelet 서버 인증서 생성

```
# kubelet.key 생성
openssl genrsa -out kubelet.key 2048

# kubelet.csr 생성
openssl req -new -key kubelet.key -subj "/CN=node01" -config openssl-kubelet.cnf -out kubelet.csr

# kubelet.crt 발급
openssl x509 -req -in kubelet.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kubelet.crt -extensions v3_req -extfile openssl-kubelet.cnf -days 365
```

> **참고:** 위와 같이 생성하면 하나의 인증서로 클라이언트 역할(권한 획득)과 서버 역할(SAN을 통한 IP 식별)을 모두 수행할 수 있습니다.
> 

## 9.인증서 상세 정보 확인 (View Certificate Details)

인증서 설정 후 컴포넌트가 동작하지 않는다면, 가장 먼저 **로그**를 확인해야 합니다.

**확인해야 할 4가지 체크리스트:**

1. **Issuer (발급자):** `CN=KUBERNETES-CA` 처럼 우리가 신뢰하는 CA가 서명했는가? (신뢰성 검증)
2. **Subject (주체):** `CN`과 `O`가 의도한 권한 그룹 및 이름과 일치하는가? (권한 검증)
3. **Validity (유효 기간):** `Not After` 날짜가 지나지 않았는가? (만료 여부 확인)
4. **Subject Alternative Name (SAN):** API 서버나 노드의 IP/도메인이 모두 포함되어 있는가? (연결 거부의 주원인)

### 1) 파드 로그 확인 (kubectl logs)

```
# 형식: kubectl logs -n <네임스페이스> <파드이름>
kubectl logs -n kube-system kube-apiserver-master
```

### 2) kubectl 명령어가 안 먹힐 때 (Docker/Crictl)

**만약 인증서 문제로 API 서버나 etcd가 아예 시작되지 않았다면, `kubectl` 명령어 자체가 실패합니다.** 이때는 해당 노드(마스터 노드)에 직접 SSH로 접속하여 **컨테이너 런타임 레벨**에서 로그를 확인해야 합니다.

- **Docker를 사용하는 경우:**
    
    ```
    # 1. 정지된(Exited) 컨테이너까지 포함하여 조회 (실패해서 죽었을 테니까요!)
    docker ps -a | grep kube-apiserver
    
    # 2. 컨테이너 ID로 로그 확인
    docker logs <container-id>
    ```
    
- **Containerd (crictl)를 사용하는 경우:**
    
    ```
    crictl ps -a | grep kube-apiserver
    crictl logs <container-id>
    ```
    

### 3) 로그에서 자주 보이는 에러 패턴

- **`x509: certificate signed by unknown authority`**: 클라이언트가 가진 CA 인증서와 서버 인증서를 서명한 CA가 다를 때 발생합니다.
- **`x509: certificate has expired or is not yet valid`**: 인증서 유효 기간이 지났거나, 서버 시간(NTP)이 맞지 않을 때 발생합니다.
- **`x509: cannot validate certificate for <IP> because it doesn't contain any IP SANs`**: 접속하려는 IP 주소가 서버 인증서의 **SAN 목록**에 없을 때 발생합니다.

## 10. Certificates API

사용자가 CSR을 API 서버로 보내면 관리자가 `kubectl` 명령어로 승인해 주는 방식입니다.

### 1) 사용자의 키 및 CSR 생성 (로컬 작업)

사용자는 자신의 로컬 컴퓨터에서 키와 CSR을 생성합니다.

```
openssl genrsa -out jane.key 2048
openssl req -new -key (사용자).key -subj "/CN=사용자" -out 사용자.csr
```

### 2) CertificateSigningRequest (CSR) 오브젝트 생성

생성된 `jane.csr` 파일의 내용을 **인코딩**하여 YAML 파일에 넣습니다.

```
cat jane.csr | base64 | tr -d "\n"
```

**사용자`-csr.yaml` 작성:**

```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: 사용자
spec:
  groups:
  - system:authenticated
  request: <Base64_Encoded_CSR_Content>  # 위에서 인코딩한 값 붙여넣기
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

### 3) CSR 등록 및 승인 (관리자 작업)

관리자는 `kubectl`로 요청을 확인하고 승인합니다.

```
# 1. 요청 확인
kubectl get csr

# 2. 요청 승인 (Approve)
kubectl certificate approve 사용자

# 3. 승인된 인증서 확인 및 추출 (Decode)
kubectl get csr jane -o yaml
# status.certificate 필드의 값을 복사하여 Base64 디코딩하면 사용자.crt가 됩니다.
kubectl get csr 사용자 -o jsonpath='{.status.certificate}' | base64 --decode > 사용자.crt
```

### 4) 누가 서명해 주는가? (Architecture)

이 과정에서 실제 서명 작업은 **Kube-Controller-Manager**가 담당합니다.
컨트롤러 매니저 설정 파일에 CA 키와 인증서 경로가 지정되어 있기 때문에 API 요청이 들어오면 자동으로 서명하여 발급해 줍니다.

## 11.  KubeConfig

인증서(`admin.crt`, `admin.key` 등)를 생성한 후 매번 `kubectl` 명령어 옵션으로 인증서 경로를 지정하는 것은 매우 번거롭습니다. 이 정보를 **`kubeconfig`** 파일(기본 경로: `~/.kube/config`)에 저장하면 손쉽게 클러스터에 접근할 수 있습니다.

### 1) Kubeconfig

Kubeconfig 파일은 크게 다음 세 가지 섹션으로 구성됩니다.

1. **Clusters (어디로?):** 접속할 쿠버네티스 API 서버의 정보 (URL, CA 인증서)
2. **Users (누구?):** 접속하는 사용자의 자격 증명 정보 (클라이언트 인증서, 키)
3. **Contexts (연결):** "이 사용자(User)가 저 클러스터(Cluster)에 접속한다"는 매핑 정보

### 2) Kubeconfig 파일 구조 예시 (YAML)

앞서 생성한 `ca.crt`, `admin.crt`, `admin.key`를 사용하여 구성한 예시입니다.

```
apiVersion: v1
kind: Config
current-context: my-admin-context  # 현재 기본으로 사용할 컨텍스트

clusters:
- name: my-cluster
  cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://my-cluster-control-plane:6443

users:
- name: my-admin
  user:
    client-certificate: /etc/kubernetes/pki/admin.crt
    client-key: /etc/kubernetes/pki/admin.key

contexts:
- name: my-admin-context
  context:
    cluster: my-cluster
    user: my-admin
```

### 3) 주요 명령어 (kubectl config)

파일을 직접 수정할 수도 있지만 `kubectl config` 명령어를 사용하면 안전하게 관리할 수 있습니다.

- **설정 확인:** `kubectl config view`
- **컨텍스트 변경:** `kubectl config use-context <context-name>`
- **현재 컨텍스트 확인:** `kubectl config current-context`

# 3. API Groups

Kubeconfig를 통해 `kubectl`이 API 서버에 접속하면 실제로 어떤 일이 일어날까요?!
우리가 무심코 쓰는 `kubectl get pod` 명령어는 사실 API 서버의 특정 URL 경로(Path)로 HTTP 요청을 보내는 것입니다.

## 1. API 그룹 (API Groups)

쿠버네티스 API는 기능별로 수백 개의 리소스를 관리하기 위해 그룹(Group)으로 나뉘어 있습니다.
가장 중요한 두 가지 대분류는 **Core(Legacy) Group**과 **Named Group**입니다.

### 1) Core API Group (Legacy)

쿠버네티스의 가장 기본적이고 핵심적인 리소스들이 모여 있는 그룹입니다.

- **URL 경로:** `/api`
- **apiVersion:** `v1` (YAML 파일에서 `apiVersion: v1`이라고 쓰는 애들)
- **포함 리소스:**
    - **Namespaces**
    - **Pods**
    - **Nodes**
    - **Services**
    - **Events**
    - **ConfigMaps**, **Secrets** 등

### 2) Named API Group (신규 그룹)

Core 그룹 이후에 추가된 기능들은 모두 기능별로 이름을 가진 그룹으로 분류됩니다.

- **URL 경로:** `/apis` (뒤에 's'가 붙습니다!)
- **구조:** `/apis/<그룹이름>/<버전>`
- **포함 리소스 (예시):**
    - **`/apis/apps/v1`**: Deployments, ReplicaSets, DaemonSets, StatefulSets
    - **`/apis/batch/v1`**: Jobs, CronJobs
    - **`/apis/networking.k8s.io/v1`**: NetworkPolicies, Ingress
    - **`/apis/rbac.authorization.k8s.io/v1`**: Roles, RoleBindings
    - **`/apis/certificates.k8s.io/v1`**: CertificatesSigningRequests (CSR)

> 특징: YAML 파일에서 apiVersion: apps/v1 처럼 그룹/버전 형식을 사용합니다.
> 

## 2. API 버전 (API Versioning)

쿠버네티스는 API의 안정성을 보장하기 위해 버전을 명시합니다.

- **v1 (GA, Stable):** 안정적인 버전. 운영 환경 사용 권장.
- **v1beta1 (Beta):** 충분히 테스트되었으나 향후 변경 가능성 있음.
- **v1alpha1 (Alpha):** 개발 초기 단계. 버그가 있을 수 있으며 기능이 사라질 수도 있음.

## 3. Practical Applications

`kubectl` 없이 API 구조를 직접 확인하는 방법입니다.

### 1) kubectl proxy 사용

인증(Authentication) 문제를 쉽게 해결하기 위해 프록시를 띄웁니다.

```
kubectl proxy
# Starting to serve on 127.0.0.1:8001
```

### 2) curl로 API 탐색

새 터미널을 열어 로컬호스트로 요청을 보냅니다.

**Core Group 확인 (`/api`)**

```
curl http://127.0.0.1:8001/api/v1/pods
# 클러스터 내의 모든 파드 정보가 JSON으로 출력됨
```

**Named Group 확인 (`/apis`)**

```
curl http://127.0.0.1:8001/apis/apps/v1/deployments
# 클러스터 내의 모든 디플로이먼트 정보 출력
```

# 4. 인가(Authorization)

이전 단계인 인증(Authentication)을 통해 "당신이 누구인지"는 증명되었습니다.
하지만, 회사 건물에 들어왔다고 해서 사장실이나 서버실에 마음대로 들어갈 수 있는 건 아닙니다.

이제 **인가(Authorization)** 단계에서 "당신이 어떤 리소스에 접근하고, 어떤 행동을 할 수 있는지"를 결정해야 합니다. 

## 1. 인가(Authorization) 메커니즘 개요

쿠버네티스 API 서버는 요청이 들어오면 다음 순서로 처리합니다.

1. **Authentication (인증):** 신원 확인 (User ID, Group 등 식별)
2. **Authorization (인가):** 권한 확인 (Allow or Deny)
3. **Admission Control:** 심화 검증 및 변형

인가 단계에서 권한이 없으면 `403 Forbidden` 에러를 반환하고 요청을 거부합니다.

## 2. 주요 인가 모드 (Authorization Modes)

쿠버네티스는 다양한 인가 방식을 지원합니다. 관리자는 API 서버 설정 시 이 중 하나 이상을 선택하여 사용할 수 있습니다.

### 1) Node Authorization

- **대상:** **Kubelet** (노드)
- **목적:** Kubelet이 API 서버에 보고(Report)하거나 파드 정보를 읽을 때 필요한 권한을 부여합니다.
- **특징:** `system:nodes` 그룹에 속한 사용자(Kubelet)의 요청을 전담하여 처리합니다.

### 2) ABAC (Attribute-based Access Control) - 속성 기반

- **방식:** "누가(User) 무엇(Resource)을 할 수 있다"는 정책을 JSON 파일로 정의합니다.
- **단점:** 정책 파일(`policy.json`)을 수정할 때마다 **API 서버를 재시작**해야 적용됩니다. 그래서 관리가 어렵고 유연하지 않습니다.

### 3) RBAC (Role-Based Access Control) - 역할 기반 (★ 표준)

- **방식:** 역할(Role)을 만들고 사용자에게 그 역할을 부여(Binding)하는 방식입니다.
- **장점:**
    - API 서버 재시작 없이 `kubectl` 명령어로 실시간 정책 변경이 가능합니다.
    - 네임스페이스별로 세밀한 권한 제어가 가능합니다.
- **활용:** 앞서 배운 **API Groups** 개념이 여기서 사용됩니다.

### 4) Webhook

- **방식:** 권한 확인을 외부 서비스(External Service)에 위임합니다.
- **활용:** **OPA (Open Policy Agent)** 같은 전문 정책 엔진을 사용하여 복잡한 규칙을 적용할 때 사용합니다.
    - 예: "주말에는 배포 금지", "특정 레지스트리의 이미지만 허용" 등

### 5) AlwaysAllow / AlwaysDeny

- **AlwaysAllow:** 모든 요청 허용 (보안 없음, 테스트용)
- **AlwaysDeny:** 모든 요청 거부 (사용 불가)

## 3. 인가 모드 설정 방법

API 서버(`kube-apiserver`)의 실행 옵션 `--authorization-mode`를 통해 설정합니다.
여러 모드를 동시에 지정할 수 있으며 순서(Order)가 중요합니다.

```
--authorization-mode=Node,RBAC,Webhook
```

### 동작 원리

지정된 순서대로 권한을 확인합니다.

1. **Node Authorizer:** 요청이 Kubelet에서 왔는가? -> 맞으면 처리, 아니면 다음으로 Toss.
2. **RBAC Authorizer:** 요청에 맞는 Role이 있는가? -> 맞으면 허용(Allow), 없으면 다음으로 Toss.
3. **Webhook Authorizer:** 외부 시스템에 물어봄 -> 허용하면 Allow.
4. **끝까지 허용되지 않음:** 최종적으로 **거부(Deny)**.

## 4. RBAC

RBAC은 크게 "어떤 권한인가(Role)"와 "누구에게 줄 것인가(Binding)"로 구성됩니다.
그리고 그 범위가 "특정 네임스페이스(Namespace)"인지 "클러스터 전체(Cluster)"인지에 따라 종류가 나뉩니다.

### 1) 네임스페이스 범위: Role & RoleBinding

특정 네임스페이스(예: `default`) 안에서만 유효한 권한입니다.

- **Role:** 권한을 정의합니다. (예: 파드를 볼 수 있음)
- **RoleBinding:** 역할을 사용자에게 연결합니다. (예: `jane`에게 파드 보는 역할을 줌)

**YAML 예시:**

```
# 1. Role 생성
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: developer-role
rules:
- apiGroups: [""] # Core Group
  resources: ["pods"]
  verbs: ["get", "list", "create"]

---
# 2. RoleBinding 생성
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: default
subjects:
- kind: User
  name: jane # 대상 사용자
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role # 연결할 Role 이름
  apiGroup: rbac.authorization.k8s.io

```

### 2) 클러스터 전체 범위: ClusterRole & ClusterRoleBinding

**ClusterRole**은 네임스페이스에 속하지 않는(Non-namespaced) 리소스입니다.

**ClusterRole이 필요한 두 가지 경우:**

1. **클러스터 범위 리소스 제어:** 네임스페이스가 없는 리소스(Nodes, PV, Namespaces 등)를 제어할 때.
2. **공통 권한 정의 및 전역 적용:** "모든 네임스페이스의 파드를 볼 수 있는 모니터링 계정"이나 "CI/CD 파이프라인"처럼 클러스터 전체에 걸쳐 동일한 권한이 필요할 때.

**YAML 예시 (클러스터 관리자 권한):**

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-role
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
- kind: User
  name: admin-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin-role
  apiGroup: rbac.authorization.k8s.io

```

> ClusterRole을 만들어두고 특정 네임스페이스 안에 RoleBinding으로 연결할 수도 있습니다.
이렇게 하면 "표준화된 권한(예: view)을 정의해두고 각 팀(네임스페이스)마다 똑같이 적용"할 때 매번 Role을 새로 만들 필요가 없어 관리 효율이 높아집니다.
> 

## 5. 권한 확인하기

RBAC 설정을 마친 후, 실제로 권한이 잘 들어갔는지 어떻게 확인할까요?
`auth can-i` 명령어

```
# 내가 파드를 만들 수 있나?
kubectl auth can-i create pods
# > yes

# jane이라는 사용자가 default 네임스페이스에서 파드를 삭제할 수 있나? (관리자만 실행 가능)
kubectl auth can-i delete pods --as jane --namespace default
# > no
```

# 4. Service Accounts

쿠버네티스 클러스터에는 두 종류의 사용자가 있습니다.
하나는 `kubectl`을 두드리는 User, 다른 하나는 클러스터 내부에서 API를 호출하는 애플리케이션(Service Account)입니다.

이번 글에서는 파드(Pod)가 API 서버와 안전하게 통신하기 위해 사용하는 **Service Account**의 개념과 토큰 관리 방식을 알아봅니다.

## 1. 계정의 두 가지 유형 (User vs Service Account)

가장 먼저 짚고 넘어가야 할 것은 "사람용 계정"과 "서비스용 계정"의 차이입니다.

| 구분 | User Account (사용자) | Service Account (서비스 어카운트) |
| --- | --- | --- |
| **대상** | 관리자(Admin), 개발자 | 프로메테우스(Prometheus), Jenkins, Pod 내부 앱 |
| **범위** | **Global** (클러스터 전역) | **Namespaced** (특정 네임스페이스 소속) |
| **관리 주체** | 외부 (LDAP, AD, 인증서 등) | **Kubernetes API**가 직접 관리 |
| **생성 방법** | `kubectl create user` (불가) | `kubectl create serviceaccount` (가능) |

> 쿠버네티스는 사람 계정을 직접 관리하지 않지만, 서비스 어카운트는 쿠버네티스 리소스로서 직접 생성하고 관리합니다.
> 

## 2. Service Account가 필요한 순간

파드 내부의 애플리케이션이 쿠버네티스 API 서버에 접근해야 할 때가 있습니다.

- **모니터링 (예: Prometheus):** 클러스터 전체의 메트릭을 수집하기 위해 API를 조회해야 합니다.
- **CI/CD (예: Jenkins):** 배포를 위해 API 서버에 파드 생성 요청을 보내야 합니다.
- **커스텀 컨트롤러:** 파드 상태를 감시하고 조작해야 합니다.

이때 애플리케이션은 "나 프로메테우스인데, 데이터 좀 줘"라고 신분증(Token)을 제시해야 하는데, 이 신분증이 바로 **Service Account**입니다.

## 3. Service Account 생성 및 관리

### 생성 (Imperative)

```
kubectl create serviceaccount my-monitoring-s
```

### 조회

```
kubectl get serviceaccount
```

## 4. 토큰의 비밀 (Token Management)

Service Account의 핵심은 인증 토큰(Bearer Token)입니다. 이 토큰이 있어야 API 서버가 문을 열어줍니다.

### 과거의 방식 (Legacy) vs 최신 방식 (Projected Volume)

1. **Old Way (Secret 기반):** Service Account를 만들면 자동으로 영구적인 Secret(토큰)이 생성되었습니다. (보안 취약)
2. **New Way (TokenRequest API 기반):**
    - Service Account를 만들어도 Secret이 자동으로 생성되지 않습니다.
    - 파드가 생성될 때, 쿠버네티스가 단기 토큰(Short-lived Token)을 발급합니다.
    - 이 토큰은 **Projected Volume** 형태로 파드 내부에 마운트됩니다.
    - 토큰 만료 시간이 다가오면 쿠버네티스가 자동으로 토큰을 교체(Rotate)해줍니다.
    - 파드가 삭제되면 토큰도 즉시 만료됩니다.

### 파드 내부에서 토큰 확인하기

파드 내부의 다음 경로에 토큰이 파일로 저장됩니다.
`/var/run/secrets/kubernetes.io/serviceaccount/token`

## 5. Default Service Account

우리가 파드를 만들 때 Service Account를 지정하지 않으면 어떻게 될까요?

- 모든 네임스페이스에는 기본적으로 `default`라는 이름의 Service Account가 존재합니다.
- 파드 생성 시 별도 지정이 없으면, 자동으로 `default` 계정이 할당됩니다.
- 그리고 `default` 계정의 토큰이 파드에 자동으로 마운트됩니다.

> 주의: 기본적으로 default 계정은 아무런 권한(RBAC)이 없습니다. 따라서 API 서버에 접근해도 403 Forbidden 에러가 뜹니다. 권한이 필요하다면 별도의 SA를 만들고 RBAC을 연결해야 합니다.
> 

## 6. 파드에 적용하기

### YAML 정의

파드 정의 파일의 `spec.serviceAccountName` 필드를 사용합니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: my-dashboard
spec:
  containers:
  - name: my-dashboard
    image: my-dashboard-image
  serviceAccountName: my-monitoring-sa  # 여기에 SA 이름 지정

```

### 주의사항

- **수정 불가:** 이미 실행 중인 파드의 Service Account는 변경할 수 없습니다. 파드를 삭제하고 다시 생성해야 합니다.
- **토큰 마운트 비활성화:** 만약 API 서버에 접근할 필요가 없는 파드라면 보안을 위해 토큰 마운트를 끄는 것이 좋습니다.
    
    ```
    automountServiceAccountToken: false
    ```
    

# 5. Image Security

아무리 튼튼한 금고(클러스터)를 만들어도, 그 안에 시한폭탄(취약한 이미지)을 넣으면 소용없습니다.

이번 글에서는 이미지를 안전하게 관리하는 방법과 Private Registry(비공개 저장소)에 있는 이미지를 가져오는 방법을 알아봅니다.

## 1. 이미지 보안의 3요소

안전한 이미지를 사용하기 위해 다음 세 가지 질문을 던져야 합니다.

### 1) 신뢰할 수 있는 출처인가? (Trusted Registries)

- 아무나 올릴 수 있는 Docker Hub의 공용 이미지를 맹신하면 안 됩니다.
- 검증된 공식 이미지(Official Image)를 사용하거나 조직 내부의 **Private Registry**를 구축하여 엄격하게 관리된 이미지만 사용해야 합니다.

### 2) 변조되지 않았는가? (Image Signing)

- 이미지가 생성된 후 배포되기 전까지 누군가 악의적으로 코드를 심지 않았는지 확인해야 합니다.
- **이미지 서명(Image Signing)** 기술을 사용하여 서명이 유효한 이미지만 클러스터에서 실행되도록 강제할 수 있습니다.

### 3) 취약점이 없는가? (Vulnerability Scanning)

- 이미지는 OS 라이브러리(Ubuntu, Alpine 등)를 포함하고 있습니다. 여기에 알려진 보안 취약점(CVE)이 있는지 주기적으로 스캔해야 합니다.
- **Trivy, Clair** 같은 도구를 사용하여 CI/CD 파이프라인 단계에서 검사를 수행합니다.

## 2. Private Registry 접근하기 (imagePullSecrets)

보안을 위해 Docker Hub의 Public 리포지토리가 아닌 AWS ECR이나 사내 Private Registry를 사용합니다. 이때 쿠버네티스는 해당 레지스트리에 로그인할 수 있는 자격 증명(Credential)이 필요합니다.

### 1단계: 로그인 정보를 담은 Secret 생성

먼저 레지스트리 접속 정보를 담은 `docker-registry` 타입의 Secret을 만듭니다.

```
kubectl create secret docker-registry reg-cred \
  --docker-server=my-private-registry.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com
```

### 2단계: 파드에 Secret 연결 (imagePullSecrets)

이제 파드를 생성할 때, "이미지를 당겨올(Pull) 때 이 Secret을 사용해라"라고 명시해야 합니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: private-reg-pod
spec:
  containers:
  - name: private-reg-container
    image: [my-private-registry.com/my-app:v1](https://my-private-registry.com/my-app:v1)
  imagePullSecrets:  # 여기가 핵심!
  - name: reg-cred
```

> Tip: 매번 imagePullSecrets를 적기 귀찮다면?
Service Account에 이 Secret을 연결해 두면, 해당 Service Account를 사용하는 모든 파드에 자동으로 imagePullSecrets가 주입됩니다.
> 

## 3. Security Context: 컨테이너 권한 제어

이미지가 안전하더라도 컨테이너가 **Root 권한**으로 실행되는 것은 위험합니다.
파드나 컨테이너 레벨에서 **Security Context**를 설정하여 권한을 제한해야 합니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000       # Root(0)가 아닌 일반 사용자(1000)로 실행
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: sec-ctx-demo
    image: busybox
    securityContext:
      allowPrivilegeEscalation: false # 권한 상승 금지
      readOnlyRootFilesystem: true    # 루트 파일 시스템을 읽기 전용으로 설정
```

# 6. Security Context

쿠버네티스에서 `Security Context`를 설정하다 보면 "왜 루트(Root)로 실행하면 안 되지?" 같은 의문이 듭니다. 이를 이해하려면 먼저 **도커(Docker)와 리눅스 커널의 관계**를 알아야 합니다.

## 1.

### 1) 커널 공유 (Shared Kernel)

- **VM:** 각자 고유한 커널(OS)을 가집니다. 하이퍼바이저가 완벽하게 분리합니다.
- **컨테이너:** 호스트의 **리눅스 커널을 공유**합니다.
    - 컨테이너 안에서 실행되는 프로세스는 사실 **호스트 OS 입장에서는 그냥 하나의 프로세스**일 뿐입니다.
    - 다만, 네임스페이스(Namespace)라는 벽을 쳐서 자기만의 세상에 갇혀 있다고 착각하게 만드는 것입니다.

### 2) 프로세스 격리 (PID Namespace)

- 컨테이너 안에서 `ps aux`를 치면 PID 1번으로 보입니다.
- 하지만 호스트에서 `ps aux`를 치면 PID 3456번(예시)으로 보입니다.
- 즉, 컨테이너는 호스트의 프로세스 공간을 나누어 쓰는 것입니다.

### 3) 사용자 격리 (User)

- 기본적으로 컨테이너 내부의 `root` 사용자는 **호스트의 `root` 사용자와 동일한 권한**을 가질 수 있습니다. (매우 위험!)
- 따라서 실무에서는 컨테이너가 Root 권한으로 실행되는 것을 엄격히 금지합니다.

## 2. 리눅스 케이퍼빌리티 (Linux Capabilities)

"Root 권한을 주기는 싫은데, 특정 작업(예: 시스템 시간 변경, 네트워크 포트 바인딩)은 해야 한다면?"

리눅스는 Root의 권한을 잘게 쪼개 놓았습니다. 이것을 **Capabilities**라고 합니다.
도커와 쿠버네티스는 컨테이너에게 Root 권한을 통째로 주는 대신 **필요한 Capability만 골라서 줄 수 있습니다.**

- **`SYS_TIME`:** 시스템 시간 변경 권한
- **`NET_ADMIN`:** 네트워크 설정 변경 권한
- **`CHOWN`:** 파일 소유권 변경 권한

## 3. 쿠버네티스 Security Context

위에서 배운 개념을 쿠버네티스 파드에 적용하는 설정이 바로 `securityContext`입니다.
이 설정은 **파드 레벨(Pod Level)**과 **컨테이너 레벨(Container Level)** 두 곳에 할 수 있습니다.

### 1) 파드 레벨 (Pod Level)

파드 내의 **모든 컨테이너**에 공통으로 적용되는 설정입니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-pod
spec:
  securityContext:
    runAsUser: 1000     # 모든 컨테이너를 User ID 1000으로 실행
    runAsGroup: 3000    # Group ID 3000
    fsGroup: 2000       # 볼륨 마운트 시 파일 소유 그룹 ID
  containers:
  - name: sec-ctx-demo
    image: busybox
    command: ["sh", "-c", "sleep 1h"]

```

### 2) 컨테이너 레벨 (Container Level)

**특정 컨테이너**에만 적용되는 설정입니다. 만약 파드 레벨과 설정이 겹치면 컨테이너 레벨의 설정이 우선(Override)됩니다. 또한, `capabilities` 설정은 컨테이너 레벨에서만 가능합니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-container
spec:
  containers:
  - name: sec-ctx-demo
    image: ubuntu
    command: ["sh", "-c", "sleep 1h"]
    securityContext:
      runAsUser: 0      # 이 컨테이너만 Root(0)로 실행 (파드 설정 덮어씀)
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"] # 네트워크 및 시간 설정 권한 추가
        drop: ["CHOWN"]                # 소유권 변경 권한 제거

```

## 4. 주요 설정 옵션 요약

| 옵션 | 설명 | 적용 레벨 |
| --- | --- | --- |
| **`runAsUser`** | 컨테이너를 실행할 사용자 ID (UID) 지정 | Pod / Container |
| **`runAsGroup`** | 컨테이너를 실행할 그룹 ID (GID) 지정 | Pod / Container |
| **`fsGroup`** | 파드에 마운트된 볼륨의 소유권을 가질 그룹 ID | **Pod Only** |
| **`capabilities`** | 리눅스 권한 추가(add) 또는 제거(drop) | **Container Only** |
| **`privileged`** | 호스트의 모든 권한을 가짐 (사실상 Root) | Container Only |

## 5. 보안 컨텍스트 설정 사례

1. **최소 권한 원칙 (Least Privilege):** 컨테이너가 동작하는 데 딱 필요한 권한만 부여해야 합니다.
2. **Root 실행 금지 (`runAsNonRoot`):** 가능하다면 `runAsNonRoot: true`를 설정하여 컨테이너가 루트 권한으로 실행되는 것을 원천 차단하는 것이 좋습니다.
3. **불필요한 기능 제거 (Drop Capabilities):** 기본적으로 부여되는 Capability 중 필요 없는 것은 `drop: ["ALL"]`로 제거하고, 필요한 것만 `add`로 추가하는 것이 가장 안전합니다.
4. **읽기 전용 파일 시스템 (`readOnlyRootFilesystem`):** 침입하더라도 파일을 변조할 수 없도록 루트 파일 시스템을 읽기 전용으로 설정합니다.

# 7.  Network Policies (네트워크 정책)

쿠버네티스 네트워크의 기본은 "모든 파드는 다른 모든 파드와 통신할 수 있어야 한다(All Allow)"입니다. 하지만 개발 편의성 측면에서는 좋지만 보안 측면에서는 문제가 됩니다..

만약 해커가 웹 프론트엔드 파드 하나를 해킹했다면? 별다른 제약 없이 내부의 DB 파드나 기밀 정보를 다루는 파드에 바로 접근할 수 있게 됩니다.

이것을 막기 위해 파드 간의 트래픽을 제어하는 규칙 즉 **Network Policy**가 필요합니다.

## 1. 전제 조건: 네트워크 플러그인 (CNI)

Network Policy를 만들기 전에 반드시 확인해야 할 것이 있습니다.
**"지금 사용 중인 네트워크 플러그인(CNI)이 Network Policy를 지원하는가?”**

## 2. Network Policy의 기본 개념

Network Policy는 **파드에 적용하는 방화벽 규칙**입니다.

1. **기본 상태:** 정책이 없으면 모두 허용(Allow All)입니다.
2. **정책 적용 시:** 특정 파드에 정책이 하나라도 연결되면 그 파드는 **기본 거부(Default Deny)** 상태로 바뀌고 **명시적으로 허용된 트래픽만 통과**됩니다.

## 3. 정책의 구조 (Anatomy)

Network Policy는 크게 3가지 요소로 구성됩니다.

1. **대상 선정 (`podSelector`):** 어떤 파드에 이 규칙을 적용할 것인가? (방화벽을 어디에 세울 것인가?)
2. **정책 유형 (`policyTypes`):** 들어오는 것(Ingress)인가, 나가는 것(Egress)인가?
3. **규칙 (`rules`):** 누구를 허용할 것인가? (IP, Pod, Namespace)

### 트래픽 방향

- **Ingress (수신):** 외부 -> 나 (들어오는 트래픽)
- **Egress (송신):** 나 -> 외부 (나가는 트래픽)

## 4. e.g DB 파드 보호하기 (Ingress)

**시나리오:**

- **DB 파드:** 3306 포트로 실행 중.
- **API 파드:** DB에 접속해야 함.
- **목표:** DB 파드는 오직 **API 파드**에서 오는 트래픽만 **3306 포트**로 허용하고 나머지는 모두 차단한다.

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  # 1. 대상 선정: 'role: db' 라벨을 가진 파드에 이 정책을 적용
  podSelector:
    matchLabels:
      role: db

  # 2. 정책 유형: 들어오는 트래픽(Ingress)을 제어하겠다
  policyTypes:
  - Ingress

  # 3. 규칙: 허용할 대상 정의 (화이트리스트 방식)
  ingress:
  - from:
    # 조건 A: 'name: api-pod' 라벨을 가진 파드만 허용
    - podSelector:
        matchLabels:
          name: api-pod
    # (필요시) 조건 B: 특정 네임스페이스에서 오는 것만 허용
    # - namespaceSelector:
    #     matchLabels:
    #       name: prod

    ports:
    - protocol: TCP
      port: 3306
```

## 5. e.g 외부 통신 차단하기 (Egress)

**시나리오:**

- **보안 파드:** 내부 로직만 수행하며 외부 인터넷(Google, Naver 등)으로 데이터를 전송하면 안 됨.
- **목표:** 이 파드에서 나가는(Egress) 모든 트래픽을 차단하되, **내부 DNS 서버(53번 포트)** 통신만 허용한다. (DNS가 안 되면 내부 통신도 마비되므로)

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external-egress
spec:
  podSelector:
    matchLabels:
      role: secure-app
  policyTypes:
  - Egress

  egress:
  - to:
    # 1. 특정 IP 대역으로만 통신 허용 (예: 쿠버네티스 내부 DNS)
    # 여기서는 예시로 모든 목적지의 53번 포트만 허용
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53

```

## 6. AND vs OR

하이픈(`-`)의 위치에 따라 의미가 완전히 달라집니다.

### 1) OR 조건 (또는)

배열(`-`)을 분리해서 작성하면 **"이거거나, 저거거나"** (합집합)로 동작합니다.

```
ingress:
  - from:
    - podSelector: { ... }      # 조건 A
    - namespaceSelector: { ... } # 조건 B

```

> 의미: "파드 라벨이 맞거나(A) 또는 네임스페이스 라벨이 맞으면(B) 통과시켜라." (개방적)
> 

### 2) AND 조건 (그리고)

하나의 항목 안에 묶어서 작성하면 **"이거면서 동시에 저거여야 함"** (교집합)으로 동작합니다.

```
ingress:
  - from:
    - podSelector: { ... }       # 조건 A
      namespaceSelector: { ... } # 조건 B (하이픈 없음!)

```

> 의미: "특정 네임스페이스(B)에 있으면서 동시에 특정 파드 라벨(A)을 가진 녀석만 통과시켜라." (제한적)
> 

## 7. Selectors

`from` (Ingress) 또는 `to` (Egress) 섹션에서 대상을 지정하는 방법은 3가지가 있습니다.

1. **`podSelector`:** 특정 라벨을 가진 파드. (같은 네임스페이스 내)
2. **`namespaceSelector`:** 특정 라벨을 가진 네임스페이스에 있는 모든 파드.
3. **`ipBlock`:** 특정 IP 대역 (CIDR). 클러스터 외부 통신 제어 시 사용.

## 8. Default Deny All

"일단 다 막고 시작하는 것"

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {} # 모든 파드 선택
  policyTypes:
  - Ingress
  # ingress 규칙이 아예 없음 = 아무것도 허용 안 함
```

이렇게 하면 해당 네임스페이스의 모든 파드는 서로 통신이 불가능해집니다. 이후 필요한 것만 하나씩 열어주는 방식(Whitelisting)이 보안상 안전합니다.

# 9. kubectx & kubens

실습 환경에서는 네임스페이스나 컨텍스트를 바꿀 일이 많지 않지만, 실제 운영(Production) 환경에서는 수십 개의 클러스터와 네임스페이스를 넘나들며 작업해야 합니다.

이때마다 `kubectl config use-context ...` 같은 긴 명령어를 치는 것은 매우 비효율적이고 오타를 유발합니다. 이 과정을 줄여주는 두 가지 필수 유틸리티가 있습니다. 

## 1. kubectx: 클러스터(Context) 순간 이동

여러 개의 클러스터(예: Dev, Staging, Prod)를 관리할 때 **컨텍스트(Context)를 빠르게 전환**해 주는 도구입니다.

### 기존 방식 (kubectl)

```
kubectl config get-contexts
kubectl config use-context my-production-cluster
```

명령어가 너무 길고 복잡합니다.

### kubectx 사용 방식

```
# 목록 확인
kubectx

# 전환
kubectx my-production-cluster

# 직전 컨텍스트로 복귀 (이게 진짜 꿀기능!)
kubectx -
```

### 설치 방법 (Linux)

```
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
```

### 주요 명령어

- **`kubectx`**: 모든 컨텍스트 목록 나열
- **`kubectx <이름>`**: 해당 컨텍스트로 전환
- **`kubectx -`**: 바로 이전에 사용하던 컨텍스트로 전환 (Alt+Tab 느낌)
- **`kubectx -c`**: 현재 사용 중인 컨텍스트 이름 확인

## 2. kubens: 네임스페이스(Namespace) 순간 이동

특정 네임스페이스를 기본값(Default)으로 설정하여, 매번 명령어 뒤에 `-n <namespace>`를 붙이지 않게 해 줍니다.

### 기존 방식 (kubectl)

```
# 매번 붙이거나..
kubectl get pods -n my-namespace

# 기본값 변경을 위해 긴 명령어 입력..
kubectl config set-context --current --namespace=my-namespace
```

### kubens 사용 방식

```
# 전환 (이제부터 모든 명령은 my-namespace 기준)
kubens my-namespace

# 다시 default로 복귀
kubens default
```

### 설치 방법 (Linux)

`kubectx` 리포지토리에 같이 포함되어 있습니다.

```
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
```

# 10. CRD (Custom Resource Definition)

쿠버네티스는 기본적으로 `Pod`, `Service`, `Deployment` 같은 다양한 리소스를 제공합니다.
하지만 사용하다 보면 "우리 회사만의 특별한 리소스가 있었으면 좋겠다"는 생각이 들 때가 있습니다.

예를 들어 `FlightTicket`이라는 리소스를 만들고 `kubectl get flightticket`이라고 명령했을 때 항공권 정보가 나오게 할 수는 없을까요? 이것을 가능하게 해주는 것이 바로 CRD(Custom Resource Definition)입니다.

## 1. 기본 개념: CRD vs Custom Resource

이 둘의 차이를 이해하는 것이 가장 중요합니다.

### 1) Custom Resource (CR)

- **개념:** 우리가 사용하고 싶은 실체(Instance)입니다.
- **비유:** 사전(Dictionary)에 등록된 단어를 사용하여 만든 **문장**입니다.
- **예시:** `my-flight-ticket`이라는 이름의 객체.

### 2) Custom Resource Definition (CRD)

- **개념:** 그 리소스가 무엇인지 쿠버네티스에게 알려주는 정의(Definition)입니다.
- **비유:** 새로운 단어를 **사전에 등록**하는 행위입니다.
- **역할:** "이 리소스의 이름은 FlightTicket이고, 필수 필드는 destination과 price야"라고 API 서버에 등록해 줍니다.

> 즉, CRD를 먼저 만들어서 등록해야만 CR을 생성할 수 있습니다.
> 

## 2. CRD 만들기 (Defining the Resource)

`FlightTicket`이라는 리소스를 정의하는 YAML 파일을 만들어 보겠습니다.

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: flighttickets.flights.com  # 이름 규칙: <plural>.<group>
spec:
  group: flights.com      # API 그룹 이름
  versions:
    - name: v1
      served: true
      storage: true
      schema:             # 데이터 검증 (Validation)
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                destination:
                  type: string
                price:
                  type: integer
  scope: Namespaced       # Namespaced 또는 Cluster
  names:
    plural: flighttickets
    singular: flightticket
    kind: FlightTicket    # YAML에서 쓸 Kind 이름
    shortNames:
    - ft                  # kubectl get ft 로 조회 가능하게 함

```

이 파일을 `kubectl apply -f crd.yaml`로 적용하면 이제 쿠버네티스는 `FlightTicket`이라는 리소스를 인식하게 됩니다.

## 3. 커스텀 리소스 사용하기 (Creating Objects)

이제 우리가 만든 정의에 따라 실제 리소스를 생성할 수 있습니다.

```
apiVersion: [flights.com/v1](https://flights.com/v1)
kind: FlightTicket        # 위에서 정의한 Kind 사용!
metadata:
  name: my-ticket
spec:
  destination: "New York"
  price: 500
```

이제 `kubectl get flightticket` (또는 `kubectl get ft`) 명령어를 입력하면 `my-ticket`이 조회됩니다!

## 4. Custom Controllers

CRD만으로는 데이터 저장소(etcd)에 데이터를 넣는 것 외에 아무 일도 일어나지 않습니다.
실제로 항공권을 예약하거나 데이터베이스를 생성하는 등의 작업을 수행하려면 **Custom Controller**가 필요합니다.

### 1) 컨트롤러의 역할 (Control Loop)

컨트롤러는 **무한 루프**를 돌며 다음 과정을 반복합니다.

1. **Watch:** API 서버를 감시하다가 `FlightTicket` 생성/수정 이벤트를 감지합니다.
2. **Reconcile (조정):**
    - 사용자가 원하는 상태(Desired State): `destination: New York`
    - 실제 상태(Current State): 아직 예약 안 됨
    - **Action:** 외부 항공사 API를 호출하여 티켓을 예약합니다.
3. **Update Status:** 예약이 성공하면 리소스의 상태(`status`)를 'Booked'로 업데이트합니다.

### 2) 개발 언어 및 도구

어떤 언어로든 만들 수 있지만 쿠버네티스 생태계가 **Go** 언어로 되어 있기 때문에 Go를 사용하는 것이 가장 강력하고 라이브러리 지원(Client-go)이 좋습니다.

- **Go:** `client-go`, `controller-runtime` (표준, 강력함)
- **Python/Java 등:** 가능은 하지만 캐싱, 큐 관리 등을 직접 구현해야 할 수 있음.

### 3) 배포 (Deployment)

개발된 컨트롤러 코드는 다음 과정을 거쳐 클러스터에 배포됩니다.

1. **빌드:** 코드를 빌드하여 실행 파일 생성.
2. **도커 이미지:** `Dockerfile`을 이용해 컨테이너 이미지로 패키징.
3. **파드 배포:** 클러스터 내에 `Deployment` 형태로 배포되어 24시간 동작.

## 5. Operator Framework & Operator Hub

앞서 배운 **CRD**와 **Controller**는 기술적인 구성 요소입니다. 이 둘을 결합하여 "특정 애플리케이션의 운영 지식(Operational Knowledge)을 자동화"한 것을 **Operator**라고 부릅니다.

### 1) Operator가 하는 일

단순히 설치(Install)만 하는 것이 아니라 수동으로 하던 작업을 대신 수행합니다.

- **설치:** `EtcdCluster` 리소스를 만들면 etcd 클러스터를 자동으로 구성.
- **백업/복구:** 주기적으로 데이터를 백업하고, 장애 시 복구.
- **업그레이드:** 무중단으로 버전을 업그레이드.

### 2) Operator Framework

개발자가 Operator를 쉽게 개발하고 배포할 수 있도록 돕는 도구 모음입니다. CRD와 컨트롤러를 패키징하여 관리하기 쉽게 해줍니다.

### 3) Operator Hub (https://operatorhub.io)

전 세계 개발자들이 만든 Operator들을 모아놓은 **앱 스토어** 같은 곳입니다.

- **etcd operator, Prometheus operator, MySQL operator** 등을 여기서 찾아 쉽게 설치할 수 있습니다.
- `kubectl` 명령어로오퍼레이터를 설치하고 CR(Custom Resource)만 생성하면 DB 클러스터가 만들어집니다.
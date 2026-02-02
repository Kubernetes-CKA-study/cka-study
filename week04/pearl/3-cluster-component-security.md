# Cluster Component Security
- 지금까지 API 접근 제어, 워크로드, 네트워크 등 클러스터를 사용하는 관점의 보안을 다뤘다.
- 다음으로, 클러스터 자체를 구성하는 핵심 컴포넌트, 특히 클러스터에서 가장 중요한 `etcd`와 `kube-apiserver`의 보안 설정을 알아야 한다.


## 1. etcd 보안
- etcd는 클러스터의 모든 상태 정보(모든 리소스 정의, 시크릿 등)를 저장하는 분산 Key-Value 저장소이다.
- etcd가 탈취당하면 클러스터 전체가 장악되는 것과 같으므로, etcd 보안은 클러스터 보안의 최우선 순위이다.

  - **핵심 원칙:** `etcd`와 다른 컴포넌트(특히 `kube-apiserver`) 간의 모든 통신은 반드시 상호 TLS(mTLS) 암호화를 통해 이루어져야 한다.
  - **주요 etcd 실행 옵션 (Flags)**
      - `--cert-file`, `--key-file`: `etcd` 서버 자체의 신원을 증명하기 위한 TLS 인증서와 개인 키 확인 (서버 측)
      - `--client-cert-auth=true`: `etcd`에 연결하는 모든 클라이언트에게 반드시 유효한 인증서를 제시하도록 강제하는 옵션이다.
      - `--trusted-ca-file`: 클라이언트의 인증서가 유효한지 검증할 때 사용할 CA(Certificate Authority) 인증서이다.
      - `--peer-cert-file`, `--peer-key-file`, `--peer-trusted-ca-file`: `etcd` 클러스터 멤버들 간의 통신을 암호화하기 위한 피어(Peer) 인증서 관련 옵션이다.

### 실습 포인트 - 1 (etcd 보안 설정 확인)

  - **시나리오:** 현재 클러스터의 `etcd`가 TLS 통신을 사용하도록 올바르게 설정되었는지 확인하기

<!-- end list -->

1.  **`etcd`의 Static Pod Manifest 파일 찾기**

      - `kubeadm`으로 설치된 클러스터의 경우, `etcd`는 Static Pod로 실행됩니다. Manifest 파일은 보통 마스터 노드의 `/etc/kubernetes/manifests/etcd.yaml` 에 위치합니다.

2.  **Manifest 파일 내용 확인**

      - `cat /etc/kubernetes/manifests/etcd.yaml` 명령어로 파일 내용을 엽니다.
      - `spec.containers.command` 섹션에서 위에서 언급한 TLS 관련 플래그들이 올바르게 설정되어 있는지 확인합니다.

    <!-- end list -->

    ```bash
    # 예시: etcd.yaml 파일의 command 섹션
    ...
    command:
    - etcd
    - --advertise-client-urls=https://...
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt         # 서버 인증서 설정 확인
    - --key-file=/etc/kubernetes/pki/etcd/server.key          # 서버 키 설정 확인
    - --client-cert-auth=true                                 # 클라이언트 인증 강제 확인 (매우 중요!)
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt       # CA 인증서 설정 확인
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt      # 피어 인증서 설정 확인
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key       # 피어 키 설정 확인
    - --peer-client-cert-auth=true                            # 피어 클라이언트 인증 강제 확인
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt  # 피어 CA 인증서 설정 확인
    ...
    ```

---
## 2. API 서버 보안
- `kube-apiserver`는 클러스터의 모든 요청이 거쳐가는 관문(Gateway)이다. 
- RBAC 외에도 다양한 보안 관련 플래그 설정을 통해 API 서버의 방어벽을 더욱 견고하게 만들 수 있다.

  - **주요 API 서버 실행 옵션 (Flags)**
      - `--authorization-mode`: API 요청에 대한 인가(Authorization) 플러그인 체인을 지정한다. `Node,RBAC` 처럼 필요한 것만 명시적으로 활성화하는 것이 권장된다. (`AlwaysAllow`는 보안상 절대 사용 금지)
      - `--anonymous-auth=false`: 인증되지 않은 익명(anonymous) 사용자의 요청을 허용하지 않도록 설정한다. 보안상 `false`로 설정하기.
      - `--kubelet-client-certificate`, `--kubelet-client-key`: API 서버가 각 노드의 `kubelet`에 접속하여 로그를 가져오거나 `exec` 명령을 실행할 때, 자신을 `kubelet`에 인증하기 위해 사용하는 클라이언트 인증서를 지정한다.

### 실습 포인트 - 2 (API 서버 보안 설정 확인)

  - **시나리오:** API 서버가 안전한 인가 모드를 사용하고, 익명 접근을 차단하고 있는지 확인

<!-- end list -->

1.  **`kube-apiserver`의 Static Pod Manifest 파일 찾기**

      - 마스터 노드의 `/etc/kubernetes/manifests/kube-apiserver.yaml` 에 위치한다.

2.  **Manifest 파일 내용 확인**

      - `cat /etc/kubernetes/manifests/kube-apiserver.yaml` 명령어로 파일 내용 열기
      - `spec.containers.command` 섹션에서 보안 관련 주요 플래그들을 확인하기

    <!-- end list -->

    ```bash
    # 예시: kube-apiserver.yaml 파일의 command 섹션
    ...
    command:
    - kube-apiserver
    - --advertise-address=...
    # 아래 두 플래그가 보안적으로 올바르게 설정되었는지 확인
    - --authorization-mode=Node,RBAC
    - --anonymous-auth=false
    # API 서버가 Kubelet과 통신할 때 사용하는 인증서
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    ...
    ```

      - 만약 `--authorization-mode=AlwaysAllow`가 포함되어 있거나 `--anonymous-auth=true`로 되어 있다면, 이는 심각한 보안 취약점이므로 즉시 수정해야 한다.
# 1.Pre-requisite - Switching, Routing, Gateways

## Switching (스위칭)

컴퓨터 두 대가 직접 케이블로 연결되면 통신할 수 있습니다. 하지만 수십, 수백 대의 컴퓨터가 있다면 모두를 직접 연결하는 건 불가능합니다. 이때 **스위치(Switch)** 가 중간에서 허브 역할을 해줍니다.

스위치는 **같은 네트워크 안**에서만 동작합니다. 예를 들어 `192.168.1.10`과 `192.168.1.11` 두 컴퓨터가 같은 스위치에 연결되어 있으면, 서로 자유롭게 통신할 수 있습니다. 이때 두 컴퓨터는 같은 네트워크 대역(`192.168.1.0/24`)에 속해 있어야 합니다.

## Routing (라우팅)

서로 다른 네트워크 간에 통신할 때 라우터를 통해 패킷을 전달합니다. 

스위치는 같은 네트워크 안에서만 통신이 가능합니다. 그렇다면 `192.168.1.x` 네트워크의 컴퓨터가 `192.168.2.x` 네트워크의 컴퓨터와 통신하려면 어떻게 해야 할까요? 이때 **라우터(Router)** 가 필요합니다.

라우터는 서로 다른 네트워크 사이에 위치해서 패킷을 어디로 전달할지 결정합니다. 라우터는 이 결정을 **라우팅 테이블(Routing Table)** 을 보고 내립니다. "이 목적지 주소로 가는 패킷은 저쪽 방향으로 보내라"는 경로 정보가 담겨 있습니다.

## Gateways (게이트웨이)

다른 네트워크로 나가는 기본 경로(Default Route)를 설정합니다.

라우팅 테이블에 경로를 하나하나 추가하는 건 내부 네트워크끼리 통신할 때는 괜찮습니다. 하지만 인터넷처럼 목적지가 무수히 많은 경우엔 일일이 경로를 추가하는 게 불가능합니다.

이때 사용하는 것이 **게이트웨이(Default Gateway)** 입니다. "라우팅 테이블에 일치하는 경로가 없으면, 일단 이쪽으로 내보내라"는 출구를 지정하는 것입니다. 가정집 공유기가 대표적인 게이트웨이입니다. 인터넷으로 나가는 모든 트래픽이 공유기를 통해 나갑니다..

# 02. Pre-requisite - DNS

## Name Resolution

호스트 이름으로 통신하려면 IP와 이름을 매핑해야 합니다. 

우리는 보통 IP 주소가 아닌 `google.com` 같은 이름으로 서버에 접속합니다. 컴퓨터는 이름만으로는 어디로 가야 할지 모르기 때문에 이름을 IP 주소로 변환하는 과정이 필요합니다. 이것을 **Name Resolution**이라고 합니다.

가장 단순한 방법은 `/etc/hosts` 파일에 직접 이름과 IP를 매핑하는 것입니다. 하지만 이 방식은 컴퓨터가 많아지면 모든 서버의 `/etc/hosts` 파일을 일일이 관리해야 하는 문제가 생깁니다. 이 문제를 해결하기 위해 **DNS 서버**가 등장했습니다.

## DNS 서버

DNS 서버는 이름-IP 매핑 정보를 중앙에서 관리합니다. 각 컴퓨터는 이름 해석이 필요할 때 DNS 서버에 물어봅니다. DNS 서버 주소는 `/etc/resolv.conf` 파일에 설정합니다.

## 조회 순서 설정

리눅스는 이름을 해석할 때 `/etc/hosts`를 먼저 볼지, DNS 서버를 먼저 볼지 순서를 설정할 수 있습니다. 이 순서는 `/etc/nsswitch.conf` 파일에서 정의합니다.

## DNS 조회

```yaml
# nslookup: DNS 서버에 이름 조회 (/etc/hosts는 무시)
$ nslookup www.google.com

# dig: 더 상세한 DNS 조회 정보 제공
$ dig www.google.com
```

# 3. Pre-requisite - CoreDNS

CoreDNS는 Go 언어로 작성된 가볍고 유연한 DNS 서버입니다. 쿠버네티스에서는 클러스터 내부 DNS 서버로 CoreDNS를 사용합니다. 파드나 서비스 이름을 IP로 변환하는 역할을 담당합니다.

## /etc/hosts 파일 연동

CoreDNS는 `/etc/hosts` 파일의 내용을 DNS 레코드로 제공할 수 있습니다. `Corefile`에 설정을 작성하고 실행하면 됩니다.

```yaml
$ cat > Corefile
. {
    hosts /etc/hosts
}

$ ./coredns
```

이 방식이 쿠버네티스 클러스터 DNS의 기초 원리입니다. 쿠버네티스에서는 파드와 서비스가 생성될 때 CoreDNS에 레코드가 자동으로 추가됩니다.

# 4. Pre-requisite - Network Namespaces

## 왜 네트워크 네임스페이스가 필요한가?

도커처럼 컨테이너를 실행하면 각 컨테이너는 호스트와 완전히 독립된 네트워크 환경을 갖습니다. 컨테이너 안에서 `ip link`를 실행하면 호스트의 네트워크 인터페이스는 보이지 않습니다. 이 격리를 가능하게 하는 리눅스 기술이 바로 **Network Namespace**입니다.

각 네임스페이스는 독립적인 네트워크 인터페이스, 라우팅 테이블, ARP 테이블을 갖습니다. 마치 각자 별도의 네트워크 장비를 가진 것처럼 동작합니다.

## 네임스페이스 생성 및 확인

```yaml
# 네임스페이스 생성
$ ip netns add red
$ ip netns add blue

# 목록 확인
$ ip netns

# 호스트의 인터페이스는 잘 보임
$ ip link

# 네임스페이스 안에서는 호스트 인터페이스가 안 보임 (격리됨)
$ ip netns exec red ip link
# loopback만 보임
```

## 가상 케이블(veth)로 두 네임스페이스 연결

두 네임스페이스를 직접 연결하려면 **veth(virtual ethernet)** 쌍을 사용합니다. 실제 LAN 케이블처럼 양쪽을 직접 연결하는 가상 케이블입니다.

```yaml
# veth 쌍 생성 (양쪽 끝이 연결된 가상 케이블)
$ ip link add veth-red type veth peer name veth-blue

# 각 끝을 네임스페이스에 연결
$ ip link set veth-red netns red
$ ip link set veth-blue netns blue

# IP 주소 할당
$ ip -n red addr add 192.168.15.1/24 dev veth-red
$ ip -n blue addr add 192.168.15.2/24 dev veth-blue

# 인터페이스 활성화
$ ip -n red link set veth-red up
$ ip -n blue link set veth-blue up

# 통신 테스트
$ ip netns exec red ping 192.168.15.2
```

## Linux Bridge (여러 네임스페이스를 하나로 연결)

네임스페이스가 여러 개라면 veth로 일일이 연결하는 건 비효율적입니다. **Linux Bridge**는 스위치처럼 동작하며 여러 네임스페이스를 하나의 네트워크로 묶어줍니다. 도커의 `docker0` 브릿지가 바로 이 원리로 동작합니다.

```yaml
# 브릿지 인터페이스 생성 (가상 스위치)
$ ip link add v-net-0 type bridge
$ ip link set dev v-net-0 up

# 각 네임스페이스와 브릿지를 연결하는 veth 쌍 생성
$ ip link add veth-red type veth peer name veth-red-br
$ ip link add veth-blue type veth peer name veth-blue-br

# 각 끝을 네임스페이스와 브릿지에 연결
$ ip link set veth-red netns red
$ ip link set veth-red-br master v-net-0
$ ip link set veth-blue netns blue
$ ip link set veth-blue-br master v-net-0

# IP 할당 및 활성화
$ ip -n red addr add 192.168.15.1/24 dev veth-red
$ ip -n blue addr add 192.168.15.2/24 dev veth-blue
$ ip -n red link set veth-red up
$ ip -n blue link set veth-blue up
$ ip link set dev veth-red-br up
$ ip link set dev veth-blue-br up

# 브릿지에도 IP 할당 (호스트가 네임스페이스와 통신하기 위해)
$ ip addr add 192.168.15.5/24 dev v-net-0
```

외부 네트워크(인터넷)와 통신하려면 NAT 설정도 필요합니다.

```yaml
# 네임스페이스에서 나가는 트래픽을 호스트 IP로 변환 (NAT)
$ iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE

# 네임스페이스에서 외부로 나가는 기본 경로 설정
$ ip netns exec blue ip route add default via 192.168.15.5

# 외부에서 네임스페이스 안으로 포트 포워딩
$ iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.15.2:80 -j DNAT
```

# 05. Pre-requisite - Docker Networking

## 네트워크 모드

도커는 컨테이너를 실행할 때 세 가지 모드를 지원합니다.

**None 모드**: 컨테이너가 완전히 네트워크에서 격리됩니다. 외부와 통신이 전혀 불가능합니다.

**Host 모드**: 컨테이너가 호스트의 네트워크를 그대로 공유합니다. 격리 없이 호스트와 동일한 IP, 포트를 사용합니다. 성능은 좋지만 격리가 없습니다.

**Bridge 모드 (기본값)**: 도커가 `docker0`라는 가상 브릿지를 만들고, 각 컨테이너는 이 브릿지에 veth로 연결됩니다. 컨테이너끼리는 통신 가능하지만 외부에서 접근하려면 포트 매핑이 필요합니다.

```yaml
$ docker run --network none nginx    # None 모드
$ docker run --network host nginx    # Host 모드
$ docker run --network bridge nginx  # Bridge 모드 (기본값)

$ docker network ls   # 네트워크 목록 확인
```

## 포트 매핑

Bridge 모드에서 컨테이너는 `172.18.0.x` 같은 내부 IP를 가집니다. 외부에서 접근하려면 호스트의 포트와 컨테이너의 포트를 매핑해야 합니다. 이 매핑은 내부적으로 `iptables` 규칙을 통해 동작합니다.

```yaml
# 컨테이너의 80포트를 호스트의 8080포트로 매핑
$ docker run -itd --name nginx -p 8080:80 nginx

# 외부에서 호스트 IP:8080으로 접근하면 컨테이너의 80포트로 전달됨
$ curl --head http://192.168.10.11:8080
```

# **6. Pre-requisite - CNI (Container Network Interface)**

## CNI란?

도커, 쿠버네티스 같은 컨테이너 플랫폼마다 네트워크를 연결하는 방식이 제각각이었습니다. 플러그인 개발자 입장에서는 플랫폼마다 별도로 드라이버를 만들어야 했고, 플랫폼 입장에서도 플러그인마다 다른 방식으로 연동해야 했습니다.

**CNI(Container Network Interface)** 는 이 문제를 해결하기 위한 **표준 인터페이스**입니다. "컨테이너를 네트워크에 연결하고 해제하는 방법"을 표준화해서, 어떤 컨테이너 런타임이든 CNI 규격만 맞추면 어떤 네트워크 플러그인이든 사용할 수 있게 합니다.

## 주요 CNI 플러그인

- **Weave**: 설치가 간단하고 멀티 호스트 네트워킹을 자동으로 처리
- **Calico**: 네트워크 정책(Network Policy) 기능이 강력, 대규모 환경에서 선호
- **Flannel**: 구조가 단순하고 가벼움
- **Cilium**: eBPF 기반으로 고성능, 보안 기능도 강력

# 7. Cluster Networking

쿠버네티스 클러스터를 구성하는 마스터 노드와 워커 노드는 서로 네트워크로 연결되어야 하며, 각 컴포넌트가 특정 포트를 통해 통신합니다.

## 주요 포트 정리

| 컴포넌트 | 포트 | 설명 |
| --- | --- | --- |
| kube-apiserver | 6443 | 모든 컴포넌트가 API 서버와 통신하는 포트 |
| etcd | 2379 | API 서버가 etcd에 접근하는 포트 |
| etcd (피어) | 2380 | etcd 클러스터 노드 간 통신 포트 |
| kube-scheduler | 10259 | 스케줄러 포트 |
| kube-controller-manager | 10257 | 컨트롤러 매니저 포트 |
| kubelet | 10250 | API 서버가 kubelet과 통신하는 포트 |

etcd가 2379와 2380 두 포트를 사용하는 이유는 2379는 API 서버 같은 클라이언트가 접속하는 포트이고 2380은 etcd 노드끼리 데이터를 동기화하는 피어 통신 포트이기 때문입니다. 단일 마스터 환경에서는 2380 포트는 사용되지 않습니다.

# 8. Pod Networking

## 쿠버네티스의 네트워킹 요구사항

쿠버네티스는 다음 규칙을 반드시 만족하는 네트워크 환경을 요구합니다.

- 모든 파드는 NAT 없이 다른 파드와 통신할 수 있어야 합니다.
- 모든 노드는 NAT 없이 모든 파드와 통신할 수 있어야 합니다.
- 파드가 자신의 IP를 볼 때와 다른 파드가 그 파드를 볼 때 IP가 동일해야 합니다.

이 요구사항을 만족하기 위해 각 노드에 브릿지 네트워크를 구성하고 노드 간 라우팅을 설정합니다.

## 노드별 브릿지 구성

각 노드에 브릿지를 만들고 노드마다 서로 다른 대역의 IP를 할당합니다. 같은 대역을 쓰면 다른 노드의 파드와 IP가 겹칠 수 있기 때문입니다.

```yaml
# node01 → 10.244.1.0/24 대역
$ ip link add v-net-0 type bridge
$ ip link set dev v-net-0 up
$ ip addr add 10.244.1.1/24 dev v-net-0

# node02 → 10.244.2.0/24 대역
$ ip addr add 10.244.2.1/24 dev v-net-0

# node03 → 10.244.3.0/24 대역
$ ip addr add 10.244.3.1/24 dev v-net-0
```

## 노드 간 라우팅 설정

브릿지만으로는 다른 노드의 파드와 통신할 수 없습니다. "저 노드의 파드 대역으로 가려면 어떤 라우터를 거쳐야 하는지" 알려주는 라우팅 규칙이 필요합니다.

```yaml
# node01에서: 다른 노드의 파드 대역으로 가는 경로 추가
$ ip route add 10.244.2.0/24 via 192.168.1.12   # node02의 실제 IP
$ ip route add 10.244.3.0/24 via 192.168.1.13   # node03의 실제 IP
```

실제 환경에서는 이 모든 작업을 **CNI 플러그인**이 자동으로 처리해줍니다. 파드가 생성되거나 삭제될 때마다 CNI 플러그인이 네트워크를 자동으로 구성합니다.

# 9. CNI in Kubernetes - Weave

## Weave가 하는 일

Weave Net은 각 노드에 **에이전트(Agent)** 를 DaemonSet으로 배포합니다. 이 에이전트들이 서로 통신하면서 전체 클러스터의 네트워크 토폴로지를 파악하고 파드 간 통신 경로를 자동으로 구성합니다. 관리자가 수동으로 라우팅 테이블을 설정할 필요가 없습니다.

설치하면 각 노드에 `weave-net` 파드가 하나씩 생성됩니다 (DaemonSet).

```yaml
# Weave 파드 상태 확인
$ kubectl get pods -n kube-system | grep weave
weave-net-jgr8x   2/2   Running   0   45m
weave-net-tb9tz   2/2   Running   0   45m

# Weave 로그 확인
$ kubectl logs weave-net-tb9tz weave -n kube-system

# 파드 내부의 기본 라우팅 확인 (Weave가 설정한 경로)
$ kubectl exec test -- ip route
default via 10.244.1.1 dev eth0
```

# 10. IPAM (IP Address Management) - Weave

## 파드 IP를 어떻게 관리하는가?

클러스터 안의 수백, 수천 개의 파드에 IP를 할당할 때 IP 충돌이 생기면 안 됩니다. 이 문제를 해결하는 것이 **IPAM(IP Address Management)** 입니다.

Weave는 전체 Pod 네트워크 대역(예: `10.32.0.0/12`)을 각 노드에 균등하게 분배합니다. 각 노드는 자신이 할당받은 대역 안에서만 파드에 IP를 부여하므로 노드 간 IP 충돌이 발생하지 않습니다. Weave 에이전트들이 서로 정보를 공유하면서 어느 노드가 어떤 대역을 쓰는지 파악하고 있습니다.

# 11. Service Networking

## 왜 Service가 필요한가?

파드는 언제든지 재시작될 수 있고 재시작되면 IP가 바뀝니다. 다른 파드가 이 파드에 접근하려면 매번 바뀐 IP를 추적해야 합니다. 이 문제를 해결하기 위해 **Service** 가 등장했습니다.

Service는 파드 앞에 붙는 안정적인 엔드포인트입니다. Service의 IP와 DNS 이름은 변하지 않기 때문에 다른 파드는 Service를 통해 안정적으로 접근할 수 있습니다.

## Service 타입

**ClusterIP**: 클러스터 내부에서만 접근 가능한 가상 IP를 부여합니다. 외부에서는 접근할 수 없습니다. 가장 기본적인 타입입니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: local-cluster
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
```

**NodePort**: 클러스터의 모든 노드에 특정 포트를 열어서 외부에서 `노드IP:포트`로 접근할 수 있게 합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-wide
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
```

## Service는 어떻게 동작하는가?

Service의 IP(ClusterIP)는 실제 어떤 인터페이스에도 할당되지 않는 **가상 IP**입니다. `kube-proxy`가 각 노드에서 `iptables` 규칙을 만들어서 이 가상 IP로 들어오는 트래픽을 실제 파드 IP로 전달해줍니다.

```yaml
$ kubectl get service
NAME            TYPE        CLUSTER-IP      PORT(S)
local-cluster   ClusterIP   10.101.67.139   80/TCP
nodeport-wide   NodePort    10.102.29.204   80:30016/TCP

# Service IP 대역 확인
$ ps -aux | grep kube-apiserver | grep service-cluster-ip-range
--service-cluster-ip-range=10.96.0.0/12

# kube-proxy가 만든 iptables 규칙 확인
$ iptables -L -t nat | grep local-cluster
```

# 12. DNS in Kubernetes

쿠버네티스는 파드와 서비스가 생성될 때 자동으로 DNS 레코드를 만들어줍니다. 덕분에 IP를 직접 몰라도 이름으로 다른 서비스에 접근할 수 있습니다.

## Service DNS 레코드

서비스가 생성되면 아래 형식의 DNS 이름이 자동으로 부여됩니다.

`<서비스이름>.<네임스페이스>.svc.cluster.local`

같은 네임스페이스 안에 있는 파드는 서비스 이름만으로도 접근이 가능합니다. 다른 네임스페이스에 있는 경우엔 네임스페이스까지 붙여야 합니다.

```yaml
# apps 네임스페이스의 서비스를 default 네임스페이스에서 조회
$ kubectl run -it test --image=busybox:1.28 --rm --restart=Never -- \
  nslookup nginx-service.apps.svc.cluster.local

# curl로 접근
$ kubectl run -it nginx-test --image=nginx --rm --restart=Never -- \
  curl -Is http://nginx-service.apps.svc.cluster.local
```

## Pod DNS 레코드

파드에도 DNS 레코드가 생성되지만 IP 주소의 점(`.`)을 대시(`-`)로 바꾼 형태로 만들어집니다.

```yaml
<파드IP의-점을-대시로>.<네임스페이스>.pod.cluster.local

예: 파드 IP가 10.244.1.3이라면
→ 10-244-1-3.apps.pod.cluster.local
```

```yaml
$ kubectl run -it test --image=busybox:1.28 --rm --restart=Never -- \
  nslookup 10-244-1-3.apps.pod.cluster.local
```

# 13. CoreDNS in Kubernetes

## CoreDNS의 역할

쿠버네티스 클러스터 안에서 DNS 서버 역할을 하는 것이 **CoreDNS**입니다. `kube-system` 네임스페이스에 Deployment로 배포되며 기본적으로 2개의 파드가 실행되어 고가용성을 확보합니다.

파드가 생성되면 kubelet이 파드의 `/etc/resolv.conf`에 CoreDNS 서비스의 IP(`10.96.0.10`)를 nameserver로 자동 설정해줍니다. 그래서 파드 안에서 서비스 이름으로 통신하면 자동으로 CoreDNS에 조회하게 됩니다.

```yaml
`# CoreDNS 파드 확인
$ kubectl get pods -n kube-system | grep coredns
coredns-66bff467f8-2vghh   1/1   Running   53m
coredns-66bff467f8-t5nzm   1/1   Running   53m

# CoreDNS 서비스 (kube-dns)
$ kubectl get service -n kube-system
NAME       TYPE        CLUSTER-IP   PORT(S)
kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP

# kubelet이 파드에 설정하는 DNS 서버 주소
$ cat /var/lib/kubelet/config.yaml | grep -A2 clusterDNS
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local`
```

## CoreDNS 설정 (Corefile)

CoreDNS의 동작은 `Corefile`이라는 설정 파일로 제어합니다. 쿠버네티스에서는 이 파일이 ConfigMap으로 관리됩니다.

```yaml
$ kubectl describe cm coredns -n kube-system

.:53 {
    errors                          # 에러 로깅
    health { lameduck 5s }          # 헬스체크
    ready                           # 준비 상태 체크
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure                # 파드 DNS 레코드 활성화
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153                # 메트릭 노출
    forward . /etc/resolv.conf      # 외부 도메인은 업스트림 DNS로 전달
    cache 30                        # 캐시 설정
    loop                            # 루프 감지
    reload                          # 설정 변경 자동 감지
}
```

## DNS 조회 확인

```yaml
# 파드 내부 DNS 설정 확인
$ kubectl run -it --rm --restart=Never test-pod --image=busybox -- cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

# search 도메인 덕분에 짧은 이름으로도 조회 가능
$ host web-service                           # → web-service.default.svc.cluster.local
$ host web-service.default                   # → web-service.default.svc.cluster.local
$ host web-service.default.svc.cluster.local # → 동일
```

# 14. Ingress

## 왜 Ingress가 필요한가?

외부에서 클러스터 안의 서비스에 접근하게 하려면 NodePort나 LoadBalancer 타입 서비스를 사용할 수 있습니다. 하지만 서비스마다 LoadBalancer를 만들면 클라우드 비용이 급격히 늘어나고 URL 경로 기반으로 다른 서비스로 라우팅하는 것도 어렵습니다.

**Ingress**는 이 문제를 해결합니다. 클러스터 앞에 하나의 진입점을 두고 URL 경로나 호스트 이름에 따라 내부의 적절한 서비스로 트래픽을 라우팅합니다. HTTP/HTTPS 레이어에서 동작하며 SSL 종료도 처리할 수 있습니다.

## 구성 요소

Ingress는 두 가지로 구성됩니다.

- **Ingress Controller**: 실제 트래픽을 처리하는 컨트롤러입니다. 기본으로 설치되지 않으므로 별도로 배포해야 합니다. (Nginx, Traefik, HAProxy 등)
- **Ingress Resource**: 라우팅 규칙을 정의하는 쿠버네티스 오브젝트입니다.

## Ingress Controller 배포

Nginx Ingress Controller를 예시로 배포 구성을 살펴봅니다.

```yaml
# nginx 설정을 저장하는 ConfigMap
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
```

```yaml
# Ingress Controller Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      serviceAccountName: ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
```

```yaml
# 외부에서 Ingress Controller에 접근하기 위한 Service
apiVersion: v1
kind: Service
metadata:
  name: ingress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    name: http
  - port: 443
    targetPort: 443
    name: https
  selector:
    name: nginx-ingress
```

ServiceAccount는 Ingress Controller가 쿠버네티스 API를 통해 Ingress Resource를 읽고 설정을 반영하기 위해 필요합니다.

## Ingress Resource 작성

### 단순 라우팅 (모든 트래픽을 하나의 서비스로)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  backend:
    serviceName: wear-service
    servicePort: 80
```

### 경로 기반 라우팅 (URL 경로에 따라 다른 서비스로)

`/wear`로 오면 쇼핑 서비스, `/watch`로 오면 영상 서비스로 라우팅합니다.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - http:
      paths:
      - path: /wear
        backend:
          serviceName: wear-service
          servicePort: 80
      - path: /watch
        backend:
          serviceName: watch-service
          servicePort: 80
```

### 호스트 기반 라우팅 (도메인에 따라 다른 서비스로)

`wear.my-online-store.com`으로 오면 쇼핑 `watch.my-online-store.com`으로 오면 영상 서비스로 라우팅합니다.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - host: wear.my-online-store.com
    http:
      paths:
      - backend:
          serviceName: wear-service
          servicePort: 80
  - host: watch.my-online-store.com
    http:
      paths:
      - backend:
          serviceName: watch-service
          servicePort: 80
```

---

# 15. Ingress - Annotations & rewrite-target

## rewrite-target이 필요한 이유

Ingress에서 `/pay` 경로로 들어온 요청을 `pay-service`로 전달할 때 서비스 내부 앱이 `/pay`라는 경로를 모를 수도 있습니다. 앱 입장에서는 `/`로 요청이 들어와야 정상 동작하는 경우가 많습니다. 이때 `rewrite-target` 어노테이션을 사용하면 Ingress가 경로를 바꿔서 전달합니다.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /pay
        backend:
          serviceName: pay-service
          servicePort: 8282
```

> 외부 요청: `http://host/pay` → 내부 서비스: `http://pay-service:8282//pay`가 `/`로 다시 쓰여서 전달됩니다.
> 

---

# 16. Gateway API

## 기존 Ingress의 한계

Ingress는 단순한 HTTP 라우팅에는 잘 동작하지만 실제 운영 환경에서는 여러 한계가 드러납니다.

**멀티 테넌시 문제**: 여러 팀이 하나의 클러스터를 공유할 때 각 팀이 자신의 Ingress 규칙을 독립적으로 관리하기 어렵습니다. 하나의 Ingress 리소스를 여러 팀이 공유하면 충돌이 발생합니다.

**HTTP만 지원**: Ingress는 HTTP/HTTPS 기반 라우팅만 지원합니다. TCP, gRPC 같은 프로토콜을 처리하려면 컨트롤러별 어노테이션을 사용해야 하는데 이는 이식성이 없습니다.

**컨트롤러 의존적 설정**: Nginx, Traefik 등 컨트롤러마다 다른 어노테이션을 사용해야 해서 컨트롤러를 바꾸면 설정도 전부 바꿔야 합니다.

## Gateway API의 해결책

Gateway API는 기존 Ingress의 이 모든 한계를 해결하기 위해 설계된 차세대 표준입니다. 핵심 아이디어는 **역할에 따른 책임 분리**입니다.

| 오브젝트 | 관리 주체 | 역할 |
| --- | --- | --- |
| **GatewayClass** | 인프라 제공자 | 어떤 컨트롤러(구현체)를 사용할지 정의 |
| **Gateway** | 클러스터 운영자 | 실제 게이트웨이 인스턴스, 리스너(포트/프로토콜) 설정 |
| **HTTPRoute / TCPRoute 등** | 애플리케이션 개발자 | 트래픽 라우팅 규칙 정의 |

그럼 자신의 역할에 해당하는 오브젝트만 관리하면 됩니다. 인프라 팀은 GatewayClass를, 운영팀은 Gateway를 개발팀은 HTTPRoute를 각자 독립적으로 관리할 수 있어 멀티 테넌시 문제가 해결됩니다.

## Gateway API의 장점

- **다양한 프로토콜 지원**: HTTPRoute, TCPRoute, gRPCRoute 등 다양한 라우트 타입 제공
- **표준화된 설정**: 어노테이션 없이 표준 스펙으로 트래픽 분할, CORS 등 고급 기능 설정 가능
- **이식성**: 컨트롤러가 바뀌어도 라우팅 규칙은 그대로 유지
- **광범위한 지원**: AWS, Azure, Nginx, Traefik 등 주요 컨트롤러들이 Gateway API를 구현하고 있음
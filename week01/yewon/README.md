# Section2 : Core Concept

스터디 날짜: 2026년 1월 9일

# Kubernetes Architecture

![image.png](image.png)

- Master
    - 클러스터 관리, 노드 정보 저장, 컨테이너 이동 계획, 컨테이너 모니터링 등 작업 수행
    - kube-apiserver : master만 가짐
    - etcd : 모든 정보는 master의 key-value store에 저장
- Worker nodes
    - kublelet : worker node가 가지고 있음
    - Kube-proxy : 타 노드들과 통신에 사용
- Scheduler : 컨테이너 리소스 요구 사항, 노드 용량, 오차, 규약 등에 따라 올바른 노드 식별
- Kube Controller Manager :
    - Node-Controller, Replication-Controller
- Kube-apiserver : 클러스터 내의 모든 작업을 오케스트레이션
    - 주기적으로 kubelet으로부터 정보를 가져와 상태 모니터링
- container runtime engine : 컨테이너를 실행 가능하게 하는 SW

# Components

1. API server : K8dml front-end 역할. 모든 통신은 API server를 통한다.
2. etcd : key-value store. cluster 관리에 사용되는 key 저장소. 모든 node, master의 정보를 저장. master끼리 충돌하지 않게 관리
3. Scheduler : 여러 노드의 작업/컨테이너 배포 역할. 새로 생성된 컨테이너에 node 할당.
4. Controller : orchestration의 brain 역할. 노드 다운을 감지해서 새로운 컨테이너를 불러오는 등 → 대응
5. Container Runtime : Container 실행의 기본 software → Docker 사용
6. Kubelet : cluster의 각 node에서 실행되는 agent

## 1. kube-apiserver

- K8의 기본 관리 컴포넌트
- kube명령어 실행 → apiserver가 요청 인증, 유효성 검사 → cluster에서 데이터 검색, 요청 응답
- Authenticate User, Validate request, retrieve data, update etcd, scheduler, kubelet
    - scheduler, kubelet 등은 apiserver를 통해 각 영역에서 클러스터 업데이트 수행
- etcd와 직접 상호 작용하는 유일한 요소
    - etcd server option으로 etcd server의 위치 지정

## 2. etcd

- 분산형 키 값 저장소(distributed reliable key-value store) → simple, secure, fast
    - key-value store
    
    ![image.png](image%201.png)
    
    - etcd 명령어로 key-value 생성 및 저장 가능 → etcd server에서 확인 가능
- nodes, pods, configs, secrests, accounts, roles, bindings, others 등 내용 저장
- 설치(사용) 방법
    1. 처음부터 설정
        1. client url → 기본 포트 : 2379 → etcd 서버 접속 위해서 kube-apiserver에 구성해야하는 url
    2. kube ADM 도구 사용 배포 → 모의고사 환경
        1. etcd서버를 시스템 namespace에 pod로 배포
- 초기 세팅 정보
    - etcdctl 명령어가 버전에 따라 다르니 버전 확인/설정 필요
        - 버전 세팅 안하면 2가 기본 → 3으로 설정 필요
            
            ```yaml
            //version3으로 설정
            export ETCDCTL_API=3
            ```
            
    - 인증서 파일 경로 설정 필요

## 3. kube-scheduler

- 어떤 노드에 어떤 파드를 배치할 지 결정
    - 배치 결정은 배치 규칙에 따라서 → 규칙은 custom도 가능
    - 실제로 배치를 하지 않음 → 실제 배치는 kubelet

## 4. kube-controller-manager

- K8의 다양한 컨트롤러 관리
    1. watch status (모니터링)
    2. remediate situation (상황에 맞는 대처)
- 여러 종류가 있음
    - Node-Controller, Replication-Controller 등

## 5. Container Runtime

- Container Runtime : Container 실행의 기본 software → Docker 사용

## 6. kubelet

- 각 노드의 선장
    - status 보고, 모니터링, pod 배치 등, 이미지 가져와서 인스턴스 실행 요청 등

## + kube-proxy

- K8 클러스터의 각 노드에서 실행되는 프로세스
    - 클러스터 전체의 모든 노드에서 서비스(데이터베이스)에 엑세스 할 수 있게 함

---

# Concept with code

## Pod

- K8이 워커 노드에 배포하는 컨테이너를 캡슐화 → Pod(Kubernetes 오브젝트)
- K8에서 생성할 수 있는 사장 작은 오브젝트
- 일반적으로 파드와 컨테이너는 1:1 관계
    - 같은 종류가 아니라면 컨테이너가 여러 개인 경우도 있음 → e.g. hepler container
    - 동일한 네트워크 공간 공유, 직접 통신 가능, 동일 저장 공간 공유 가능

```yaml
// pod 생성
kubectl run nginx --image nginx

// 클러스터 파드 목록 확인
kubectl get pods

// 파드에 대한 자세한 정보 확인
kubectl describe pod (파드 이름)
```

### Pod with YAML

- pod, replica, deployment, service 등의 오브젝트 생성을 위한 입력으로 YAML 사용
- 구성
    
    ```yaml
    // 루트 속성(필수 필드)
    apiVersion: v1
    kind: Pod
    metadata:
    	name: myapp-pod
    	labels:
    		app: myapp
    		type: front-end
    
    spec:
    	containers:
    		- name: nginx-container
    			image: nginx
    ```
    
    - apiVersion / kind (string)
        
        ![image.png](image%202.png)
        
    - metadata (dictionary)
    - containers (List/Array)

```yaml
// YAML파일 설정 적용/생성
kubectl create -f (파일이름).yml
```

### Lab

![image.png](image%203.png)

- READY → 포드에 준비된 컨테이너 수 / 포드에 있는 총 컨테이너 수

```yaml
// pod 삭제
kubectl delete pod (포드이름)
```

- YAML 파일 생성 및 적용

```yaml
kubectl run redis --image=redis123 --dry-run=client -o yaml > redis-definition.yaml

kubectl create -f redis-definition.yaml

kubectl get pods
```

```yaml
kubectl edit pod (포드이름)

// 저장 및 나가기
:wq

kubectl apply -f redis.yaml
```

---

## ReplicaSets

- 만드는 법
    
    ```yaml
    apiVersion: v1
    kind: ReplicationController
    //rc
    metadata:
    	name: myapp-rc
    	labels:
    		app: myapp
    		type: front-end
    
    //rc
    spec:
      - template:
    		  //pod
    			metadata:
    				name: 
    				labels:
    					app:
    					type:
    				//pod
    				spec:
    					containers:
    					- name:
    						image: 
        replica: 3
    ```
    
- 생성, 확인
    
    ```yaml
    kubectl create -f rc-definition.yml
    
    kubectl get replicationcontroller
    
    kubectl get pods
    ```
    
- 만드는 법
    
    ```yaml
    apiVersion: apps/v1
    kind: ReplicaSet
    metadata:
    	name: myapp-replicaset
    	labels:
    		app: myapp
    		type: front-end
    
    spec:
    	  template:
      
    			metadata:
    				name: myapp-pod
    				labels:
    					app: myapp
    					type: front-end
    				spec:
    					containers:
    					- name: nginx-container
    						image: nginx
        replica: 3
        selector: 
    	    matchLabels:
    		    type: front-end
    ```
    
    - selector : 이 세트가 어떤 파드에 해당하는지
        - 레플리카 세트 생성으로 생기지 않은 파드도 관리 가능하기 때문에
        - 많은 파드들 중에서 같은 설정의 파드들만 모니터링 가능
- 생성, 확인
    
    ```yaml
    kubectl create -f replicaset-definition.yml
    
    kubectl get replicaset
    
    kubectl get pods
    ```
    
- commands
    
    ```yaml
    //파일 수정
    kubectl replace -f replicaset-definition.yml
    
    //replica 수정
    kubectl scale --replicas=6 -f replicaset-definition.yml
    
    -> 갯수만 수정한거고 파일은 자동 업데이트 되지 않음
    ```
    

---

## Deployment

- 배포와 관련한 사항 컨트롤

```yaml
// 모든 정보 확인
kubectl get all
```

- 유용한 명령어들

```yaml
//Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)

kubectl run nginx --image=nginx --dry-run=client -o yaml

//Generate Deployment YAML file (-o yaml). Don’t create it(–dry-run) and save it to a file.

kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml

//In k8s version 1.19+, we can specify the --replicas option to create a deployment with 4 replicas.

kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > nginx-deployment.yaml
```

- vi 편집기 사용법

![image.png](image%204.png)

## Services

- 애플리케이션 내/외부의 다양한 구성 요소 간의 통신 지원
    1. NodePort : 노드간의 포트 수신, 요청을 파드에 전달
    2. ClusterIP : 클러스터 내부에 가상의 IP 생성, 프론트/백앤드 서버 등 다른 서비스 간의 통신 제공
    3. LoadBalancer : 클라우드 제공 업체에서 app을 위한 로드 밸런서 프로비저닝 

### 1. NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
	name: myapp-service

spec:
		type: NodePort
		ports:
				//pod꺼 (지정 안하면 targetPort와 같게)
		  - targetPort: 80
			  // 서비스꺼 (필수)
			  port: 80
			  //노드꺼, 지정 안하면 무료 포트 할당
			  nodePort: 30008
		//pod의 내용 가져옴, 연결될 pod를 지정
		selector:
			app: myapp
			type: front-end
```

### 2. ClusterIP

- 파드를 그룹화, 그룹 내 파드에 엑세스할 수 있는 단일 인터페이스 제공
- 각 서비스에 클러스터 내의 IP와 이름 할당. 다른 경로에서 서비스에 엑세스할 때 사용.

```yaml
apiVersion: v1
kind: Service
metadata:
	name: back-end

spec:
		type: ClusterIP
		ports:
				//pod꺼 (지정 안하면 targetPort와 같게)
		  - targetPort: 80
			  // 서비스꺼 (필수)
			  port: 80

		//pod의 내용 가져옴, 연결될 pod를 지정
		selector:
			app: myapp
			type: back-end
```

### 3. LoadBalancer

- 사용자가 서비스에 접속하기 위한 단일 URL 필요
    - → 로드 밸런서 용도로 새 가상 머신 생성 → 적절한 로드 밸런서 설치 빛 구성 → 기본 노드로 트래픽을 라우팅 → 외부 부하 분산 기능 설정 및 유지 관리
        - 클라우드 플랫폼의 기본 로드 밸런서 활용 가능
- K8은 특정 클라우드 공급자의 기본 로드 밸런서 통합을 지원

## Namespaces

- 각 환경을 구분 짓고, 자원을 각각 할당
    - 기본 값은 default
- default, kube-public, kube-system 등의 종류가 있음

### DNS

- 같은 네임스페이스의 리소스는 이름만으로 호출 가능

```java
mysql.connect("db-service")
```

- 다른 네임스페이스의 리소스 호출은 서비스의 DNS 전체 이름 입력

```java
mysql.connect("db-service.dev.svc.cluster.local")
```

- command

```java
// 특정 namespace에 있는 pod 조회 방법
kubectl get pods --namespace=(이름)

// 특정 namespace에 pod 생성
kubectl create -f pod-definition.yml --namespace=(이름)

--namespace -> -n/ns?

//namespace 생성 방법
kubectl create -f namespace-dev.yml

kubectl create namespace dev
```

```java
kubectl config set-context $(kubectl config current-context) --namespace=dev
```

## Imperative(명령형) vs Declarative(선언형)

### Impresive

- 생성, 삭제 명령어를 사용
- yaml 파일을 쓰지 않으니 빠르다.
- 실행 기록이 사라짐 → 세션 기록에서만 확인 가능
    - create objects
        
        ```java
        kubectl run --image=nginx nginx
        kubectl create deployment --image=nginx nginx
        kubectl expose deployment nginx --port 80
        ```
        
    - update objects
        
        ```java
        kubectl edit deployment nginx
        kubectl scale deployment nginx --relicas=5
        kubectl set image deployment nginx nginx=nginx:1.18
        ```
        
    - Imperative Object Configuration Files : create objects
        
        ```java
        kubectl create -f nginx.yaml
        ```
        
    - Imperative Object Configuration Files : update objects
        
        ```java
        kubectl edit deployment nginx
        kubectl replace -f nginx.yaml
        kubectl replace --force -f nginx.yaml
        ```
        

### Declarative

- apply 명령어 사용
    - object 없으면 생성까지 해줌
    - 3개의 파일에서 다 적용? 먼말이지 ㅜ
    - create objects
        
        ```java
        kubectl apply -f nginx.yaml
        kubectl apply -f (파일 경로)
        ```
        
    - update objects
        
        ```java
        kubectl apply -f nginx.yaml
        ```
        

## Command

```java
// 모든 리소스 나열
kubectl api-resources
-> 여기서 대소문자 확인 가능
-> 확인한 단어로 explain 사용 가능

kubectl explain pods
// 특정 부분 설명
kubectl explain pods.spec
// 해당 리소스에서 사용할 수 있는 전체 필드 목록 출력
kubectl explain pods --recursive
```

# Options

- `run` : 빠른 Pod 실행/테스트 (임시성, 디버깅)
- `create` : 리소스 생성의 기본기 (파일/imperative 둘 다)
- `expose` : 기존 리소스를 Service로 노출 (Service 생성기)
- `kubectl api-resources | grep …` : 클러스터가 인식하는 리소스 종류 목록 필터링해서 보여줌
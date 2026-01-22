# Questions

## List

### 1. 만약에 노드 어피니티, 테인트, 톨러레이션 설정이 꼬여서 파드가 올라갈 수 없는 상태라면 어떻게 될까?
- 왜 이 파드가 안뜨지? 디버깅해야하는 실전 문제의 순서 맞추기
- 순서대로 체크리스트
    1. get pod 명령어로 파드 상태 보기 (pending)

    2. 해당 pod의 이벤트 보기 kubectl describe pod <pod-name>

    3. 노드의 taint 확인하기 kubectl describe node <target-node>

    4. pod의 toleration 확인하기 kubectl describe pod <pod-name>

        
        
### 2. 실무에서 라벨, 셀렉터를 그룹핑해서 사용한다고 하는데 어떤 형식이나 규칙으로 사용할까?
- **리소스 관리, 모니터링, 비용 산정, 배포 자동화의 핵심 키(Key)** 역할
- 하나의 파드(Pod)에 기술적 정보, 비즈니스 정보, 보안 정보 등을 조합하여 붙입니다. ⇒ 결국 셀렉터로 찾기 쉽게 만드는게 핵심!
- 쿠버네티스 커뮤니티와 도구(Helm, ArgoCD, Prometheus 등)들이 공통으로 사용하는 **표준 라벨**
    
    | **키 (Key)** | **설명** | **예시** |
    | --- | --- | --- |
    | **`app.kubernetes.io/name`** | 애플리케이션의 이름 | `mysql` |
    | **`app.kubernetes.io/instance`** | 해당 앱의 고유 인스턴스 식별자 (보통 릴리스 이름) | `mysql-abc` |
    | **`app.kubernetes.io/version`** | 애플리케이션의 버전 (이미지 태그와 일치 권장) | `5.7.21` |
    | **`app.kubernetes.io/component`** | 아키텍처 내에서의 역할 | `database`, `frontend` |
    | **`app.kubernetes.io/part-of`** | 전체 시스템의 이름 (상위 개념) | `wordpress` |
    | **`app.kubernetes.io/managed-by`** | 리소스를 관리하는 도구 | `helm`, `kustomize` |
    
### A. 환경 및 구성 (Environment & Configuration)
배포된 환경을 구분하여 ConfigMap이나 Secret을 분리하거나, 실수로 운영 환경을 건드리지 않도록 합니다.
    
- `environment`: `dev`, `stage`, `prod`, `dr` (재해복구)
- `region`: `ap-northeast-2`, `us-east-1`
- `tier`: `frontend`, `backend`, `cache`
    

### B. 조직 및 소유권 (Ownership & Cost)
    
비용 정산(FinOps)이나 장애 발생 시 담당 팀을 찾기 위해 사용합니다.
    
- `team`: `payment-team`, `search-team`
- `owner`: `hong-gildong`
- `cost-center`: `1234-56` (부서 코드)


### C. 배포 및 버전 관리 (Release & Versioning)
    
카나리 배포나 블루/그린 배포 시 트래픽 제어를 위해 사용합니다.
    
- `release-track`: `stable`, `canary`
- `release-method`: `blue`, `green`


### D. 보안 및 규정 (Security & Compliance)
    
보안 수준에 따라 네트워크 정책(NetworkPolicy)을 적용할 때 사용합니다.

- `data-sensitivity`: `high`, `low`, `pci-dss`
- `internet-facing`: `true`, `false`


### 3. 실무에서 특정 노드에 pod 배치하는거 신경쓰는 규칙을 어떤 기준으로 정할까? (리소스 문제 등)
1. 고가용성: 물리적인 서버 한 두개가 죽어도 서비스는 유지되어야 하기 때문 (AZ 나눠서 배치)
2. 워크로드 격리: 작업 타입에 따라서 성격이 다른 작업 pod의 특성을 이해하고 배치
    - e.g.) 배치 서버 pod와 실시간 응답이 중요한 WAS pod가 같은 노드에 떠있으면 cpu/memory 사용량에 영향을 받아서 WAS 응답에 문제가 생길수도 있음
3. 비용: 클라우드 비용을 줄이기 위해 인스턴스 유형과 pod 특성에 따라 배치
    - e.g.) 인스턴스 타입을 고려할 때 (온디맨드와 스팟 타입) & 인스턴스 사양 (cpu와 메모리 스펙)


### 4. 쿠버네티스 환경이 다양한 곳에서 스케줄링과 pod 배치는 어떤 식으로 구성해야 할까? (심화 질문, AWS EKS / IDC 온프레미스 / 다른 클라우드사)
- 인프라 추상화를 잘하는게 필요
    - "노드 라벨 표준화 (Normalization)” + 명확한 컨벤션을 통해 일관적으로 리소스 관리하는게 필요
- 환경별 스케줄링 전략 (이원화 운영 필요)
    - AWS EKS: 비용 효율과 탄력성 중심 - 탄력적으로 scailing이 가능하기 때문에 비용 아끼는게 제일 중요
    - IDC 온프레미스: 자원 효율과 우선순위 중심 - 자원 스펙이 정해져있어 유동적으로 늘릴 수 없기 때문에 우선순위를 잘 정해서 운영하는게 중요
- 데이터 지역성(Data Locality)
    - 쿠버네티스 물리적인 노드가 멀리 떨어져서 존재한다면 네트워크 왕복 비용 소모된다
    - 빠른 데이터 처리와 상호 데이터 교환이 잦은 경우는 물리적으로 가까운 서버에 배치할 수 있다. (같은 클러스터, 같은 region에 배치)
    - 특히 온프레미스에서 AWS EKS 등으로 옮기는게 비용 많이 들기 때문에 되도록 하나의 환경에서 일관되게 처리해주는게 좋다.


### 5. CKA 빈출 유형
- 전용 노드에 pod 띄우는 문제 → pod에 **tolerations + requiredAffinity** 추가
- 왜 이상한 노드에 떴지? (목적지 노드 엉뚱한 곳에 뜬 문제) → required가 아니라 preferred를 씀
- affinity 맞게 설정했는데 여전히 pending 상태인 문제 → pod toleration 누락 가능성


### 6. Daemonset의 특징을 잘 생각해보고, 실무에서 어떤 경우에 쓰일지 추측해보기
실무적으로 DaemonSet을 사용하는 용도
    

- 로그 수집기: `fluentd`, `filebeat`
- 모니터링 에이전트: `prometheus-node-exporter`
- 네트워크 플러그인: CNI 관련 에이전트
- 노드 레벨 보안/트레이싱 도구
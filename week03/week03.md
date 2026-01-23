# Logging & Monitoring
## 1. Monitoring Cluster Components
- 모니터링 할 수 있는 것
  - 노드 수준 : 클러스터의 노드 수, 정상 노드 수
  - 성능 메트릭 : CPU, 메모리, 네트워크, 디스크 사용률
  - 파드 수준 메트릭 : 파드의 수, 각 파드의 CPU 및 메모리 사용량
- 모니터링 솔루션 : Metrics Server
  - K8 클러스터 하나당 metrics server 1개
  - 인메모리 모니터링 솔루션 → 디스크에 메모리를 저장하지 X (과거 실적 데이터를 볼 수 X)
- Kubelet의 C 어드바이저가 파드의 성능 메트릭 검색 → kubelet api를 통해 메트릭 서버에 노출
```bash
// metrics server 가져오기
git clone [metircs server url]

// 메트릭 활성화를 위해 포드, 서비스 및 역할 집합 배포 → 서버를 사용해서 클러스터이 노드에서 성능 메트릭 폴링
kubectl create -f deploy/1.8+/

// 클러스터의 성능 확인 가능
kubectl top node
```
## 2. Application Logs
```bash
// docker의 실시간 로그 추적 가능
docker logs -f ecf
```
```bash
kubectl logs -f event-simulator-pod event-simulator
```
# Application Lifecycle Management

## 1. Rolling Updates and Rollbacks in Deployments
![img1](img/img1.png)
- 처음 배포할 때 롤 아웃이 트리거 → 새 롤아웃은 새 replicaset 생성 → 새 배포 리비전이 만들어짐 → 배포에 대한 변경 사항 추적 가능
```bash
// 롤아웃 상태 확인 가능
kubectl rollout status deployment/myapp-deployemnt
// 히스토리 확인
kubectl rollout history deployment/myapp-deployment
```
- Rolling Update : 기본 배포 업데이트 전략
  - 컨테이너를 하나씩 내리고 올려서 업데이트 함
- Rollbacks : 업데이트를 되돌리고 싶을 때 사용 가능
  ```bash
  kubectl rollout undo deployment myapp-deployment
  ```
## 2. Configure Application
## 3. Scale Application
## 4. Self-Healing Application
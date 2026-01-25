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



---
# Cluster Maintainance
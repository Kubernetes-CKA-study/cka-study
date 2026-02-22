# Section 8: Storage

## Docker Storage
| 구분      | 스토리지 드라이버                 | 볼륨 드라이버                                 |
| --------- | --------------------------------- | --------------------------------------------- |
| 역할      | 이미지와 컨테이너의 스토리지 관리 | Docker 볼륨(영구 저장소) 관리                 |
| 대상      | Docker의 계층화 파일 시스템       | 데이터 영속성을 위한 외부 스토리지            |
| 예시      | AUFS, Overlay2, Device Mapper 등  | NFS, AWS EBS, Azure Disk 등 (플러그인 기반)   |
| 동작 방식 | 컨테이너 레이어 관리 중심         | 스토리지 드라이버와 별도로 동작               |
| 관리 주체 | Docker 엔진 내부                  | 볼륨 드라이버 플러그인(volume driver plugins) |
### Storage Drivers
모든 작업(레이어 유지, RW 레이어 관리, Copy-on-Write 등)을 수행하는 것이 바로 Storage Driver

- 주요 드라이버 : AUFS, BTRFS, ZFS, Device Mapper, Overlay, Overlay2
- Docker는 OS에 맞는 최적의 드라이버를 자동 선택
- 드라이버마다 성능/안정성 특성이 다르므로, 환경에 맞는 선택이 필요

**Docker 데이터 저장 위치**
- Docker 설치 시 기본 데이터 디렉터리:  
  **`/var/lib/docker`**
- 주요 하위 디렉터리
  - `images/` : 이미지 관련 데이터
  - `containers/` : 컨테이너 관련 데이터
  - `volumes/` : Docker 볼륨 데이터

→ **이미지·컨테이너·볼륨의 모든 실제 데이터는 여기 저장됨**


![img1](img/img1.png)

**Docker의 계층형(Layered) 아키텍처**
- Dockerfile의 **각 명령어 = 하나의 이미지 레이어**
- 예시 레이어 구조 (아래 → 위)
  1. Base Image (Ubuntu)
  2. OS 패키지 설치
  3. 언어/라이브러리 설치 (Python, Flask)
  4. 애플리케이션 소스 코드
  5. Entrypoint / CMD
- 특징
  - 각 레이어는 **변경된 내용만 저장**
  - 이미지 레이어는 **읽기 전용(Read-only)**
  
![img2](img/img2.png)
**이미지 계층 (읽기 전용)**
- Docker 빌드 완료 후 **수정 불가**
- `docker build` 명령어로만 변경 가능
- 여러 컨테이너가 공유

**컨테이너 계층 (읽기/쓰기)**

- 컨테이너 실행 시 이미지 위에 생성
- 로그 파일, 임시 파일, 사용자 수정 파일 저장
- **컨테이너 삭제 시 함께 삭제**

![img3](img/img3.png)
**Copy-on-Write**
- 이미지 레이어 파일은 **읽기 전용**
- 컨테이너에서 특정 파일 수정 시 RW 레이어에 복사, 수정은 RW 레이어에서 진행
→ 원본 이미지 레이어는 항상 동일하게 유지됨

#### Volume Mount vs Bind Mount
- Volume Mount : `/var/lib/docker/volumes` 하위의 Docker 관리 볼륨
- Bind Mount : 호스트 파일시스템의 특정 경로를 직접 연결
```bash
[Volume Mount]
# 볼륨 생성
docker volume create data_volume
# 볼륨 마운트
docker run -v data_volume:/var/lib/mysql mysql
# data_volume2가 없으면 Docker가 자동 생성
docker run -v data_volume2:/var/lib/mysql mysql

[Bind Mount]
# 호스트 디렉토리 직접 마운트
docker run -v /data/mysql:/var/lib/mysql mysql
```
**--mount : 추천 마운트 방식**
```bash
# --mount 옵션 사용 (더 명확함, Key=Value 형식)
# source: 호스트에서의 위치
# target : 컨테이너에서의 위치
docker run --mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql
```

**+ Container Storage Interface (CSI)**

![img5](img/img5.png)

### Volume Drivers
#### Local Volume Driver
- Docker가 기본 제공하는 드라이버
  - 드라이버명: `local`
  - 저장위치: `/var/lib/docker/volumes/`
  - 용도: Docker 호스트 로컬 디스크에 데이터 저장
```bash
# 기본 로컬 볼륨 생성
docker volume create my_volume

ls /var/lib/docker/volumes/
```
#### Volumes
**Docker에서**
- Docker 컨테이너 : 일시적(transient)
  - 필요할 때 실행되어 데이터를 처리하고, 작업이 끝나면 제거됨
  - 컨테이너 내부 데이터도 컨테이너와 함께 삭제됨

→ 데이터를 유지하려면 컨테이너 실행 시 **볼륨(volume)** 을 붙여야 함

**Kubernetes에서**
- Kubernetes Pod : 일시적
    - Pod 생성 → 데이터 처리 → Pod 삭제 시 데이터도 삭제됨
→ 데이터를 유지하려면 Pod에 볼륨을 붙임
```yaml
# 환경: 단일 노드 Kubernetes 클러스터
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - name: alpine
    image: alpine
    command: ["/bin/sh", "-c"]
    args:
    - shuf -i 0-100 -n 1 >> /opt/number.out
    # 컨테이너 내부의 /opt 디렉토리에 볼륨 마운트
    volumeMounts:
    - name: data-volume
      mountPath: /opt
  volumes:
  - name: data-volume
    # 저장되는 데이터는 호스트의 /data 디렉토리에 저장됨
    hostPath:
      path: /data
      type: Directory
```
→ pod가 삭제되어도 호스트의 `/data` 디렉토리에 데이터는 남아있음

**외부 스토리지(Replicated Cluster Storage)**

- `hostPath`는 단일 노드에서만 적합
- 다중 노드 환경에서는 각 노드의 `/data` 디렉토리가 서로 다름

→ 동일 데이터 유지하려면 외부 스토리지(Replicated Cluster Storage) 필요
  - Kubernetes가 지원하는 다양한 스토리지 옵션: NFS, iSCSI, GlusterFS, CephFS, 클라우드 스토리지(AWS EBS, Azure Disk/File, Google Persistent Disk) 등

#### Persistent Volume
대규모 환경에서는 수많은 pod 배포 필요 → 모든 Pod 정의 파일에 해당 스토리지 구성을 직접 기입해야 함 → 비효율

**해결 방법**
- Persistent Volume (PV) : 클러스터 전역에서 사용 가능한 스토리지 풀
- 사용자는 Persistent Volume Claim(PVC) 을 통해 필요한 스토리지를 할당 받음
```yaml
# PV 기본 템플릿
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  # 접근 모드
  accessModes:
    - ReadWriteOnce
  capacity:
    # 스토리지 용량
    storage: 1Gi
  hostPath:
    path: "/tmp/data"
```
- accessModes (접근 모드):
  - `ReadOnlyMany` : 여러 노드에서 읽기 전용
  - `ReadWriteOnce`  : 단일 노드에서 읽기/쓰기 가능
  - `ReadWriteMany` : 여러 노드에서 읽기/쓰기 가능

- `kubectl create -f pv-definition.yaml` : PV 생성
- `kubectl get persistentvolume` : PV 조회

#### Persistent Volume Claims
- PVC는 사용자가 필요한 스토리지를 요청하는 방법
>관리자(Admin) → Persistent Volume을 생성 (스토리지 풀 제공)
>사용자(User) → Persistent Volume Claim을 생성 (스토리지 요청)

- PVC가 생성되면 K8은 PV 속성과 PVC 요청을 고려하여 1:1 바인딩
  - 고려되는 속성: 용량, 접근 모드, 볼륨 모드, 스토리지 클래스
  - 특정 PV를 선택하려면 Label + Selector 를 사용해 지정 가능
  - 적합한 PV가 없는 경우 : PVC는 `Pending` 상태 유지
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```
- `kubectl create -f pvc-definition.yaml` : PVC 생성
- `kubectl get persistentvolumeclaim`, `kubectl get pvc` : PVC 조회

**Reclaim Policy : PVC 삭제 시 PV 동작**

| Reclaim Policy       | Description                                                                       |
| -------------------- | --------------------------------------------------------------------------------- |
| Retain               | PV와 해당 데이터가 유지됨                                                         |
| Delete               | PV와 관련 스토리지 리소스가 삭제됨                                                |
| Recycle (Deprecated) | 데이터를 정리한 후 PV를 다시 사용 가능 상태로 만듦 (현재는 더 이상 사용되지 않음) |

#### Storage Classes
**정적 프로비저닝(Static Provisioning)**
PV 생성 전에 관리자가 PV를 미리 만들어야 하는 방식 → 비효율적
e.g. 애플리케이션에서 스토리지가 필요할 때마다
1. Google Cloud에서 디스크를 수동으로 생성
2. 동일한 이름으로 PV 정의 파일 작성

**동적 프로비저닝(Dynamic Provisioning)**
PVC가 생성될 때 K8이 자동으로 PV를 생성하는 방식 → 효율적
- StorageClass : 동적 볼륨 프로비저닝 지원
- PVC 생성 시
  - 스토리지 벤더 API 호출 → PV(디스크) 자동 생성
  - PV와 PVC 자동 바인딩

```yaml
# StorageClass 예시
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
# 스토리지 백엔드 지정
provisioner: kubernetes.io/gce-pd
# 옵션
parameters:
  type: pd-standard
  replication-type: none
```
```yaml
# PVC에서 StorageClass 사용
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  # 여기서 어떤 StorageClass를 사용할지 지정
  storageClassName: standard
```
# Section 9: Networking

## Before Networking
### Linux Networking Basics
### DNS
### Network Namespaces

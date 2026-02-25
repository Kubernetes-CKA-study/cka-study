- **Storage Drivers**: 이미지와 컨테이너의 레이어 구조를 관리
- **Volume Driver Plugins**: 실제 데이터가 저장되는 공간을 관리

# 1. 도커의 기본 폴더 구조

도커를 설치하면 호스트 시스템의 `/var/lib/docker` 경로에 데이터가 저장됩니다.

- **`aufs`**: 스토리지 드라이버 관련 데이터
- **`containers`**: 컨테이너 관련 정보 저장
- **`image`**: 다운로드한 이미지 레이어 저장
- **`volumes`**: 도커가 관리하는 볼륨 데이터 저장

# 2. 계층형 아키텍처 (Layered Architecture)

도커는 이미지를 빌드할 때 **계층형(Layered) 아키텍처**로 구성합니다. Dockerfile의 각 명령어(`RUN`, `COPY` 등)는 이전 레이어와의 변경 사항만을 담은 새로운 레이어를 생성합니다.

```sql
레이어 5: 엔트리포인트 업데이트       ← 변경분만 저장
레이어 4: 소스 코드
레이어 3: pip 패키지 변경 내용
레이어 2: apt 패키지 변경 내용
레이어 1: 기본 Ubuntu 레이어
```

### 캐시를 활용한 빌드 최적화

두 번째 애플리케이션이 첫 번째 애플리케이션과 동일한 베이스(Ubuntu, apt 패키지, pip 패키지)를 사용한다면, 도커는 레이어 1~3을 **새로 빌드하지 않고 캐시에서 재사용**합니다. 소스 코드와 엔트리포인트가 다른 레이어 4~5만 새로 빌드합니다.

- 빌드 속도가 빠르고 디스크 공간을 효율적으로 사용할 수 있습니다.
- 애플리케이션 코드를 업데이트할 때도 마찬가지로 이전 레이어들은 캐시에서 가져오고, 변경된 최신 소스 코드 레이어만 새로 빌드합니다.

### 1) 읽기 전용 이미지 레이어 (Read-Only Image Layers)

`docker build` 명령어로 빌드가 완료된 이미지 레이어들은 모두 읽기 전용(Read-Only)입니다. 빌드 이후에는 내용을 수정할 수 없으며 수정하려면 새로운 빌드를 시작해야 합니다.

### 2) 쓰기 가능한 컨테이너 레이어 (Writable Container Layer)

`docker run` 명령어로 컨테이너를 실행하면 도커는 읽기 전용 이미지 레이어들 맨 위에 얇은 **쓰기 가능한 레이어**를 하나 추가합니다.

- 컨테이너 내부에서 생성되는 로그 파일, 임시 파일 등은 모두 이 레이어에 저장됩니다.
- **같은 이미지로 만든 모든 컨테이너는 이미지 레이어를 공유합니다.**
- 컨테이너가 삭제되면 쓰기 가능 레이어도 함께 파괴되어 데이터가 유실됩니다.

### 3) Copy-on-Write (CoW) 메커니즘

이미지 레이어에 있는 기존 파일을 수정하고 싶을 때 어떻게 할까요? 도커는 **Copy-on-Write** 전략을 사용합니다.

1. 수정하려는 파일을 이미지 레이어(Read-only)에서 컨테이너 레이어(Writable)로 복사해 옵니다.
2. 복사본을 수정합니다.
3. 컨테이너는 이제 이미지 레이어의 원본 대신 컨테이너 레이어의 수정본을 보게 됩니다.

이미지 레이어가 읽기 전용이라는 것은 **이미지 자체 파일이 수정되지 않는다**는 의미입니다. `docker build`로 새로 빌드하기 전까지 이미지는 항상 동일한 상태를 유지합니다.

# 3. Volumes

컨테이너가 삭제되어도 데이터를 유지하려면 **볼륨(Volume)**을 사용해야 합니다.

## 1) 볼륨 생성

```bash
$ docker volume create data_volume
```

`$ docker volume create data_volume`

- `/var/lib/docker/volumes/` 하위에 `data_volume` 디렉토리가 생성됩니다.

```bash
$ ls -l /var/lib/docker/volumes/
drwxr-xr-x 3 root root 4096 Aug 01 17:53 data_volume

$ docker volume ls
DRIVER    VOLUME NAME
local     data_volume
```

## 2) 볼륨 마운트

```bash
$ docker run -v data_volume:/var/lib/mysql mysql
```

- `볼륨이름:컨테이너내경로` 순서로 작성합니다.
- 컨테이너가 삭제되어도 `data_volume`의 데이터는 그대로 유지됩니다.
- 볼륨을 미리 생성하지 않아도 됩니다. `docker run` 실행 시 볼륨이 없으면 자동으로 생성합니다.

```bash
$ docker run -v data_volume2:/var/lib/mysql mysql  # 볼륨 없어도 자동 생성

$ docker volume ls
DRIVER    VOLUME NAME
local     data_volume
local     data_volume2
```

→ 볼륨마운트를 해야한다!!

## 3) 바인드 마운트

호스트 시스템의 **어디든지(어떤 경로든)** 내가 원하는 폴더를 컨테이너와 연결합니다. 예를들어 이미 호스트의 특정 경로에 데이터가 있다면, 그 경로를 직접 컨테이너에 마운트할 수 있습니다. 볼륨 마운트와 달리 `/var/lib/docker/volumes/`가 아닌 **호스트의 임의 경로**를 사용합니다.

- 호스트에 존재하는 데이터를 컨테이너에 제공하거나 소스 코드를 실시간으로 컨테이너에 반영할 때 유용합니다.

```bash
$ mkdir -p /data/mysql
$ docker run -v /data/mysql:/var/lib/mysql mysql  # 완전한 경로를 지정
```

### 볼륨 마운트 vs 바인드 마운트 정리

| 구분 | 볼륨 마운트 | 바인드 마운트 |
| --- | --- | --- |
| 저장 위치 | `/var/lib/docker/volumes/` | 호스트의 임의 경로 |
| 관리 주체 | Docker | 사용자 |
| 주요 용도 | 데이터 영속화, 컨테이너간 공유 | 호스트 파일 공유, 소스코드 실시간 반영 |

**4) --mount 옵션** 

```bash
# 바인드 마운트 방식
$ docker run --mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql

# 볼륨 마운트 방식
$ docker run --mount type=volume,source=data_volume,target=/var/lib/mysql mysql
```

# 04. Volume Driver Plugins in Docker

스토리지 드라이버가 레이어 구조를 관리한다면 **볼륨 드라이버**는 실제 데이터가 저장되는 공간을 관리합니다. 기본 드라이버는 `local`이며 데이터를 `/var/lib/docker/volumes/`에 저장합니다.

로컬 서버가 아닌 외부 스토리지나 클라우드 서비스를 연결할 때는 서드파티 볼륨 드라이버 플러그인을 사용합니다.

- **클라우드 스토리지**: Amazon EBS, Azure File Storage, Google Compute Persistent Disks
- **공유 파일 시스템**: GlusterFS, Flocker, VMware vSphere Storage, RexRay

# 05. Storage Drivers

계층형 아키텍처 유지, 쓰기 가능 레이어 생성, Copy-on-Write 처리 등 **레이어 관련 모든 작업을 담당**하는 것이 바로 스토리지 드라이버입니다.

- AUFS
- ZFS
- BTRFS
- Device Mapper
- Overlay
- **Overlay2** ← 현재 가장 권장되는 드라이버

스토리지 드라이버의 선택은 **운영체제(OS) 환경에 따라 다르며** 도커가 자동으로 가장 적합한 드라이버를 선택합니다.

# 06. Container Storage Interface (CSI)

도커의 볼륨 드라이버 개념은 쿠버네티스 환경에서 **CSI(Container Storage Interface)** 라는 표준 규격으로 발전했습니다. CRI(Container Runtime Interface), CNI(Container Network Interface)와 함께 쿠버네티스의 **3대 표준 인터페이스** 중 하나입니다.

과거에는 새로운 저장 장치를 지원하려면 쿠버네티스 소스 코드를 직접 수정해야 했습니다. CSI는 표준화된 API를 제공하여 스토리지 업체가 쿠버네티스 코드를 건드리지 않고도 자신들의 솔루션을 연동할 수 있게 합니다.

- **표준화된 RPC 호출**: 오케스트레이터와 스토리지 드라이버 간에 RPC(Remote Procedure Call) 메서드로 통신합니다. (볼륨 생성, 삭제, 복제 등)
- **멀티 환경 지원**: 클라우드(AWS, GCP, Azure)부터 온프레미스(NFS, Ceph)까지 동일한 방식으로 관리합니다.
- **플러그인 방식**: CSI 규격에 맞춰 드라이버만 제공하면 어떤 컨테이너 환경에서도 즉시 사용 가능합니다.

# 7. Volumes (쿠버네티스)

파드(Pod)에 직접 볼륨을 붙이는 가장 기본적인 방법입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: app
    image: alpine
    volumeMounts:
    - mountPath: /data
      name: my-volume
  volumes:
  - name: my-volume
    hostPath:
      path: /tmp/data
      type: Directory
```

- `hostPath`는 노드의 로컬 디렉토리를 마운트합니다. 단일 노드 테스트 환경에는 적합하지만, 멀티 노드 환경에서는 권장하지 않습니다.
- 멀티 노드 환경에서는 NFS, AWS EBS 등 외부 스토리지를 사용해야 데이터의 일관성을 보장할 수 있습니다.

# 8. Persistent Volumes (PV)

## 왜 PV가 필요한가?

규모가 큰 환경에서는 수많은 파드를 배포합니다. 이때 각 파드마다 스토리지를 일일이 설정해야 하는 문제가 생깁니다.

- 어떤 스토리지 솔루션을 쓰든 파드를 배포하는 사람이 **모든 파드 정의 파일에 스토리지를 직접 설정**해야 합니다.
- 변경사항이 생기면 모든 파드 정의 파일을 일일이 수정해야 합니다.

이 문제를 해결하기 위해 **Persistent Volume(PV)** 개념이 등장했습니다.

> **PV란?** 관리자가 클러스터 전체에서 사용할 수 있도록 미리 구성해둔 스토리지 볼륨의 풀(Pool)입니다. 사용자는 이 풀에서 **Persistent Volume Claim(PVC)** 을 통해 필요한 스토리지를 가져다 씁니다.
> 

## PV 생성 예시

```bash
# pv-definition.yaml

kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

```bash
$ kubectl create -f pv-definition.yaml
persistentvolume/pv-vol1 created

$ kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
pv-vol1   1Gi        RWO            Retain           Available                          3min

$ kubectl delete pv pv-vol1
persistentvolume "pv-vol1" deleted
```

---

# 9. Persistent Volume Claims (PVC)

## PVC란?

- **Volumes**와 **Persistent Volume Claim**은 쿠버네티스에서 **별개의 오브젝트**
- PVC가 생성되면 쿠버네티스는 요청 내용과 볼륨의 속성을 기반으로 **적절한 PV를 찾아 자동으로 바인딩**합니다.
- 조건에 맞는 PV가 없으면 PVC는 **Pending** 상태가 됩니다.

## PVC 생성 예시

```yaml
# pvc-definition.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi`
```

## PV → PVC 바인딩 흐름

```yaml
# 1. PV 먼저 생성
$ kubectl create -f pv-definition.yaml
persistentvolume/pv-vol1 created

$ kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
pv-vol1   1Gi        RWO            Retain           Available                          10s

# 2. PVC 생성
$ kubectl create -f pvc-definition.yaml
persistentvolumeclaim/myclaim created

# 처음엔 Pending 상태 (바인딩 중)
$ kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myclaim   Pending                                                     35s

# 잠시 후 Bound 상태로 변경
$ kubectl get pvc
NAME      STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myclaim   Bound    pv-vol1   1Gi        RWO                           1min
```

## PVC / PV 삭제

```yaml
$ kubectl delete pvc myclaim
$ kubectl delete pv pv-vol1
```

---

# 10. Using PVC in PODs

- 파드는 PVC를 **볼륨처럼** 사용하여 스토리지에 접근합니다.
- PVC는 **파드와 같은 네임스페이스**에 존재해야 합니다.
- 클러스터는 파드의 네임스페이스에서 PVC를 찾고 그 PVC에 연결된 PV를 찾아 파드에 마운트합니다.

| 오브젝트 | 스코프 |
| --- | --- |
| Persistent Volume (PV) | 클러스터 전체 (Cluster-scoped) |
| Persistent Volume Claim (PVC) | 네임스페이스 (Namespace-scoped) |

## PV → PVC → Pod 전체 흐름

### 1) PV 생성

```yaml
# pv-definition.yaml

kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

```yaml
$ kubectl create -f pv-definition.yaml
```

### 2) PVC 생성

```yaml
# pvc-definition.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```yaml
$ kubectl create -f pvc-definition.yaml
```

### 3) Pod에서 PVC 사용

```yaml
`# pod-definition.yaml

apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: web
  volumes:
    - name: web
      persistentVolumeClaim:
        claimName: myclaim   # PVC 이름 지정`
```

```yaml
$ kubectl create -f pod-definition.yaml

# 전체 상태 한번에 확인
$ kubectl get pod,pvc,pv
```

---

# 11. Storage Class

## Static Provisioning의 문제점

지금까지 배운 방식(PV 수동 생성)은 **Static Provisioning(정적 프로비저닝)** 입니다.

- AWS, GCP, Azure 같은 클라우드 스토리지를 쓰려면 PV 정의 전에 **클라우드에서 디스크를 먼저 수동으로 생성**해야 합니다.
- 파드 정의 파일에 PV를 쓸 때마다 매번 이 작업을 반복해야 합니다.

## Dynamic Provisioning (동적 프로비저닝)

**Storage Class**를 사용하면 PVC가 생성되는 순간 스토리지를 **자동으로 프로비저닝**할 수 있습니다.

- PV를 미리 수동으로 만들 필요가 없습니다.
- StorageClass가 PVC 요청을 감지하고 자동으로 PV를 생성하고 바인딩합니다.

## StorageClass 생성 예시

```yaml
# sc-definition.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
```

```yaml
$ kubectl create -f sc-definition.yaml
storageclass.storage.k8s.io/google-storage created

$ kubectl get sc
NAME             PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
google-storage   kubernetes.io/gce-pd   Delete          Immediate           false                  20s
```

## StorageClass를 사용하는 PVC

`storageClassName`을 지정하면, PVC 생성 시 StorageClass가 자동으로 PV를 만들어 연결합니다.

```yaml
# pvc-definition.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: google-storage   # StorageClass 이름 지정
  resources:
    requests:
      storage: 500Mi
```

```yaml
$ kubectl create -f pvc-definition.yaml
```

## PVC를 사용하는 Pod

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: frontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: web
  volumes:
    - name: web
      persistentVolumeClaim:
        claimName: myclaim
```

```yaml
$ kubectl create -f pod-definition.yaml
```
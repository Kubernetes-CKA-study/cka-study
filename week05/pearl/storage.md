# Storage
## 핵심 내용 Summary
### 핵심 개념
  - PV: 클러스터 리소스, 실제 스토리지 조각(관리자 생성 가능)
  - PVC: 파드가 요청하는 스토리지(사용자 생성)
  - StorageClass: 동적 프로비저닝 정책(스토리지 종류/옵션)
  - AccessModes: RWO, ROX, RWX (스토리지 타입에 따라 지원 다름)
  - ReclaimPolicy: Retain, Delete, Recycle(deprecated)

### PV vs PVC 비교
| 항목 | PV (PersistentVolume) | PVC (PersistentVolumeClaim) |
| --- | --- | --- |
| 관점 | 관리자/클러스터 리소스 | 사용자/네임스페이스 리소스 |
| 역할 | 실제 스토리지 조각 정의 | 필요한 스토리지 요청 |
| 생성 주체 | 관리자(또는 StorageClass) | 사용자/애플리케이션 |
| 바인딩 | PVC 조건에 맞는 PV 선택됨 | 조건에 맞는 PV에 바인딩됨 |
| 파드 사용 | 직접 사용하지 않음 | 파드가 마운트 대상으로 사용 |

### 프로비저닝
  - 실제 볼륨을 만들어주려고 준비 생성하는 과정을 프로비저닝이라고 함
  - Static: PV를 미리 만들어두고 PVC가 매칭
  - Dynamic: PVC가 StorageClass를 통해 PV 자동 생성
  - 기본 SC 설정: storageclass.kubernetes.io/is-default-class=true

### 볼륨 타입 (CKA 범위)
  - emptyDir: 노드 로컬 임시, 파드 삭제 시 데이터 사라짐
  - hostPath: 노드 로컬 경로, 테스트용
  - nfs: 공유 파일시스템 (RWX)
  - persistentVolumeClaim: PVC 마운트
  - (클라우드): awsEBS, gcePersistentDisk, azureDisk 등

### PVC 매칭 규칙
  - 용량 >= 요청
  - AccessMode 지원
  - StorageClass 일치(또는 둘 다 없음)
  - Label selector 조건(있다면)


## Volumes
- 쿠버네티스에서 볼륨은 파드가 쓰는 저장 공간이다. 
- 컨테이너 파일시스템과 분리돼서 파드 재시작에도 데이터를 유지할 수 있고(종류에 따라), 스토리지(로컬/네트워크/클라우드)를 파드에 마운트하는 추상화라고 생각할 수 있다.
- 도커에서 컨테이너는 일시적이기 때문에, 컨테이너를 삭제할 경우 컨테이너에서 사용한 데이터가 유실될 수 있다.
    - 이런 단점을 보완하기 위한 기술이 볼륨이다.
- 볼륨을 사용하면, 컨테이너에 의해 처리된 데이터가 볼륨에 배치되기 때문에 데이터가 영구적으로 유지된다.


### Volumes & Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
    - image: alpine
      name: alpine
      command: [ "/bin/sh", "-c" ]
      args: [ "shuf -i 0-100 -n 1 >> /opt/number.out;" ]
      volumeMounts:
        - mountPath: /opt
          name: data-volume
  volumes:
    - name: data-volume
      hostPath:
        path: /data
        type: Directory
```

- `spec.volumeMounts` 필드를 통해 데이터 볼륨을 컨테이너 내의 /opt 디렉토리에 마운트한다.
- `volumes.hostPath` 필드를 사용하여 디렉토리를 구성했기 때문에 호스트에는 볼륨에 대한 스토리지 공간이 있다. 이는 단일 노드에서는 정상적으로 작동하지만 다중 노드 클러스터에서의 사용은 적합하지 않다.
- 다중 노드 클러스터 환경에서는 파드가 모든 노드에서 `/data` 디렉토리를 사용하고, 모든 노드가 동일한 데이터를 가질 것이기 때문이다.
- `volumes.hostPath` 필드를 사용하여 다중 노드 클러스터 환경을 구성해야 한다면 AWS EBS와 같은 솔루션을 사용할 수 있다.


## Persistent Volumes

- 퍼시스턴트 볼륨은 애플리케이션을 배포하는 사용자가 사용하도록 구성한 클러스터 전체 스토리지 볼륨 풀이다.
- 관리자가 큰 풀의 스토리지를 생성하고, 사용자들이 필요한 만큼 조각으로 사용하는 방식이다.
- 퍼시스턴트 볼륨을 사용하기 위해서는 퍼시스턴트 볼륨 클레임이 필요하다. 퍼시스턴트 볼륨 클레임을 사용하여 스토리지 볼륨 풀에서 퍼시스턴트 볼륨을 선택할 수 있다.

### Create Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolume
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

- 아래 명령어를 통해 퍼시스턴트 볼륨을 생성할 수 있다.
  ```bash
  kubectl create -f pv-definition.yaml
  ```
- 아래 명령어를 통해 퍼시스턴트 볼륨 목록을 조회할 수 있다.
  ```bash
  kubectl get persistentvolume
  ```

## Persistent Volume Claims

- 퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임은 쿠버네티스 네임스페이스에서 서로 다른 오브젝트이다.
- "관리자"는 퍼시스턴트 볼륨을 생성하고, "사용자"는 스토리지에 사용할 퍼시스턴트 볼륨 클레임을 생성한다.
- 퍼시스턴트 볼륨 클레임이 생성되면 쿠버네티스는 퍼시스턴트 볼륨을 생성할 때의 구성에 따라 퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임을 바인딩한다. 
    - 이때, 모든 퍼시스턴트 볼륨 클레임은 하나의 퍼시스턴트 볼륨과 마운트된다.
- 바인딩 동안 쿠버네티스는 클레임에 의해 요청되는 대로 충분한 용량을 가진 퍼시스턴트 볼륨이나 다른 기타 요청 속성(Access Mode, Volume Modes, Storage Class, Selector, etc)을 찾으려고 한다.
    - 여러 개의 퍼시스턴트 볼륨 구성에서 매칭되는 클레임이 하나라면 라벨과 셀렉터를 통해 매칭시킬 수 있다.
- 퍼시스턴트 볼륨의 용량은 큰데, 퍼시스턴트 볼륨 클레임의 용량이 작다면 바인딩 될 수 있다. 하지만 이 둘은 일대일 관계이므로 다른 클레임이 퍼시스턴트 볼륨의 남은 용량을 사용할 수는 없다.
    - 사용 가능한 볼륨이 없는 경우, 클러스터에서 새 볼륨을 사용할 수 있게 될 때까지 Pending 상태가 된다.

### Create Persistent Volume Claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

- 아래 명령어를 통해 퍼시스턴트 볼륨 클레임을 생성할 수 있다.
  ```bash
  kubectl create -f pvc-definition.yaml
  ```
- 아래 명령어를 통해 퍼시스턴트 볼륨 클레임 목록을 조회할 수 있다.
  ```bash
  kubectl get persistentvolumeclaim
  ```

### Delete Persistent Volume Claim

- `persistentVolumeReclaimPolicy` 필드를 통해 퍼시스턴트 볼륨 클레임 삭제 이후 퍼시스턴트 볼륨을 관리할 수 있다.
  - Retain: 기본 설정으로, 관리자가 삭제하기 전까지 퍼시스턴트 볼륨은 남아 있다.
  - Delete: 퍼시스턴트 볼륨 클레임이 삭제되면 곧 퍼시스턴트 볼륨도 삭제된다.
  - Recycle: 다른 클레임이 사용할 수 있도록 퍼시스턴트 볼륨 내 데이터만 삭제한다.

### Using PVCs in Pods

```yaml
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
          name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

- 퍼시스턴트 볼륨 클레임이 생성되면, 파드 매니페스트 파일의 `volumes.persistentVolumeClaim` 필드를 명시하여 퍼시스턴트 볼륨 클레임을 지정할 수 있다. 이는 레플리카셋이나 디플로이먼트에서도 동일하다.


## Storage Class

- 정적 프로비저닝은 관리자가 디스크를 먼저 만들고 PV에 수동 매칭해야 한다.
- StorageClass는 이 과정을 자동화해, PVC가 생기면 PV를 자동 생성하는 동적 프로비저닝을 제공한다.
- 예) GCP에서는 StorageClass가 필요할 때마다 새 Persistent Disk를 만들어 PV로 연결한다.

### Create Storage Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
```

- 스토리지 클래스가 생성되면 스토리지가 자동 생성된다.
- `spec.storageClassName` 필드를 지정하여 퍼시스턴트 볼륨 클레임에서 스토리지 클래스를 사용할 수 있다.
- 스토리지 클래스는 GCP에 필요한 크기의 새 디스크를 프로비저닝한 다음 퍼시스턴트 볼륨을 생성한 뒤, 퍼시스턴트 볼륨 클레임을 해당 볼륨에 바인딩한다.
- 스토리지 클래스를 사용한다고 해서 퍼시스턴트 볼륨이 생성되지 않는 것은 아니다. 직접 생성할 필요가 없는 것이다.

## CKA 추가 체크포인트 (Storage)

- `volumeBindingMode`: `WaitForFirstConsumer`는 파드 스케줄 시점에 바인딩되어 존/노드 토폴로지 이슈를 줄인다.
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: fast-ssd
  provisioner: kubernetes.io/gce-pd
  volumeBindingMode: WaitForFirstConsumer
  ```
- PVC 확장: `allowVolumeExpansion: true`가 필요하며 `requests.storage`를 늘려 확장한다.
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: expandable
  provisioner: kubernetes.io/gce-pd
  allowVolumeExpansion: true
  ```
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: data
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  ```
- StorageClass 핵심 필드: `provisioner`(CSI/클라우드), `parameters`로 디스크 타입/성능 등을 지정한다.
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: gp3
  provisioner: ebs.csi.aws.com
  parameters:
    type: gp3
    iops: "3000"
    throughput: "125"
  ```

# Persistent Volumes
- 관리자가 provisioning하거나 storage class를 사용하여 동적으로 provisioning한 cluster의 storage
- Cluster Resource(Cluster level)
## Persistent Volume Claim
- 사용자의 storage에 대한 요청, Claim 검사
- ReadWriteOnce, ReadOnlyMany, ReadWriteMany 등으로 Access mode를 요청할 수 있다.
- 사용자가 추상화된 storage resource를 사용할 수 있지만, PV가 필요한 것이 일반적
- PV resource를 consume
- Namespace Level

## Volume & Claim Lifecycle
### Provisioning
- Static
    - Cluster manager가 여러 PV를 만듬
    - Cluster 사용자가 사용할 수 있는 실제 storage의 detail을 제공
    - k8s API에 존재하며 사용 가능
- Dynamic
    - static PV가 사용자의 PVC와 일치하지 않으면 cluster가 volume을 dynamic provisioning(storage class 기반)
    - PVC는 storage class를 요청해야 하고, manager는 dynamic provisioning이 발생하도록 해당 class 생성 및 구성해야 함

### Binding
- Master의 control loop가 새로운 PVC를 watch, 일치하는 PV를 찾아 binding
- PV가 새 PVC에 dynamic provisioning된 경우 항상 해당 PV를 binding
- PVC : PV = 1 : 1(ClaimRef 사용:양방향 바인딩)
- 일치하는 Volume이 없다면 Claim은 Binding되지 않은 상태로 남아 있음(제공되면 binding)

### Using
- Pod는 Claim을 Volume으로 사용
- cluster가 Claim을 검사 및 binding된 Volume을 찾고, Pod에 mount
- 한 번 바인딩 되면 필요없어질 때 까지 해당 사용자에 귀속
- Pod에 volumes block에 persistentVolumeClaim을 작성하여 Pod Schedule, Claim한 PV에 접근

### Protect stroage object
- PVC에 binding된 Pod와 PV가 사용 중인 PVC를 system에서 삭제되지 않게 함

## PersistentVolume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity: # storage size가 설정, 요청할 수 있는 유일한 resource
    storage: 5Gi
  volumeMode: Filesystem # Filesystem, Block 지원
  # FileSystem: volume이 Pod의 directory에 mount
  # Block: raw block device로 사용
  accessModes: # ReadWriteOnce(하나의 노드에서 마운트), ReadOnlyMany, ReadWriteMany -> 여러 노드에서 마운트
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle # Retatin(수동), Recycle(재활용), Delete(삭제)
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```
### affinity
- Node Affinity를 지정하여 해당 volume에 접근할 수 있는 Node 제약 조건 정의 가능

### Phase
- Available: 아직 claim에 binding되지 않은, 사용할 수 있는 resource
- Bound: Volume이 Claim에 binding됨
- Released: Claim은 삭제, Cluster에서 resource 반환은 아직
- Faield: volume 자동 반환 실패

### PersistentVolumeClaim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce # PV와 동일
  volumeMode: Filesystem # PV와 동일
  resources:
    requests: # 특정 수량의 resource 요청 가능
      storage: 8Gi
  storageClassName: slow # storage class 이름 지정을 통해 특정 class 요청 가능(동일한 storageClassName을 갖는 PV만 binding 가능)
  selector: # volume set을 추가로 필터링하기 위해 지정. 일치하는 volume만 claim에 binding
    matchLabels: # volume에 있어야 하는 label
      release: "stable"
    matchExpressions: # key, value의 list, key와 관련된 연산자를 지정하여 만든 requirements. In,NotIn, Exists, DoesNotExist
      - {key: environment, operator: In, values: [dev]}
```
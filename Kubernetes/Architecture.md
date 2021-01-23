# Node
- 각 Node는 Control Plane을 가지고 있다.(HA를 위해 Master/Slave로 구조로 클러스터 구성시에는 Master에만)
- Node의 Component에는 kubelet, container runtime, kube-proxy가 포함
## 노드 추가 방법
- Node의 kubelet으로 Control Plane에 자체 등록
- User가 Node Object를 수동으로 추가

## 노드 상태
| Node 상태 | 설명 |
|----------|-----|
|Ready     |노드 상태 양호 및 파드 수용 준비 완료-> `True`, 상태 불량 및 파드 수용 불가 -> `False`, Node Controller가 일정 기간동안 응답 받지 못한 경우 -> `Unknown`|
|DiskPressure| Disk 용량이 넉넉치 않음 -> `True`|
|MemoryPressure| Memory가 넉넉치 않음 -> `True`|
|PIDPressure| 너무 많은 Process 존재 -> `True`|
|NetworkUnavailable| 네트워크가 올바르게 구성되지 않은 경우 -> `True`|
- Ready 중 pod-eviction-timeout 보다 길게 Unknown, False인 경우 Node 상의 모든 Pod가 삭제되도록 스케줄링
- 노드에 접근이 불가한 경우 apiserver가 kubelet과 통신이 불가하므로 재개될 때까지 Pod 삭제에 대한 결정이 kubelet에 전해질 수 없음
- 그 사이 삭제되도록 스케줄되어진 Pod가 분할된 Node 상에서 동작할 수도 있음

## Node Controller
- Node의 다양한 측면을 관리하는 Control Plane Component
- 역할
    - CIDR 할당이 사용토록 설정된 경우 할당
    - available machine list를 최신화
    - Node 동작 상태 모니터링

# Controller
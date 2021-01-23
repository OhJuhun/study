# Workloads
- Kubernetes에서 구동되는 Application
- Kubernetes는 workload를 Pod 집합 내에서 실행한다.
## built-in workload resources
- Deployment, ReplicaSet
    - Deployment의 모든 Pod가 필요 시 교체, 상호 교체가 가능한 경우 Stateless Application Workload를 관리하기 적합
- StatefulSet
    - state를 가지는 하나 이상의 Pod 동작
    - 지속적으로 데이터 기록할 경우 PersistentVolume과 연계하여 StatefulSet 실행 가능
- DaemonSet
    - Node 수에 맞게 각 Node에서 Pod 정의
- Job, Cron Job
- CRD 사용시 3rd party workload resource 추가 가능

# Pod
- Kubernetes에서 생성, 관리할 수 있는 배포 가능한 가장 작은 컴퓨팅 단위
- 하나 이상의 Container Group(보통 하나의 Container만 사용하기를 권장함)
- Storage, Network 공유
## LifeCycle
- Pending
- Running
    - Container 중 하나 이상이 OK
- Succeded / Faield
    - Container의 성공 여부
### 수명
- self-healing이 안된다.
- 스케줄된 후 Fail, 작업 자체 Fail 시 Pod 삭제
### 단계
| 값 | 의미 |
|---|----|
|pending| 승인되었지만, Container 설정 및 실행 준비가 안되었다.+이미지 다운로드 시간 포함|
|running | Pod가 Node에 Binding, 모든 Container 생성, 적어도 하나의 Container가 실행 중 또는 시작, 재시작|
|succeeded | 성공적으로 종료 및 재시작 되지 않을 것|
| failed | 모든 컨테이너가 종료, 하나 이상의 Container가 Fail|
|Unknown | Pod 통신 오류|
|ready| 왜 설명이 없지|
    
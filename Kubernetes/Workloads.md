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
## Lifecycle
- Pending
- Running
    - Container 중 하나 이상이 OK
- Succeded / Faield
    - Container의 성공 여부
### Lifetime
- self-healing이 안된다.
- 스케줄된 후 Fail, 작업 자체 Fail 시 Pod 삭제
### Phase
| 값 | 의미 |
|---|----|
|pending| 승인되었지만, Container 설정 및 실행 준비가 안되었다.+이미지 다운로드 시간 포함|
|running | Pod가 Node에 Binding, 모든 Container 생성, 적어도 하나의 Container가 실행 중 또는 시작, 재시작|
|succeeded | 성공적으로 종료 및 재시작 되지 않을 것|
| failed | 모든 컨테이너가 종료, 하나 이상의 Container가 Fail|
|Unknown | Pod 통신 오류|

### Condition
- Pod는 하나의 PodStatus를 가지며, Passed, Not Passed Condition에 대한 PodCondition array를 가짐
    - PodScheduled : Pod가 Node에 Scheduled
    - ContainersReady: Pod의 `모든 Container`가 Ready
    - Initialized: 모든 Init Container가 성공적으로 started
    - Ready: Pod가 요청을 처리할 수 있고 일치하는 모든 Service의 Load balancing pool에 추가되어야 한다.

### Container Probe
- Container에서 kubelet에 의해 주기적으로 수행되는 diagnostic
- kubelet이 diagnostic을 위해 호출하는 Handler
    - ExecAction: Container 내에서 지정된 명령어 실행. exit code 0 => 성공
    - TCPSocketAction: 지정된 포트에서 Container의 IP 주소에 대해 TCP 검사 수행. 포트가 활성화되어 있다 => 성공
    - HTTPGetAction: 지정한 포트 및 경로로 Container의 IP Address에 대한 HTTP Get. 200 <= code <400 => 성공
#### Probe 결과
- Success: 성공
- Failure: 실패
- Unknown: diagnostic 자체가 실패. 아무런 액션도 수행되면 안됨
### Probe 종류(https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/ 여기에 추가정보, 한글은 없음)
- livenessProbe
    - Container 동작 여부
    - Probe 실패 후 Container가 종료 또는 재시작되길 원하는 경우 사용
    - 실패시 kubelet이 Container를 죽이고 restartPolicy의 대상이 됨
    - 미제공시 Success
- readinessProbe
    - 요청을 처리할 준비가 되었는지 여부
    - Probe가 성공한 경우에만 Pod에 Traffic을 전송하길 원하는 경우 사용
    - Container가 대량의 Data, configuration files, startup 시 migration 처리시 필요
    - 실패시 EndPoint Controller가 Pod와 연관된 모든 Service의 End-Point에서 Pod의 IP 주소 제거
    - initialDelay 이전 기본 상태는 Failure
    - 미제공시 Success
- startupProbe
    - Container 내의 Application의 시작 여부
    - Service 시작까지 오랜 시간이 걸리는 Container가 있는 Pod에서 사용
    - 성공할 때까지 다른 Probe 비활성
    - 실패시 kubelet이 Container를 죽이고 restartPolicy에 의해 처리
    - 미제공시 Success


# Init Container
- Pod의 App Container들이 실행되기 전에 실행되는 특수한 Container
- App Image에 없는 Utility, setup script 등을 포함할 수 있다.
- 실패시 Init container가 성공할 때까지 반복하여 재시작
- restartPolicy가 Never이면서 Pod 시작 동안 Init container 실패시 전체 Pod 실패 처리

## 일반 Container와 차이점
- Init container는 `항상 완료를 목표로 실행된다.`
- 각 Init container는 `다음 Init Container가 시작되기 전에 성공적으로 완료되어야 한다.`
- lifecycle, liveness,readiness,startup Probe 미지원 -> Container Pod가 Ready가 되기 전 완료되어야 하므로

## 예제
- init-myservice, init-mydb가 완료되면 Pod가 시작된다.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

# Workload Resources
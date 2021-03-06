# Workloads
- Kubernetes에서 구동되는 Application
- Kubernetes는 workload를 Pod 집합 내에서 실행한다.
## built-in workload resources
- Deployment, ReplicaSet
    - Deployment의 모든 Pod가 필요 시 교체, 상호 교체가 가능한 경우 Stateless Application Workload를 관리하기 적합
- StatefulSet
    - state를 가지는 하나 이상의 Pod 동작(각 파드에 식별자가 존재)
    - 지속적으로 데이터 기록할 경우 PersistentVolume과 연계하여 StatefulSet 실행 가능
- DaemonSet
    - Node 수에 맞게 각 Node에서 Pod 정의
- Job, Cron Job
- CRD 사용시 3rd party workload resource 추가 가능

# Pod
- Kubernetes에서 생성, 관리할 수 있는 배포 가능한 가장 작은 컴퓨팅 단위
- 하나 이상의 Container Group(보통 하나의 Container만 사용하기를 권장함)
- Storage, Network 공유
## Singleton Pod
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
- ReplicaSet을 확장한 것들이 관리하는 Pod들의 내용이 서로 다를 수 있다.(?)
## Deployment
- Pod와 ReplicaSet에 대한 Declarative Update를 제공
- Deployment에서 `의도하는 상태 설명`, Deployment Controller에서 현재 상태에서 의도하는 상태로 비율을 조정하며 변경
- 예시
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: nginx-deployment # deployment 이름
    labels:
        app: nginx
    spec:
      replicas: 3 # replica pod 수
      selector: # deployment가 관리할 Pod를 찾는 방법 정의
          matchLabels:
          app: nginx # app: nginx인 Pod
      template:
          metadata:
          labels:
              app: nginx #nginx 레이블
          spec:
          containers:
          - name: nginx
              image: nginx:1.14.2
              ports:
              - containerPort: 80
    ```
    - 3개의 nginx Pod를 불러오기 위한 replicaSet
### scaling
- kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
### Deployment 상태
- 새로운 ReplicaSet을 Rollout하는 동안 Progressing, Complete, Fail To Progress 
#### Progressing
- 새로운 ReplicaSet 생성
- 새로운 ReplicaSet Scale Up
- 기존 ReplicaSet Scale Down
- 새 Pod Ready or Available
#### Complete
- 요청한 모든 update가 완료되었을 때
- 관련한 모든 Replica를 사용할 수 있을 때
- 이전 Replica가 실행되고 있지 않을 때
#### Fail To Progress
- 할당량 부족
- readinessProbe 실패
- Image pull Error
- 권한 부족
- 범위 제한(?)
- Application Runtime의 잘못된 구성
- .spec.progressDeadlineSeconds를 지정하여 찾을 수 있음
    - Deployment의 진행이 정지되었음을 나타내는, Deployment Controller가 대기하는 시간

## ReplicaSet
- Pod 집합의 실행을 항상 안정적으로 유지하는 것이 목표
- 명시된 동일 Pod 수에 대한 가용성을 보장하는 데에 사용
- Pod의 metadata.ownerReferences 필드를 통해 Pod에 연결
    - 현재 Object가 소유한 Reosurce 명시
    - 해당 Pod를 소유한 ReplicaSet을 식별하기 위한 소유자 정보를 가짐
- Selector를 통해 새 Pod 식별
- custom update orchestration가 필요하거나, don't require updates at all 의 경우에만 ReplicaSet 사용

## StatefulSet
- Application의 Stateful을 관리하는데 사용하는 워크로드 API Object
- Pod 집합의 Deployment와 Scaling을 관리, Pod의 순서 및 고유성 보장
- 각 Pod의 `독자성`을 유지한다. -> 동일한 Spec이지만 교체가 불가하다.
- Pod가 `re-schedule 간에도 지속적으로 유지되는 식별자`를 가짐
- Storage Volume을 사용해서 Workload에 지속성을 제공하려는 공우, 솔루션의 일부로 StatefulSet을 사용할 수 있다.
- 장애에 취약
### 사용
- 아래에 만족하지 않는다면 Deployment 또는 ReplicaSet을 사용해야 한다.
    - stable, unique network identifier
    - stable, persistent storage
    - ordered, graceful deployment and scaling
    - ordered, automated rolling update

### 제한 사항
- Pod에 지정된 Storage는 Persistent Volume Provisioner를 기반으로 하는 storage class를 요청해서 provision하거나 사전에 provision되어 있어야 한다.
- 삭제 또는 Scale Down해도 연관된 Volume이 삭제되지는 않는다.
- 현재 Pod의 Network 신원을 책임지고 있는 headless service가 필요하다.
- StatefulSet 삭제 시 Pod의 종료에 대한 보증을 제공하지 않는다.
- Pod가 순차적, 정상적으로 종료되려면 삭제전 Scale을 0으로 축소
### 구성요소
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx #StatefulSet의 .spec.template.metadata.labels.app 과 일치해야 한다.
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx #.spec.template.metadata.labels.app 과 일치해야 한다.
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx #.spec.selector.matchLabels.app 과 일치해야 한다.
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

## DaemonSet
- 모든 Node가 Pod의 Replica를 실행하게 한다.
- Node가 Cluster에 추가되면 Pod도 추가된다.
- 주로
    - 모든 Node에서 Cluster Storage Daemon 실행
    - 모든 Node에서 Log 수집 Daemon 실행
    - 모든 Node에서 Node Monitoring Daemon 실행
### 일부 Node에서만 실행
- .spec.template.spec.nodeSelector를 명시하면 DaemonSet Controller는 이와 일치하는 Node에 Pod 생성
- .spec.template.spec.affinity를 명시하면 Node Affinity와 일치하는 Node에 Pod 생성

# Replication Controller
- 언제든지 지정된 수의 Pod Replica가 실행 중임을 보장한다.
- Pod 또는 동일 종류의 Pod Set이 항상 기동되고 사용 가능한지 확인한다.

## 동작 방식
- Pod가 너무 많으면 추가적인 Pod를 제거한다.
- Pod가 너무 적으면 더 많은 Pod를 시작한다.
- Replication Controller가 유지, 관리하는 Pod는 실패, 삭제, 종료 시 자동으로 교체됨

# Kubernetes GC
- 소유자가 없어진 Object를 삭제하는 역할을 한다.
- 일부 Kubernetes Object는 다른 Object의 Owner이다.
- Owner Object에 소유된 Object를 Dependent라고 한다.
- Dependent는 Owner를 나타내는 metadata.ownerReferences field를 가지고 있다.
- replication controller, replicaset, statefulset, daemonset, deployment, job, cronjob에 의해 생성된 Object의 ownerReference 값을 자동으로 설정한다.
## 삭제 방식
- Foreground Cascading
  - root object가 먼저 deletion in progress 상태가 된다.
  - 이 때, Object는 REST API를 통해 볼 수 있고, deletionTimestamp가 설정되며  foregroundDeletion에 metadata.finalizers 값이 포함된다.
  - 또한, 이 상태가 설정되면 GC는 Object의 dependent 항목을 삭제한다.
- Background Cascading
  - Owner Object 즉시 삭제 및 GC는 Background에서 dependent를 삭제한다.

# TTL Controller
- 실행이 완료된 Resource Object 수명을 제한하는 TTL 메커니즘 제공
- 현재는 Job만 처리하며 Custom Resource와 같이 실행을 완료할 다른 Resource를 처리하도록 확장될 수 있다.
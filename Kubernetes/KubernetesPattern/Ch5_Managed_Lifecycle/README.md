# Managed Lifecycle
- Cloud native로 관리되는 컨테이너화된 application은 자기 자신의 수명주기를 제어할 수 없다.
- 좋은 cloud native 일원으로서 자격을 갖추려면(?) 관리 플랫폼에 의해 생성된 event를 받아서 그에 맞춰 수명주기를 조절해야 한다.
- Managed Lifecycle 패턴은 application이 수명주기 이벤트에 어떻게 반응해야 하며, 반응할 수 있는지를 설명한다.

## 문제
- cloud native platform은 application에 명령을 보내고 그 명령에 반응할 것을 기대한다.
- 정책, 외부 요인 등에 따라 cloud native platform은 자신이 관리하는 application에 대해 언제든 시작, 중지를 결정할 수 있어야 한다.
- 어떤 이벤트의 중요 여부, 어떻게 반응해야 하는지를 결정하는 것은 containerized application의 몫이다.(실제로는 API가 수행한다.)
- Application은 수명주기 관리로부터 혜택을 얻거나 이런 서비스가  필요하지 않은 경우, 수명주기 관리를 무시하기도 한다.

## 해결책
### SIGTERM Signal
- k8s가 `container를 멈추기로 결정할 때마다 SIGTERM 신호를 수신`
- 이는 SIGLKILL을 보내기 전에 Container가 깨끗이 종료될 수 있도록 `Container를 슬쩍 찔러보는 것`
- SIGTERM을 받은 Application은 `가능한 한 빨리 멈춰야 한다.`
- 어떤 application은 SIGTERM을 받으면 빨리 종료될 수도 있고, 어떤 경우는 진행 중인 요청 처리->연결 해제-> 임시 파일 삭제 등의 오랜 작업이 걸릴 수도 있다.
- 어떤 경우든 `container를 깨끗하게 종료하기 위한 좋은 순간은 SIGTERM에 반응하는 것`

### SIGKILL Signal
- SIGTERM 수신 후에도 Container Process가 종료되지 않으면 `SIGKILL에 의해 강제 종료`된다.
- SIGTERM 이후 .spec.terminationGracePeriodSeconds(default=30) 만큼 기다린 후 SIGKILL을 보낸다.
- 하지만 k8s 명령이 실행되는 동안 `재정의될 수도 있으므로 보장할 순 없다.`
- 목적은 프로세스를 빠르게 시작하고 멈출 수 있는 Containerized Application을 임시로 설계, 구현하는 것

### lifecycle hooks
- 수명주기 관리를 위해 Process SIGNAL만 사용하는 것은 제한적
- k8s에서는 postStart, preStop 같은 수명주기 Hook을 제공
- 지원하는 handler type
  - exec: container 안에서 직접적으로 명령어 실행
  - httpGet: 하나의 Pod container에 의해 공개된 port로 HTTP GET 요청 수행

#### postStart
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: post-start-hook
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    lifecycle:
      postStart:
        exec:
          command: # 30초를 기다리는 postStart command.(긴 시간 실행되는 시작 코드가 있을 수 있으므로 이에 대한 시뮬레이션)
          - sh
          - -c
          - sleep 30 && echo "Wake up!" > /tmp/postStart_done # 병렬 시작하는 주 application과의 sync를 위해 트리거 파일 사용
```

- Container가 생성된 후 주 컨테이너 프로세스와 async하게 실행된다.
- Blocking call이라 postStart handler가 완료될 때까지는 `container 상태가 waiting`, `Pod 상태가 Pending` 유지
- `주 컨테이너 프로세스의 초기화 시간을 벌기 위해` 컨테이너의 `시작 상태를 지연`시키는 데에 사용된다.
- 또 다른 용도는, Pod가 `어떤 전제조건을 만족시키지 못했을 경우 Container가 시작되지 않게 하는 것`
  - 예를 들어 poststart hook이 0이 아닌 종료 코드를 리턴하면 주 컨테이너 프로세스는 k8s에 의해 죽는다.
- postStart 실행이 보장되지 않으므로 이걸로 중요한 로직을 실행할 때에는 주의해야 함
- hook은 container process와 병렬로 실행되기 때문에 container 시작전에 hook이 실행될 수도 있음 
- at least once 실행이므로 중복 실행될 수도 있음
- handler에 도달하지 못한 실패한 HTTP request 요청에 대해서 재시도를 수행하지 않는다.

#### preStop
- container가 종료되기 전에 container로 전송되는 blocking call
- SIGTERM과 동일한 의미를 지니며, `SIGTERM에 응답하는 것이 불가능할 때 container를 정상적으로 종료하기 위해 사용`해야 한다.
- preStop이 블로킹되거나 계속 진행중이거나 실패한 결과가 리턴되더라도 Container가 삭제되거나 프로세스가 종료되는 상황을 막을 수는 없다.
- application 종료를 위한 SIGTERM의 대안일 뿐
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pre-stpp-hook
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    lifecycle:
      preStop:
        httpGet: # application 내에서 실행중인 8080/shutdown 호출
          port: 8080
          path: /shutdown
```

### 그 외 수명주기 제어
- 위와 같은 Container level이 아닌 Pod level에서 또 다른 메커니즘으로 초기화 명령을 수행할 수 있다.
#### 초기화 컨테이너(Init container)
- 일반 application container와는 달리 Init container는 
  - 순차적으로 실행
  - 완료될 때까지 실행
  - Pod 내 Application container가 실행되기 전에 실행
- 위와 같은 특성으로, Pod level 초기화 작업에 Init Container를 사용할 수 있다.
- Lifecycle Hook과 Init container는 각각 Container level, Pod level에서 서로 다른 granularity로 작동, 서로 바꾸거나 보완해서 사용할 수 있다.

|측면 | Lifecycle Hook | Init Container |
|----|----------------|----------------|
|활성화 단계| Container 수명주기 단계 |  Pod 수명주기 단계 |
| 시작 단계 동작 | postStart | 실행될 Init container 목록 |
| 종료 단계 작업 | preStop | -|
|타이밍 보장 | postStart는 Container ENTRYPOINT와 동시 실행 | application container시작 전에 모든 초기화 컨테이너는 성공해야 함 |
| 사용 사례| Container별 특화된 중요하지 않은 시작/정리 종료 실행 | Container를 사용해 workflow같은 순차작업 수행. <br>작업 실행을 위해 컨테이너 재사용|
- 특정 타이밍 보장 요구 외에 어떤 메커니즘을 사용할 지에 대한 규칙은 없다.
- 이러한 기능들을 사용하여 재사용이 가능하게 해준다.

## 정리
- 이와 같은 이벤트를 잘 처리, 대응함으로써 사용중인 서비스에 미치는 영향을 최소화, 애플리케이션을 정상 시작 및 종료할 수 있다.

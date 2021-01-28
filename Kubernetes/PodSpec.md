# Pod Spec
- Pod Spec에 포함되는 것들 정리
## Pod QoS 클래스 분류
- BestEffort(최하위 우선순위)
- Burstable
- Guranteed
## Request/limit
## restartPolicy
- Pod의 재시작 정책
- Pod의 모든 Container에 적용
- 동일 노드에서 kubelet에 의한 Container 재시작만을 의미
- Always(`default`), OnFailure, Never

## .spec.selector
- Label Selector를 지정하는 필수 필드
- .spec.template.metadata.labels와 일치해야 함

## .spec.strategy
- 이전 Pod를 새로운 Pod로 대체하는 전략
- .spec.strategy.type -> Recreate 또는 RollingUpdate(`Default`)
    - Recreate: 새 Pod 생성 전 모든 Pod가 죽는다.
    - RollingUpdate: Rolling Update 방식으로 업데이트
        - .spec.strategy.rollingUpdate.maxUnavailable: 업데이트 프로세스 중 사용할 수 없는 최대 Pod 수 지정(절댓값 또는 비율)
            - 예를 들어 30%로 설정하면, 시작시 ReplicaSet의 크기를 의도한 Pod 중 70%를 Scale Down 가능
            - 새 Pod가 준비되면 기존 ReplicaSet을 Scale Down할 수 있다.
            - Update 중 항상 사용 가능한 전체 Pod 수는 의도한 파드의 수의 70% 이상 되도록 새 ReplicaSet을 Scale Up할 수 있다.
        - .sepc.strategy.rollingUpdate.maxSurge : 의도한 Pod 수에 대해 생성할 수 있는 최대 Pod 수 지정(절댓값 또는 비율)
            - 업데이트하는 동안 실행하는 총 Pod 수는 최대 의도한 Pod 수의 130%가 되도록 보장

## .spec.progressDaelineSeconds
- Type=Progressing, Status=False의 상태, Resource가 Reason=ProgressDeadlineExceeded 상태로 Fail to Progress 전에 Deployment가 진행되는 것을 대기시키는 시간을 명시
- default: 600s
- .spec.minReadySeconds 보다 커야한다.
- 미래에 자동화된 롤백이 구현된다면 디플로이먼트 컨트롤러는 상태를 관찰하고, 그 즉시 디플로이먼트를 롤백할 것이다.

## .spec.minReadySeconds
- 생성된 Pod의 Container가 어떤 것과도충돌하지 않고 사용할 수 있도록 준비되어야 하는 최소 시간
- default 0s

# Logging
- 각 Node에서 /var/log/containers 아래에 Log가 쌓이게 된다.

# Daemonset
- DaemonSet은 모든 노드에서 파드를 실행하게 한다(일부도 가능)
## 용도
- 모든 노드에서 클러스터 스토리지 데몬 실행
- 모든 노드에서 로그 수집 데몬 실행
- 모든 노드에서 노드 모니터링 데몬 실행

# Toleration
- 보통 Taints와 함께 동작하여 `파드가 부적절한 노드에 스케줄되지 않게 한다.`
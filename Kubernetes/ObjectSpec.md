# Object Spec
- https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#-strong-api-overview-strong-(상세 스펙)
```yaml
apiVersion: apps/v1 # 이 Objeect 생성을 위해 사용하고 있는 kubernetes API 버전
kind: Deployment # Object의 종류
metadata: # 이름, UUID, namespace 등
  name: nginx-deployment
spec: #Object의 상태
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

## Namespace
- Kubernetes에서 제공하는 가상 클러스터
- 여러 팀, 여러 프로젝트에 걸쳐 많은 사용자가 있는 환경에서 사용하도록 만들어짐
- 사용자가 거의 없거나 수십명 정도(?)의 경우 사용할 필요가 없음
### 초기 namespace
- default
- kube-system
    - kubernetes system에서 생성한 Object를 위한 Namespace
- kube-public
    - 자동으로 생성되며, 모든 사용자에게 READ 권한이 있음
    - 관례적인 성격
- kube-node-lease
    - Cluster가 Scaling될 때 Node의 heartbeat 성능을 향상시키는 각 Node와 관련된 lease object에 대한 namespace
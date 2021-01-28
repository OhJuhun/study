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

## Label & Selector
### Label
- Object에 첨부된 key,value 쌍
- Object의 특성을 식별하는 데에 사용되지만, system적인 의미는 없다.
- Label로 하위 집합을 선택, 구성하는 데에 사용할 수 있다.
```json
"metadata": {
  "labels": {
    "key1": "value1",
    "key2": "value2"
  }
}
```
- UI, CLI에서 효율적으로 Query하기에 적합
#### Syntax
- a-z0-9A-Z - _ . 사용 가능
- Label Key는 /로 구분되는, `Prefix/name`의 Segment가 있다.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production #label
    app: nginx #label
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

#### 권장되는 Labels
|Key | desc | ex |
|----|------|----|
|app.kubernetes.io/name | Application name | mysql | 
|app.kubernetes.io/instance | Application Instance name | mysql-abcxyz|
|app.kubernetes.io/version	| 애플리케이션의 현재 버전 |5.7.21|
|app.kubernetes.io/component|아키텍처 내 구성요소 |database|
|app.kubernetes.io/part-of|이 애플리케이션의 전체 이름|wordpress|
|app.kubernetes.io/managed-by|애플리케이션의 작동을 관리하는데 사용되는 도구|helm|
### Label Selector
- 고유한 값이 아니다.
- Client와 User가 Object를 식별할 수 있다.
- 일치성 기준, 집합성 기준 두 Selector가 존재
- 일치성 기준 요건
    - 일치 또는 불일치의 요구사항으로 Label key, value를 filtering
    - =, ==, !=
- 집합성 기준 요건
    - Label 요건에 따라 value 집합을 Key로 filtering
    - in, notin, exists, !

## API
- LIST, WATCH Filtering
    - 불일치 기준 요건: ?labelSelector=environment%3Dproduction,tier%3Dfrontend
    - 집합성 깆ㄴ 요건: ?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend$29
### kubectl
```shell
kubectl get pods -l environment=production,tier=frontend #일치성 기준 요건
kubectl get pods -l 'environment in (production), tier in (frontend)'
```

## Annotation
- 비식별 metadata를 object에 첨부할 수 있다.
- tool, library 같은 client들이 이 metadata를 검색할 수 있다.
- 작거나 크고, 구조적이거나 구조적이지 않을 수 있으며, label에 허용되지 않는 문자를 포함할 수 있다.

## Field Selector
- 한 개 이상의 Resource Field 값에 따라 Kubernetes Resource를 선택하기 위해 사용된다.
```go
metadata.name=my-service
metadata.namespace!=default
status.phase=Pending
```
- 예시
```shell
kubectl get pods --field-selector status.phase=Running
```
# Service Topology
- service topology 활성화시 Service는 Node Topology를 기반으로 Traffic Routing
- EX) 서비스는 트래픽을 클라이언트와 동일한 노드이거나 동일한 가용성 영역에 있는 엔드포인트로 우선적으로 라우팅되도록 지정할 수 있다.

## Intro
- public cloud의 운영자들이 service traffic을 동일 영역에서 유지하는 것을 선호
- Service Topology를 통해 Service Creator가 발신, 수신 Node에 대해 `Node Label에 기반한 Traffic Routing Policy 정의`할 수 있도록 함
    - ClusterIP에서 external Traffic을 `수신한 Node에서 실행중인 Pod로 Routing할 수 있게 함.`
- source, destination의 label matching을 통해 요구 사항에 적합한(closer or farther) Node Group 지정 가능


## Usage
- Service spec에서 `topologyKeys field를 지정`하여 `Service Traffic routing을 제어`할 수 있음(Service Topology가 활성화된 경우)
    - Service에 접근할 때 End-point를 정렬하는 데에 사용되는 Node label의 우선순위 목록
    - 첫 번째 `label 값이 해당 label의 발신 node 값과 일치하는 node로 보내짐` -> 없을 경우 다음, 다음 반복
    - 일치하는 것을 끝까지 못찾으면 traffic 거부
```yaml
topologyKeys:
    - "kubernetes.io/hostname"
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
```


## Constraints
- externalTrafficPolicy=Local 와 호환되지 않음(함께 사용 불가)
- 현재 사용 가능한: keys kubernetes.io/hostname, topology.kubernetes.io/zone, topology.kubernetes.io/region(다른 노드 레이블로 일반화 될 것)
- 최대 16개
- catch-all("*")은 마지막으로 사용해야 한다.

## examples
```yaml
# Node Local End-point로만
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname" # match되지 않을 경우 traffic drop

--
# Node Local End-point 선호
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname" # match되지 않을 경우 아무 곳이나 
    - "*"

--
# Only Zonal or Regional Endpoints 
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "topology.kubernetes.io/zone" #match zone이 없을 경우 region, 없을 경우 traffic drop
    - "topology.kubernetes.io/region"
    

```
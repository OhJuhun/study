# Service
- Pod 집합에서 실행중인 Application을 Network Service로 노출하는 추상화 방법
- k8s는 Pod에게 고유한 IP Address와 Pod 집합에 대한 단일 DNS 명을 부여, 그것들 간 LB를 수행할 수 있다.
- BE Pod가 해당 클러스터의 FE Pod에 기능을 제공하는 경우, 어떻게 연결할 IP Address를 찾을 수 있을지

## Service Resources
- k8s에서 service는 Pod의 논리적 집합과 그것들에 접근할 수 있는 정책을 정의하는 추상적 개념(Micro-service)
- Service가 대상으로 하는 Pod set은 selector가 결정
- pod의 replicaset이 변경되어도 FE에서는 그걸 인식할 필요가 없고, 추적할 이유도 없음
- 서비스 추상화는 이러한 `디커플링`을 가능하게 한다.

### Cloud-Native Service discovery
- App에서 Service Discovery를 위해 k8s API를 사용할 수 있는 경우, pod가 변경될 때마다 update되는 end-point를 api server에 query할 수 있다.

## Service 정의하기
- Service는 Pod와 비슷한 REST Object이다.
- Service definition을 API Server에 POST하여 new instance를 생성할 수 있다.
- Service Object의 이름은 유효한 DNS Sub-domain name이어야 한다.
    - 253자를 넘지 말아야 한다.
    - 소문자와 영숫자 - 또는 . 만 포함한다.
    - 영숫자로 시작한다.
    - 영숫자로 끝난다.
### 예제
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp # app=MyApp의 Label을 가진 Pod Set와 연결
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376 #TCP 9376 포트에서 수신
```

## Virtual IP & Service Proxy
- k8s의 `모든 Node는 kube-proxy를 실행`한다.
- kube-proxy는 ExternalName 이외의 유형의 Service에 대한 virtual IP 형식을 구현한다.

## Service Discovery
- 환경 변수, DNS 두 가지의 기본 모드 지원

### 환경 변수
- Pod가 Node에서 실행될 때, kubelet은 `각 활성화된 Service에 대해 환경 변수 Set을 추가`한다.
- {SVCNAME}_SERVICE_HOST 및 {SVCNAME}_SERVICE_PORT 변수 지원
- 서비스 이름은 대문자, -는 _로 변환된다.
- Service에 접근이 필요한 Pod가 있고, `환경 변수를 통해 Port, ClusterIP를 Client Pod에 부여`하는 경우, `Client Pod 생성 전에 Service를 만들어야 한다.` -> 그렇지 않으면 해당 Client Pod는 환경 변수를 생성할 수 없다.

### DNS
- Add-on을 사용하여 k8s cluster의 dns service를 설정할 수 있다.(필수)
- my-ns에 my-service가 있는 경우
  - control plane과 dns 서비스가 함께 작동하여 my-service.my-ns에 대한 dns record를 만든다.
  - my-ns namespace의 Pod들은 간단히 my-service에 대한 이름 조회를 수행하여 찾을 수 잇어야 한다.
- k8s dns server는 ExternalName service에 접근할 수 있는 유일한 방법이다.

## Headless Service
- lb와 single service ip가 필요하지 않은 경우 headless service를 만들 수 있음
  - spec.clusterIP에 "None" 명시
- Cluster IP가 할당되지 않고 kube-proxy가 이러한 Service를 처리하지 않으며, platform에 의해 lb 또는 proxy를 하지 않는다.

## Service Publishing
- service를 외부 IP주소에 노출하고 싶은 경우

### Service Types
#### ClusterIP(default)
- Service를 Cluster-internal IP에 Service를 expose
- `Cluster 내에서만 Service에 도달`할 수 있다.
- Ingress를 통해서도 Service를 노출시킬 수 있다.(Cluster의 진입점 역할을 하며 Service가 아니다.)

#### NodePort
- 고정 Port로 각 Node의 IP에 Service를 expose
- NodePort Service가 Routing되는 ClusterIP Service가 자동으로 생성된다.
- NodeIP:NodePort에 요청을 통해 `외부에서 서비스에 접속할 수 있다.`
- --service-node-port-range Flag로 지정된 범위에서 port 할당(default:30000-32767)
- 각 NodePort는 해당 Port를 Service로 Proxy
- .spec.ports[*].nodePort에 나타냄
- --nodeport-addresses Flag로 특정 IP block으로 프록시하기 위한 특정 IP 지정 가능(10.0.0.0/8, 192.0.2.0/25 등) -> kube-proxy가 local node로 고려해야 하는 IP 주소 범위
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
      # 기본적으로 그리고 편의상 `targetPort` 는 `port` 필드와 동일한 값으로 설정된다.
    - port: 80
      targetPort: 80
      # 선택적 필드
      # 기본적으로 그리고 편의상 쿠버네티스 컨트롤 플레인은 포트 범위에서 할당한다(기본값: 30000-32767)
      nodePort: 30007
```

#### LoadBalancer
- Cluod provider의 lb를 사용하여 service를 외부에 노출
- NodePort와 ClusterIP Service가 자동으로 생성된다.
- 일부는 loadBalancerIP를 지정할 수 있지만, 이를 지원하지 않는 provider에서 지정시 무시된다.
- protocol을 MixedProtocolLBService가 활성화된 경우 둘 이상의 Port가 정의되어 있을 때 다른 Protocol을 사용할 수 있다.(cloud provider가 지원하는 protocol이어야 함)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp # 외부 lb의 traffic이 이 pod로 전달된다.
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```
#### ExternalName
- 값과 함께 CNAME Record를 Return
- Service를 externalName field의 콘텐츠에 매핑한다.
- 어떤 종류의 Proxy도 설정되어 있지 않다.

## 단점
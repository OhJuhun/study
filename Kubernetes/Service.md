# Service
- Pod 집합에서 실행중인 Application을 Network Service로 노출하는 추상화 방법
- k8s는 Pod에게 고유한 IP Address와 Pod 집합에 대한 단일 DNS 명을 부여, 그것들 간 LB를 수행할 수 있다.

## Motivation
- 각 Pod는 고유 IP 주소를 갖지만, `Deployment에서는 한 시점에서 실행되는 Pod Set이 잠시 후 실행되는 것과 다를 수 있다.`
- 이에 따라 `BE Pod가 해당 클러스터의 FE Pod에 기능을 제공`하는 경우, `어떻게 연결할 IP Address를 찾을 수` 있을지

## Service Resources
- k8s에서 service는 Pod의 논리적 집합과 그것들에 접근할 수 있는 정책을 정의하는 추상적 개념(=Micro-service)
- Service가 대상으로 하는 Pod set은 selector가 결정
- BE pod의 replicaset이 변경되어도 FE에서는 그걸 인식할 필요가 없고, 추적할 이유도 없음
- 서비스 추상화는 이러한 `디커플링`을 가능하게 한다.

### Cloud-Native Service discovery
- App에서 Service Discovery를 위해 k8s API를 사용할 수 있는 경우, pod가 변경될 때마다 update되는 end-point를 api server에 query할 수 있다.

## Defining Service
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
  # selector controller는 selector와 일치하는 Pod 지속 검색,
  # my-service 라는 endpoint object에 대한 모든 Update Post
spec:
  selector:
    app: MyApp # app=MyApp의 Label을 가진 Pod Set와 연결
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376 # app=MyApp label을 가진 Pod의 9376 port, default: targetPort=port
```
- k8s는 이 service에 `service proxy가 사용하는 IP 주소(=Cluster IP` 할당
- Pod의 Port 정의에는 이름이 있고, `service의 targetPort에서 참조 가능`
- Service가 하나 이상의 Port를 노출해야 하기 때문에, 다중 포트 정의 지원(`다른 프로토콜로도 가능`)

## Selector가 없는 Service
- Service는 Pod가 아닌 `다른 종류의 Back-end도 추상화할 수 있다.`
  - 환경별로 외부 DB Cluster, 자체 DB 사용
  - 한 Service에서 `다른 Namespace` 또는 `다른 Cluster의 Service를 지정`하려 한다.
  - workload를 k8s로 migration하고 있다(?) -> k8s에서는 back-end의 일부만 실행한다.
- 위 경우 Pod Selector 없이 Service 정의가 가능
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
# selector가 없어, endpoint-object를 수동으로 추가해서 수동 매핑해야 한다.
--
apiVersion: v1
metadata:
  name: my-service # 유효한 DNS sub-domain name이어야 한다.
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
# Traffic은 192.0.2.42:9376으로 라우팅
```

## EndpointSlices
- Endpoint와 유사하지만 여러 resource에 endpoint를 분산시킬 수 있다.(full=100)

## Application protocol
- appProtocol field는 각 service port에 대한 Application protocol을 지정하는 방법 제공
- 해당 endpoint, slice object에 의해 Mirroring

## Virtual IP & Service Proxy
- `모든 Node는 kube-proxy를 실행`한다.
- kube-proxy는 ExternalName 이외의 유형의 `Service에 대한 virtual IP 형식을 구현`한다.

### User space proxy
- kube-proxy는 k8s control plane의 service / endpoint object 추가, 제거 감시
- 각 Service는 local node에서 port를 open
- 이 proxy port에 대한 연결은 back-end pod 중 하나로 proxy됨
- kube-proxy가 사용할 back-end pod를 결정할 때 Service의 SessionAffinity 설정 고려
- service의 clusterIP, port에 대한 트래픽을 캡처하는 Iptables 규칙 설치 -> `traffic을 back-end pod를 proxy하는 poxy port로 redirect`(round-robin)
- 선택된 첫 pod 미응답 시에 다른 back-end pod로 재시도
- Client -> clusterIP(iptables) -> kube-proxy (pod 선택 with round-robin)
- apiserver -> kube-proxy

### iptables proxy
- kube-proxy는 k8s control plane의 service / endpoint object 추가, 제거 감시
- 각 Service는 clusterIP, port에 대한 트래픽 캡처 및 traffic을 back-end set 중 하나로 redirect하는 iptables 규칙 설치
- `임의의 back-end 선택`
- `iptables 사용시 system overhead가 줄어듬`(userspace, kernel space 사이를 전환할 필요 없이 linux netfilter가 traffic을 처리하기 때문) -> 더 신뢰적
- 선택된 첫 pod 미응답 시에 연결 fail -> 정상적으로 테스트된 BE만 볼 수 있다 -> `traffic이 failed pod로 전송되는 것 막음`
- client -> clusterIP(iptables)(BE pod선택)
- apiserver -> kube-proxy -> clusterIP

### IPVS proxy
- kube-proxy를 실행하려면, node에서 IPVS를 사용 가능하게 해야 한다.
  - kube-proxy가 IPVS Proxy에서 시작될 때 IPVS kernel module을 사용할 수 있는지 확인
  - 감지되지 않으면 iptables proxy에서 실행된다.
- kube-proxy는 k8s service, end-point 감시
- netlink interface 호출 및 이에 따른 `IPVS 규칙 생성`, 이를 k8s `service, end-point와 주기적으로 동기화`
  - IPVS 상태가 `원하는 상태와 일치하도록 보장`
- service에 접근하면 `Traffic을 IPVS가 Back-end pod중 하나로 보낸다.`
- iptables와 유사하게 netfilter hook 기반, hash table을 기본 자료구조로 사용, kernel space에서 동작
  - iptables의 kube-proxy보다 latency가 짧은 traffic redirection
  - proxy 규칙 동기화시 성능 향상
  - 높은 network traffic throughput 지원
- client가 k8s, service, pod에 대해 알지 못하는 경우 
  - `Service의 IP:port로 향하는 traffic은 적절한 BE로 Proxy된다.`
- 특정 Client의 연결이 `매번 동일 Pod로 전달되도록 하려면`
  - service.spec.sessionAffinity=ClusterIP -> `Client의 IP Address 기반으로 Session affinity를 선택`할 수 있게 한다.
  - service.spec.sessionAffinityConfig.clientIP.timoutSeconds를 통해 세션 고정 시간 설정 가능

- Traffic을 BE Pod로 balancing하기 위한 추가 옵션
  - rr: round-robin
  - lc: least connected
  - dh: destination hashing
  - sh: source hashing
  - sed: shorted expected delay
  - nq: never queued
- Client -> clusterIP(Vritual Server) (BE pod(real server) 선택)
- apiserver -> kube-proxy -> clusterIP
## Service Discovery
- 환경 변수, DNS 두 가지의 기본 모드 지원

### 환경 변수
- Pod가 실행될 때, kubelet은 `각 활성화된 Service에 대해 환경 변수 Set을 추가`한다.
- {SVCNAME}_SERVICE_HOST 및 {SVCNAME}_SERVICE_PORT 변수 지원
- 서비스 이름은 대문자, -는 _로 변환된다.
- Service에 접근이 필요한 Pod가 있고, `환경 변수를 통해 Port, ClusterIP를 Client Pod에 부여`하는 경우, `Client Pod 생성 전에 Service를 만들어야 한다.` -> 그렇지 않으면 해당 Client Pod는 환경 변수를 생성할 수 없다.

### DNS
- Add-on을 사용하여 k8s cluster의 dns service를 설정할 수 있다.(필수)
- my-ns에 my-service가 있는 경우
  - control plane과 dns 서비스가 함께 작동하여 `my-service.my-ns에 대한 dns record를 만든다.
  - my-ns namespace의 Pod들은 간단히 my-service에 대한 이름 조회를 수행하여 찾을 수 잇어야 한다.
- k8s dns server는 ExternalName service에 접근할 수 있는 유일한 방법이다.

## Headless Service
- lb와 single service ip가 필요하지 않은 경우 headless service를 만들 수 있음
  - spec.clusterIP에 "None" 명시
- k8s 의 implementation에 종속되지 않고 다른 Service Discovery 메커니즘과 interface할 수 있다.
- Cluster IP가 할당되지 않고 kube-proxy가 이러한 Service를 처리하지 않으며, platform에 의해 lb 또는 proxy를 하지 않는다.

### Selector가 있는 Headless Service
- Endpoint Controller가 API에서 endpoint record 생성, DNS 구성 수정 및 `Service를 지원하는 파드를 직접 가리키는 주소를 반환`한다.

### Selector가 없는 Headless Service
- endpoint record를 생성하지 않는다.
- DNS system이 아래 두 가지 중 하나를 찾고 구성
  - ExternalName-유형 서비스에 대한 CNAME 레코드(=Canonical Name, Domain addr를 다른 Domain addr로 매핑)
  - 다른 모든 유형에 대해, 서비스의 이름을 공유하는 모든 엔드포인트에 대한 Record


## Service Publishing
- service를 외부 IP주소에 노출하고 싶은 경우 원하는 서비스 종류 지정 가능
- cf: https://blog.leocat.kr/notes/2019/08/22/translation-kubernetes-nodeport-vs-loadbalancer-vs-ingress
### Service Types
#### ClusterIP(default)
- Service를 Cluster-internal IP에 Service를 expose
- `Cluster 내에서만 Service에 도달`할 수 있다.
- Ingress를 통해서도 Service를 노출시킬 수 있다.(Cluster의 진입점 역할을 하며 Service가 아니다.)

#### NodePort
- Service에 외부 Traffic을 직접 보내는 가장 원시적 방법(cost sensitive할 때를 제외하곤 거의 사용하지 않음)
- Node의 특정 Port를 열어두고 `이 Port로 보내는 Traffic을 특정 Service로 forwarding`
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
- 노출하려는 service 마다 자체 IP Addr을 갖게 되어 LB 비용을 지불해야 한다.
- Cluod provider의 lb를 사용하여 service를 외부에 노출
- NodePort와 ClusterIP Service가 자동으로 생성된다.
- 일부는 loadBalancerIP를 지정할 수 있지만, 이를 지원하지 않는 provider에서 지정시 무시된다.
- protocol을 MixedProtocolLBService가 활성화된 경우 둘 이상의 Port가 정의되어 있을 때 다른 Protocol을 사용할 수 있다.(cloud provider가 지원하는 protocol이어야 함)
- TLS/SSL
```yaml자체 
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
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod # prod의 my-service 서비스를
spec:
  type: ExternalName
  externalName: my.database.example.com # my.database.example.com 에 매핑
```

#### ExternalIPs
- 하나 이상의 Cluster Node로 Routing되는 `External IP가 있는 경우 service가 여기로 노출될 수 있다.`
- k8s에 의해 관리되지 않아, `클러스터 관리자에게 책임이 있다.`
- 모든 ServiceTypes와 함께 지정할 수 있다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
  externalIPs:
    - 80.11.12.10 # 80.11.12.10:80으로 외부에서 접근이 가능하다.
```

## 단점
- VIP용 userspace proxy 사용시, `대규모 클러스터로 확장이 불가능`
- userspace proxy 사용시 Service에 `접근하는 Packet의 Source IP 주소가 가려진다.` -> NETWORK 방화벽이 불가능하게 한다.
  - iptables proxy mode의 경우에는 `Cluster 내부 source IP는 가리지 않지만`, `lb 또는 nodePort로 오는 Client에는 영향을 미침`

## virtual IP 구현시 고려해야 할 점
- 충돌 방지
- Service IP 주소
  - Service IP는 `단일 호스트에서 응답하지 않는다.` -> kube-proxy가 iptables를 필요에 따라 redirection되는 Virtual IP 주소를 정의하기 위해 사용
  - Client가 VIP에 연결하면 `Traffic이 자동으로 적절 end-point로 전송`
  - 환경 변수와 서비스 용 DNS는 실제로 서비스의 가상 IP 주소 (+ port)로 채워진다.
- Userspace
  - Backend service가 생성되면 k8s master는 virtual IP 주소를 할당한다.

## Supported Protocols
- TCP
- UDP: type=LoadBalancer 서비스의 경우, UDP 지원은 이 기능을 제공하는 클라우드 공급자에 따라 다르다.
- SCTP(스트림 제어 프로토콜): SCTP Traffic을 지원하는 Network Plugin을 사용하는 경우 대부분의 Service에 SCTP를 사용할 수 있다. LoadBalancer 서비스의 경우, UDP 지원은 이 기능을 제공하는 클라우드 공급자에 따라 다르다.(대부분 X)
- HTTP: Cloud provider가 지원하는 경우 lb mode의 service를 사용하여 service의 end-point로 전달하는 외부 http/https reverse proxy 설정 가능
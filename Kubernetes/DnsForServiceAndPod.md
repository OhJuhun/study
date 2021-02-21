# Intro
- k8s DNS는` Cluster의 service와 DNS Pod 관리`, Container들이 `DNS name을 해석할 때 DNS service의 IP를 사용하도록 kubelet을 구성`
## DNS Name이 할당되는 것들
- Cluster 내의 `모든 Service에는 DNS Name이 할당`
- 동일 namespace에서는 `service name을 조회하면, DNS Query를 통해 해당 service를 찾을 수 있다.`
- 다른 namespace에서는 `service.namespace를 조회하는 DNS Query를 통해 찾을 수 있다.`

# For Service
## A/AAAA Record
- Headless가 아닌 Service는 IP Family에 따라 my-svc.my-namespace.svc.cluster-domain.example 형식의 이름을 가진 DNS A or AAAA Record가 할당(service의 ClusterIP로 해석)
- Headless Service 또한 동일하지만, 이는 Service에 의해 선택된 Pods의 IP Set으로 해석된다.
    - Client는 해석된 IP Set에서 `IP를 직접 선택`하거나 `표준 Round-Robin을 통해 선택`할 수 있다.

## SRV Record
- Service에 속하는 Named Port를 위해 만들어짐
- _my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example의 형식을 가질 수 있음
    - Normal Service는 Port, Domain name으로 해석된다. my-svc.my-namespace.svc.cluster-domain.example
    - Headless Service는 Service를 지원하는 각 Pod에 대해 하나씩 복수 응답으로 해석되며 이 응답은 Pod의 Port, Domain Name 포함 auto-generated-name.my-svc.my-namespace.svc.cluster-domain.example

# For Pod
- 일반적으로 pod-ip-address.my-namespace.pod.cluster-domain.example와 같은 DNS 주소를 가짐
    - default namespace의 IP가 172.17.0.3인 Pod, Cluster Domain Name = cluster.local인 경우
        - 172-17-0-3.default.pod.cluster.local
    - Service에 의해 노출된 Deployment나 DaemonSet 의해 생성된 모든 Pod는 다음과 같은 DNS 주소를 갖는다.
        - pod-ip-address.deployment-name.my-namespace.svc.cluster-domain.example.

## Pod의 Host Name
- metadata.name
- Pod Spec에 hostName이 명시되어 있다면, metadata.name보다 우선순위
## Pod의 Sub-domain
- domain.sub-domain.my-namespace.svc.cluster-domain.example
- Pod와 `동일한 namespace 내에 같은 Sub-domain을 가진 Headless Service가 있다면, ` Cluster의 DNS 서버는 A 또는 AAAA Record 반환(전체 Host Name)

## setHostnameAsFQDN
- true 설정시 hostname 과 hostname --fqdn 모두 FQDN 반환
- 아닐 경우 hostname 명령어는 hostname만 반환한다.

## Pod DNS Policy
- Pod 별로 설정할 수 있다.
- dnsPolicy 필드에 지정
    - Default: Pod가 실행되고 있는 Node로부터 the name resolution configuration을 상속 받음.
    - ClusterFirst(명시적으로 지정되어있지 않다면 "ClusterFirst"가 기본값)
        - cluster domian suffix 구성과 일치하지 않는 DNS Query는 Node에서 상속된 Upstream Name Server로 전달
        - cluster 관리자는 추가 stub-domain과 Upstream DNS Server를 구축할 수 있다.
    - ClusterFirstWithHostNet: hostNetwork에서 running 상태인 Pod의 경우 DNS Policy인 "ClusterFirstWithHostNet"을 명시적으로 설정해야 한다.
    - None
        - Pod가 k8s 환경의 DNS 설정을 무시하게 한다.
        - 모든 DNS 설정은 Pod Spec 내에 dnsConfig필드를 사용하여 제공해야 한다.

## Pod DNS 설정
- dnsConfig 필드는 선택적
    - dnsPolicy 세팅과 함께 동작한다. 
    - 이때, 파드의 dnsPolicy의 값이 "None"으로 설정되어 있어야 dnsConfig 필드를 지정할 수 있다.
- dnsConfig
    - nameservers
        - Pod의 DNS 서버가 사용할 IP 주소들의 목록
        - Pod의 dnsPolicy가 "None" 으로 설정된 경우, `적어도 하나의 IP 주소가 포함`되어야 하며, 그렇지 않으면 이 속성은 생략
        - nameservers에 나열된 server는 `지정된 DNS 정책을 통해 생성된 기본 네임 서버와 합쳐지며 중복되는 주소는 제거`
    - searches
        - Pod의 Host name 찾기 위한 DNS 검색 Domain의 목록
        -  생략이 가능
        - 값을 지정한 경우 나열된 검색 domain은 지정된 DNS 정책을 통해 생성된 기본 검색 domain에 합쳐진다.
        - 병합 시 중복되는 도메인은 제거, k8s는 최대 6개의 검색 domain을 허용
    - options
        - name 속성(필수)과 value 속성(선택)을 가질 수 있는 오브젝트들의 선택적 목록
        - 이 속성의 내용은 지정된 DNS 정책에서 생성된 옵션으로 병합
        - 이 속성의 내용은 지정된 DNS 정책을 통해 생성된 옵션으로 합쳐지며, 병합 시 중복되는 항목은 제거
- 예시
```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
# 생성시 Container test의 /etc/resolv.conf에 아래 내용이 추가된다.
# nameserver 1.2.3.4
# search ns1.svc.cluster-domain.example my.dns.search.suffix
# options ndots:2 edns0
```

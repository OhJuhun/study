# Intro
- 쿠버네티스 클러스터 내의 Network End-point를 추적하는 방법을 제공
- End-point를 더 확장하고, 확장 가능한 대안을 제안


# Motivation
- Service, Cluster가 많은 수의 Pod로 많은 Traffic을 처리하게 성장함에 따라 End-Point API의 한계가 눈에 띔
- 많은 수의 네트워크 엔드포인트로 확장하는 것에 어려움이 있음
- service를 위한 모든 network endpoint가 단일 endpoint resource에 저장되어서, endpoint resource가 매우 커질 수 있음
- 위 사항에 따라 주로 Control Plane의 성능에 영향을 미쳤고, endpoint가 변경될 때마다 Traffic이 컸다.
- Endpoint Slice는 `위 문제들을 완화`하고, `Topology Routing` 같은 추가 기능을 위한 확장 가능한 플랫폼을 제공

# EndpointSlice Resource Definition
- Endpoint set을 참조한다.
- Service에 selector가 지정되면, Control Plane이 자동으로 EndpointSlice를 생성
- Service Selector와 매치되는 모든 Pod를 포함하며 참조
- `Protocol, Port, Service Name`으로 Network Endpoint를 그룹화
```yaml
apiVersion: discovery.k8s.io/v1beta1
kind: EndpointSlice
metadata:
  name: example-abc
  labels:
    kubernetes.io/service-name: example
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    topology: # deprecated
      kubernetes.io/hostname: node-1
      topology.kubernetes.io/zone: us-west2-a
```

# Address Type
- IPv4
- IPv6
- FQDN

# Conditions
- Consumer에게 유용한 Endpoint에 대한 condition 저장

## Ready
- Pod의 Ready와 매핑되는 Condition
- Ready = true 인 running Pod는 이것도 True가 되어야 함.
- Pod가 종료될 때 호환성의 이유로 `ready는 절대 true이면 안된다.`
- 예외) spec.publishNotReadyAddresses = true인 Service -> 항상 ready = true

## Serving
- `terminating 상태를 고려하지 않는 것` 외에 ready와 동일
- Endpointslice API Consumer는 Pod의 terminating 동안 Pod의 readiness에 관심이 있다면 이 조건을 확인해야 한다.

## Terminating
- Endpoint가 종료되는지의 여부

# Management

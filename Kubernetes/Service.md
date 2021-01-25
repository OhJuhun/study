# Service
- Pod 집합에서 실행중인 Application을 Network Service로 노출하는 추상화 방법
- k8s는 Pod에게 고유한 IP Address와 Pod 집합에 대한 단일 DNS 명을 부여, 그것들 간 LB를 수행할 수 있다.

## Service Resources
- k8s에서 service는 Pod의 논리적 집합과 그것들에 접근할 수 있는 정책을 정의하는 추상적 개념
- Service가 대상으로 하는 Pod 집합은 selector가 결정
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
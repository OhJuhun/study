# Intro
- Cluster 내 Service에 대한 외부 접근을 관리하는 API
- 일반적으로 HTTP 관리
- Load Balancing, SSL 종료, 명칭 기반의 가상 호스팅을 제공
- Cluster외부에서 클러스터 내부 서비스로 들어올 수 있는 HTTP, HTTPS Expose
- Ingress Controller가 LB를 사용하여 Ingress를 수행할 책임이 있다.(필수, ex-ingress-nginx)
- Traffic 처리를 돕기 위해 Edge Router 또는 추가 FE를 구성할 수도 있다.
![](./images/ingress.png)

# Resource Definition
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

# Resource Backend
- Ingress Object와 동일한 namespace 상에 있는 `다른 k8s resource에 대한 ObjectRef`
- 일반적인 용도는 static asset이 있는 Object Storage Backend로 Data를 수신하는 것
- Resource는 Service와 상호 배타적 -> 둘 다 지정시 Validation 실패
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-backend
spec:
  defaultBackend:
    resource:
      apiGroup: k8s.example.com
      kind: StorageBucket
      name: static-assets
  rules:
    - http:
        paths:
          - path: /icons
            pathType: ImplementationSpecific
            backend:
              resource:
                apiGroup: k8s.example.com
                kind: StorageBucket
                name: icon-assets
```
# pathType
- pathType을 포함하지 않는 경로는 validation fail
## ImplementationSpecific
- IngressClass에 따라 일치 여부가 달라짐
- 구현시 pathType으로 처리하거나, Prefix 또는 Exact pathType과 동일하게 처리할 수 있다.
## Exact
- Exact Matching
## Prefix
- URL path의 prefix를 / 기준으로 split한 값과 일치시킨다.

## 다중 매칭
- 이 경우 가장 긴 일치하는 경로 우선
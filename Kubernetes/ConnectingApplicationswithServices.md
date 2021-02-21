# 컨테이너 연결을 위한 쿠버네티스 모델 
## vs docker
- docker는 host-private networking을 사용하므로 `동일 machine에 있는 Container끼리만 통신이 가능` 
- docker container가 node를 통해 통신하려면, machine port에 `IP addr가 할당되어야 Container에 전달되거나 Proxy된다.`
    - container가 사용하는 port를 매우 신중하게 조정하거나 port를 동적으로 할당해야 한다.
    - 팀에서 port를 조정하고 관리하는 것은 매우 어려움
- k8s는 `Pod가 배치된 host와 무관하게 다른 Pod와 통신`할 수 있다.
    - 모든 `Pod에게 자체 cluster-private IP 제공` -> Pod간 `명시적으로 link를 만들거나 Container Port를 Host Port에 매핑할 필요가 없다.`
    - Pod 내 `모든 container는 모두 localhost에서 서로의 port에 도달`할 수 있음
    - cluster의 `모든 Pod는 NAT(네트워크 주소 변환)없이 서로 볼 수 있다.`

## Creating Service
- 이론적으로 Pod와 직접 통신할 수 있지만, 무슨 문제로 인해 `Pod가 죽을 경우, 기존 Pod를 삭제 후 다른 IP를 가진 새로운 Pod가 생성된다.`
    - service는 이 문제를 해결한다.
- Service는 Cluster 어딘가에서 실행되는 `논리적 Pod Set을 정의 및 추상화, 모두 동일한 기능을 제공하게 한다.`
- Service는 생성시 고유 IP Addr가 할당
    - 이는 수명과 연관
    - 활성화 되어 있는 동안 변경되지 않음
    - Pod는 Service와 통신되도록 구성 가능
    - Service와의 통신은 Service의 맴버 중 일부 Pod에 자동적으로 lb 된다.
- 생성된 Service는 ENDPOINTS를 갖고, 여기에 curl을 할 수 있다.
    - 이는 Virtual하므로 `외부에서는 절대 연결되지 않음`

### Replica 생성 이후에 Service를 생성하는 경우
- 아래 환경변수가 생성되지 않는다.
```
MY_NGINX_SERVICE_HOST=10.0.162.149
MY_NGINX_SERVICE_PORT=80
```
- 여러 Pod가 같은 Machine에 배치될 수도 있다.(HA가 보장이 안됨)

### Replica 생성 이전에 Service를 생성하는 경우
- 올바른 환경 변수(위 내용) 생성
- Pod의 Scheduler-level Service 분배가 가능

## DNS(내일 실습 해보기)
- k8s는 DNS cluster add-on service를 제공, `dns name을 다른 service에 자동 할당`
    ```bash
    # 이것이 클러스터에서 실행 중인지 확인하는 명령어
    $ kubectl get services kube-dns --namespace=kube-system

    NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
    kube-dns   ClusterIP   10.0.0.10    <none>        53/UDP,53/TCP   8m
    ```

## Service 보안
- Internet에 Service 배포 이전에 안전성 확인
    - https에 대한 자체 서명한 인증서 (신원 인증서를 가지고 있지 않은 경우)
    - 인증서를 사용하도록 구성된 Server
    - Pod에 접근할 수 있는 인증서를 만드는 Secret

### 인증서 및 Secret 생성

```bash
make keys KEY=/tmp/nginx.key CERT=/tmp/nginx.crt # key, cert 생성
kubectl create secret tls nginxsecret --key /tmp/nginx.key --cert /tmp/nginx.crt # secret 생성
```

### ConfigMap
```bash
echo "server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        listen 443 ssl;

        root /usr/share/nginx/html;
        index index.html;

        server_name localhost;
        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;

        location / {
                try_files $uri $uri/ =404;
        }
}" >> default.conf

kubectl create configmap nginxconfigmap --from-file=default.conf
```

### 80/443 노출
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort #외부 노출을 위해 NodePort 사용
  ports:
  - port: 8080
    targetPort: 80 # 80 / 443 모두 노출
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: nginxsecret
      - name: configmap-volume
        configMap:
          name: nginxconfigmap
      containers:
      - name: nginxhttps
        image: bprashanth/nginxhttps:1.0
        ports:
        - containerPort: 443
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/nginx/ssl # 여기에 mount된 volume을 통해 key에 접근 가능
          name: secret-volume
        - mountPath: /etc/nginx/conf.d
          name: configmap-volume
```

## Service Expose
# Volume
- Container 내 disk에 있는 자료는 일시적이므로 재생성시 모두 손상됨
- 이를 해결하기 위해 volume을 추상화
## Background
- Docker Volume은 Disk에 있는 Directory이거나 다른 container에서 존재한다.
- kubernetes는 다양한 volume을 지원하며 Pod는 여러 volume을 동시에 사용할 수 있다.
- Ephemeral volume은 Pod의 lifecycle을 따르지만, Persistent volume은 Pod의 수명과 관계없이 존재한다.(Container 재시작시에도 보존)
- .spec.volumes에서 사용할 볼륨을 지정, .spec.containers[*].volumeMounts에서 해당 볼륨 마운트 위치 선택

## 종류
### configMap
- Pod에 Configuration Data를 주입하는 방법 제공
- 사용하기 전에 반드시 생성해두어야 한다.
- 아래 예시는 안됨 아마 config-vol이라는 configMap을 정의해주어야 하는듯(?)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox
      volumeMounts: 
        - name: config-vol # configMap Mount
          mountPath: /etc/config # /etc/config/log_level에 저장됨
  volumes:
    - name: config-vol # ConfigMap 정의
      configMap: 
        name: log-config
        items:
          - key: log_level
            path: log_level
```

### emptyDir
- Pod가 Node에 할당될 때 처음 생성(비어있음)
- 해당 Node에서 Pod가 실행 중인 동안에만 존재(Pod가 제거되면 emptyDir의 데이터는 영구삭제)
- `Pod내 모든 container에서 동일한 파일 R/W가능`(각각 Container에서 다른 경로에 Mount될 수도 있음)
- Container Crash로부터는 안전하다.
- emptyDir.medium: "Memory" 설정시 tmpfs를 마운트할 수 있고, 이는 빠르지만 Node reboot시 삭제되고 작성하는 모든 file이 container memory limit에 포함된다.
- 용도
    - 디스크 기반의 병합 정렬과 같은 스크레치 공간
    - Crash로부터 Recover하기 위해 오래 걸리는 연산을 checkpointing
    - Web server container가 data를 처리하는 동안 Content-manager Container가 fetch하는 파일을 보관
    - container 간 데이터 교환
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### hostPath
- host node의 fs에 있는 file이나 directory를 Pod에 mount
- 용도
    - 도커 내부에 접근해야 하는 Container. /var/lib/docker 를 hostPath 로 이용함
    - Container에서 cAdvisor의 실행. /sys 를 hostPath 로 이용함
    - 주어진 hostPath를 Pod가 실행되기 이전에 있어야 하거나, 생성해야 하는지 그리고 존재해야 하는 대상을 지정할 수 있도록 허용함

### local
- disk, partition, directory같은 mount된 local storage device를 나타냄
- static하게 생성된 `Persistent Volume으로만 사용할 수 있다.` dynamic Provisioning된 것은 지원하지 않는다.
- hostPath volume에 비해 수동으로 Pod를 Node에 schedule 하지 않고도, 내구성, 휴대성을 갖춘 방식으로 사용됨
- system이 Persistent volume의 nodeAffinity를 확인하여 volume의 node constraint를 인식

### secret
- 암호 등 민감한 정보를 Pod에 전달하는 데에 사용된다.
- secret을 kubernetes API에 저장할 수 있고, k8s에 직접적으로 연결하지 않고도 Pod에서 사용할 수 있도록 file로 mount할 수 있다.
- tmpfs로 지원되므로 `비휘발성 storage에 절대 기록되지 않음`

## subPath
- volumeMounts.subPath를 사용하여 root 대신 참조하는 Volume내의 하위 경로를 지정할 수 있다.
- 단일 Pod에서 여러 용도의 한 Volume을 공유하는 것이 가능
```yml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data # site-data volume 공류
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html 
        name: site-data # site-data volume 공류
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

### subPath with extended environmental variables
- subPathExpr field를 사용해서 downward API 환경 변수로부터 subPath Directory명을 구성
- subPath와 subPathExpr는 mutually exclusive
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox
    command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs # container의 logs에 mount
      subPathExpr: $(POD_NAME) # workdir1(/var/log/pods) 내에 pod1 directory 생성. Downward API에서 Pod name 사용
  restartPolicy: Never
  volumes:
  - name: workdir1
    hostPath:
      path: /var/log/pods
```
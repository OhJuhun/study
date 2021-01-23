# Container
- 어디에서든 동일하게 동작
- 다양한 Cloud, OS에서 쉽게 배포됨
## Container Image
- Application을 실행하는 데 필요한 모든 것이 포함된 Software pacakge
## Container Run-time
- Container의 실행을 담당하는 SW

# Image
- Pod에서 Image 참조 전에 Application의 Container Image를 생성해서 Registry로 Push

## Image Update
- imagePullPolicy

| 이름 | 동작| 
|-----|-------------------------------------------------------------|
|IfNotPresent | kubelet이 이미 존재하는 이미지에 대한 Pull 생략 (기본 정책)|
|Always| Default, 항상 Image Pull |
|Never| Local Image 우선 적용|
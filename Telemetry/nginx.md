# WAS/Web Server 동작과정
1. Web Server가 Client로부터 HTTP 요청 수신
1. Web Server가 WAS에게 Request
1. WAS가 관련 Servlet을 메모리에 올림
1. WAS가 Thread 생성(Servlet에 대한)
1. HttpServeletRequest, HttpServletResponse 생성 및 로직 수행, Servlet에 전달
1. 적절하게 생성된 동적 페이지를 Response에 담아 WAS에 전달
1. WAS가 Reponse를 HttpResponse로 바꾸어 Web Server에 전달
1. Thread 종료 및 HttpServletRequest, HttpServletResponse 객체 제거

# Nginx Ingress Controller For kubernetes
- Ingress는 Kubernetes에서 Load Balancer의 역할
- Ingress Controller는 Cluster에서 실행되며, Ingress Resource에 따라 HTTP Load Balancer를 구성하는 Application
## Logging
- 두 가지의 로그를 갖는다
### log-foramt
- for HTTP & HTTPS traffics
### stream-log-format
- for UDP, TCP, TLS Passthrough traffics
```shell
kubectl logs ingress-nginx-pod-name -n ingress-nginx namespace
```
# XFF(X-Forward-For)
- HTTP Header 중 하나
- Web Server에 요청한 Client IP 식별을 위한 표준(`중간에 LB, Proxy가 껴있으면 해당 주소`)

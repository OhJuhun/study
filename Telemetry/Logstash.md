# Logstash
- 실시간 파이프라인 기능을 가진 오픈소스 데이터 수집 엔진
- input -> filter -> output 구조
- 얘를 ingress,service까지 만들 필요가 있을까?
## conf
```conf
input{
    file {
        path => ["/home/logstash/testdata.log"]
        sincedb_path => "/dev/null"
        start_position => "beginning"
    }
}
output{
    stdout {
        codec => rubydebug
    }
}
```
## 명령어
```shell
logstash -f .conf
```

## output Plugin
### Elasticsearch
```conf
elasticsearch{
    hosts => "elasticsearch.host:9200"
    index => "default-index"
}
```
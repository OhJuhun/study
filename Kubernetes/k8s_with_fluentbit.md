# Fluent-bit의 장점
- hight performance
- 소규모 자원 사용
- simple configuration format

# 문제의 발생
```json
{"reason":"rejected execution of processing of [1467813202][indices:data/write/bulk[s][p]]: request: BulkShardRequest [[index-2020.11.11][0]] containing [2434] requests, target allocation id: 3wVc6ZYQSPOGobyFfMFL9g, primary term: 1 on EsThreadPoolExecutor[name = name-logs/write, queue capacity = 200, org.elasticsearch.common.util.concurrent.EsThreadPoolExecutor@3cc64ef2[Running, pool size = 2, active threads = 2, queued tasks = 206, completed tasks = 596177059]]"}}}
```
- Elasticsearch가 request를 handling 하기에 가용 thread 수가 부족하다.

# 임시 해결책
- thread queue capaity가 부족하다는 것을 발견하여, vm에 cpu추가 후 es 재시작
- 이를 통해 더 이상 위 에러가 발생하지는 않았지만, 해당 Node의 fluent-bit mertric을 보고 이슈가 발생했다는 것을 알 수 있었다.
- 실제로 대부분의 record가 miss되었다.
- 에러가 발생하지도 않는데 기능이 수행이 안되니 에러를 찾기가..?

# Trouble shooting
- minor version으로 rollback
- 최신 버전으로 upgrade
- configuration parameter 변경
- `debug logging을 turn on`
    - initial sync 이후 해당 fluent-bit instance가 logging을 중지했다.
    - 반면, 다른 node들에서는 많은 양의 logging이 이루어졌다.

# Fluent-bit to Fluentd
- 문제는 있지만 해결책이 없어 일단 fluentd로 변경
- 문제가 있는 노드부터 해결하기 위해 `kubernetes node affinity`를 활용하여 rolluing deployment

# Fluentd Errors
```bash
[warn]: #0 emit transaction failed: error_class=Fluent::Plugin::Buffer::BufferOverflowError error="buffer space has too many data"
```
- Buffer Overflow error
- flush_thread_count 증가, flush_interval 최적화, buffer size 증가를 시도했지만 해결되지 않음
- 추가로 elasticsearch도 몇몇 request에 대해 fail이 또 발생(인덱싱 지연)
## 원인
- fluentd가 처음부터 로그를 긁어와서 오래된 log가 다시 push되고 있었기 때문에 발생
## 해결책
- 속도를 늦추기 위해 flush_thread_count를 1로 줄이고, flush_interval을 5초로 변경시킴
- Position DB feature enable로 과거 데이터는 보내지 않게 함.

# 아직 해결되지 않은 에러
- Prometheus metric이 fluent-bit의 어떠한 에러도 표출하지 않았다.(https://github.com/fluent/fluent-bit/issues/1935)

# 해결된 에러
- fluent-bit에서 위 에러 자체는 해결이 되었지만, 다른 이유에서 해당 문제는 계속 발생하고 있다.
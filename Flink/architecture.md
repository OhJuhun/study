# Flink
- unbounded, bounded data stream에 대한 상태 계산을 위한 분산 프로세싱 엔진이자 Framework
- streaming & batch processing platform 
- storm & spark과 같은 포지션

> Flink는 Unbounded / bounded data set 처리에 탁월하다.

## unbounded, bounded data 처리
- 모든 종류의 데이터가 event의 stream으로 생성된다.
- 데이터는 `unbounded / bounded stream`으로 처리될 수 있다.
- 시간, 상태를 정확히 제어하는 것으로 Flink runtime이 `unbounded stream 상의 어떠한 종류의 application에서도 사용할 수 있다.`
### Unbounded streams
- start만 있고 end는 없다.
- 지속적으로 처리되어야 한다.(event는 반드시 `즉시 처리되어야 한다.`)
- data가 순서대로 수집되어야 할 경우가 있다.

### Bounded streams
- start와 end가 정의되어 있다.
- 계산을 수행하기 전에 모든 데이터를 수집 및 처리할 수 있다.
- bounded data는 언제든 정렬될 수 있으므로 `순서대로 수집되지 않아도 된다.`
- 고정된 크기의 data set를 위해 설계된 알고리즘, 자료구조에 의해 `내부적으로 처리되어 높은 성능 산출`
![stream](./image/stream.png)
## 어디에든 Application Deploy
- Flink는 `분산 시스템`이며 `application을 실행하기 위해 compute 자원을 필요로` 한다.
- hadoop yarn, kubernetes 같은 common cluster resource manager와도 통합되고, stand-alone하게도 실행될 수 있다.
- 위 cluster들과 상호작용하여 잘 작동한다.
- Flink Application을 배포할 때 설정된 병렬 처리를 기반으로 `필요한 resource를 자동으로 식별`, `resource manager(k8s 등)에 요청`
- Fail 발생시에는 새로운 resource를 요청하여 container를 교체한다.
- 모든 submit, control이 `REST Call로 발생`하여, `Flink를 많은 환경에서 사용`할 수 있다.

## 규모와 상관 없는 Application 실행
- 어떤 scale의 stateful streaming application도 실행하게 설계됨(쉽게 매우 큰 application state를 유지보수 할 수 있음)
- Application은 `분산되어 동시에 실행될 수 있는 수 많은 Task로 병렬화 됨` -> 거의 `무한하게 CPU, memory, disk, network IO 사용 가능`
- 비동기, 증분 checkpoint 알고리즘으로 `processing latency를 최소화`
- `exactly-once state consistency 보장`

## In-memory 성능 활용
- Stateful Flink Application은 `local state acess에 최적화`되어있다.
- Task state는 `항상 in-memory로 유지`
- state size가 `memory를 초과`하면 `access-efficient한 on-disk 자료구조로 유지`
- 위 두 사항으로 인해 `processing latency가 낮다.`
- 주기적이면서 비동기적으로 `durable storage로 checkpointing`하여 fail시 `exactly-once state consistency를 보장`
![메모리 활용 구조](./image/in-memory.png)


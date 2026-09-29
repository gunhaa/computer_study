# Part3. Logs & Real-time Stream Processing

## 스트림 처리란?

> stream processing is "infrastructure for continuous data processing. I think the computational model can be as general as MapReduce or other distributed processing frameworks, but with the ability to produce low-latency results."

> "it is just processing which includes a notion of time in the underlying data being processed and does not require a static snapshot of the data so it can produce output at a user-controlled frequency instead of waiting for the 'end' of the data set to be reached."

| 조건 | 의미 | 배치와의 대비 |
|---|---|---|
| **데이터 자체에 시간 개념이 포함됨** | 레코드가 "언제 일어난 일인지"를 담음 | 배치는 데이터셋을 무시간적 집합으로 취급 |
| **정적 스냅샷을 요구하지 않음** | 입력이 계속 자라도 처리 가능 | 배치는 "입력이 고정됨"을 전제 |
| **출력 빈도를 사용자가 정함** | 원할 때 중간 결과를 낼 수 있음 | 배치는 "데이터의 끝"에 도달해야 출력 |

## 배치와 스트림은 다른 패러다임이 아니다

> "Data which is collected in batch is naturally processed in batch. When data is collected continuously, it is naturally processed continuously."

> "Production 'batch' processing jobs that run daily are often effectively mimicking a kind of continuous computation with a window size of one day."

```
  통념:  배치 처리 ≠ 스트림 처리   (다른 패러다임, 다른 도구, 다른 팀)

  저자:  배치 처리 = 윈도우 크기가 1일인 스트림 처리

         옛날: 데이터가 배치로 도착 (야간 파일 덤프)  → 배치로 처리하는 게 자연스러움
         지금: 데이터가 연속으로 도착 (이벤트 스트림)  → 연속으로 처리하는 게 자연스러움
```

- 배치 처리는 데이터 수집이 배치였던 시대의 잔재라는 주장
  - 데이터가 이미 연속으로 흐르고 있는데 인위적으로 하루씩 모아서 처리하는 건, 원래 없던 지연을 스스로 만들어 넣는 것이다
  - 즉, 배치란 window size가 1인 stream이라고 말할 수 있다

## Stateful Real-Time Processing(Kafka Streams)

- 무상태(stateless) 처리(필터, 맵)는 쉽다
- 어려운 건 상태를 가진 처리이다(조인, 추가정보 필요시)
- 여기서 나오는 Stateful Real-Time Processing(kafka streams)는 특정이벤트 발생시 handler(코드스니펫)를 실행해 다음이벤트를 만들어 발행하는 구조이다

### 문제

- "지난 1시간 동안 사용자별 클릭 수"를 계산하려면 사용자별 카운터를 어딘가에 들고 있어야 한다
- 저자의 답은 **로컬 상태 + changelog** 이다
  - 이 상태를 이용해 Otel collector와 유사한 동작으로 상태가 있는 스트림 처리를 진행한다
  - 사이드카 구조를 통해 log(CDC)를 가져오며, 이 이벤트 처리에 필요한 정보를 inmemory(RocksDB)로 가지고 있어 이를 매핑시켜 해결한다
  - e.g. 주문 발생 시, 주문 한 user가 vip라면 쿠폰 발행이라는 이벤트를 처리하기 위해서는 user의 정보를 모두 메모리에 쥐고 있으면 db의 select가 필요없어진다

### 로컬 상태

- 로컬 상태는 처리에 맞는 자료구조를 자유롭게 고를 수 있다
  - 스트림 처리기의 로컬 스토어는 log의 projection일 뿐이다

### changelog를 통한 장애 복구

- "이 시스템은 충돌 및 재시작 시 상태를 복원할 수 있도록 로컬 인덱스에 대한 변경 로그를 기록할 수 있다."
  - 이로 인해 로컬 처리기는 fault tolerance를 얻을 수 있다
- 실제 구현
  - Kafka Streams가 정확히 이 구조이다
  - 각 상태 저장소(state store)마다 `<app-id>-<store-name>-changelog` 라는 내부 토픽이 자동 생성되고, log compaction이 적용된다
  - 태스크가 다른 노드로 옮겨가면 그 노드가 changelog를 읽어 RocksDB를 재구축 한다
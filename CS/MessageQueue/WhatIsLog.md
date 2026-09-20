# The Log

> 원본: https://www.linkedin.com/blog/engineering/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying<br>
> 관련 영상: https://www.youtube.com/watch?v=v2Eab3U0W9s <br>
> 자세한 정리: https://github.com/gunhaa/QnA/tree/main/distributed-systems/the-log

- 원본 글은 Kafka를 만들고 운영한 경험(2010~)을 바탕으로, 2013년에 이벤트 스트리밍과 분산 시스템의 핵심 abstraction인 'Log'의 철학을 정리한 포스트이다
- Event Driven Architecture 및 현대 분산 환경의 인사이트를 담았다

## Part 1. What is Log?

### Log의 정의

> `A log is perhaps the simplest possible storage abstraction. It is an append-only, totally-ordered sequence of records ordered by time`
- append-only: 끝에만 추가. 중간 수정·삭제 없음 → 동시성 제어가 극단적으로 단순해짐(경쟁 지점이 tail 한 곳뿐)
- totally-ordered: 전순서. 임의의 두 레코드에 대해 어느 쪽이 앞인지 항상 결정됨 → 결정론적 재생(replay)이 가능해지는 근거
- ordered by time: 여기서 "time"은 실제 시간이 아닌 논리 시간(순서)를 의미한다

> A log is not all that different from a file or a table. A file is an array of bytes, a table is an array of records, and a log is really just a kind of table or file where the records are sorted by time
- 로그는 특별하지 않고, 바이트 배열인 파일이며, 레코드의 집합인 테이블과 같다

### Application Log vs Data Log

> The application log is a degenerative form of the log concept I am describing. The biggest difference is that text logs are meant to be primarily for humans to read and the 'journal' or 'data logs' I'm describing are built for programmatic access

| 구분 | 애플리케이션 로그 (syslog, log4j) | 데이터 로그 / 저널 (이 글의 주제) |
|---|---|---|
| 소비자 | 사람 | 프로그램 |
| 형식 | 비정형 텍스트 | 정형 레코드 (Protobuf 등) |
| 순서 보장 | 느슨함, 종종 유실 허용 | 전순서, 내구성 필수 |
| 목적 | 디버깅·감사 | **상태 재구성의 원본(system of record)** |
| 대표 도구 | Fluentd, Loki, ELK | Kafka, BookKeeper, WAL |

- 저자는 애플리케이션 로그를 "degenerative form"이라고 부른다
  - 같은 뿌리에서 나왔지만 "기계가 다시 읽고 상태를 재구성한다"는 핵심 기능을 잃어버린 형태

### Log의 역사적 기원

> The usage in databases has to do with keeping in sync the variety of data structures and indexes in the presence of crashes. To make this atomic and durable, a database uses a log to write out information about the records they will be modifying, before applying the changes to all the various data structures it maintains

- DB는 원자성(atomicity)과 내구성(durability)을 위해 실제 데이터 구조를 건드리기 전에 변경 의도를 먼저 로그에 쓰는 것이 먼저 사용되었다
  - Redo Log (WAL): 디스크의 실제 데이터 페이지(테이블)를 변경하기 전에 먼저 로그 기록/플러시 (목적: Crash Recovery, 위치: InnoDB 스토리지 엔진)
  - Binlog: 트랜잭션 커밋 완료 시점에 논리적 변경 사항 기록 (목적: 복제 및 시점 복구, 위치: MySQL Server)

> The log is the record of what happened, and each table or index is a projection of this history into some useful data structure or index

- 보통 우리는 "테이블이 진짜 데이터고, 로그는 크래시 복구를 위한 보조 장치"라고 생각하지만, 이를 정확히 반대로 생각한다
  - 즉, projection은 여러 개(table, index)여도 되고, 언제든 버리고 다시 만들 수 있으며, 새로운 종류를 나중에 추가할 수 있으며, log(history of what happen)만 온전하면 된다

### Logs in Distributed Systems

> If two identical, deterministic processes begin in the same state and get the same inputs in the same order, they will produce the same output and end in the same state

> You can reduce the problem of making multiple machines all do the same thing to the problem of implementing a distributed consistent log to feed these processes input

- 이 글에서 가장 중요한 State Machine Replication (SMR) 원리이다
- 즉, 분산 시스템에서 가장 중요한 즉 "N대의 머신을 일치시키는 문제"를 "일관된 분산 로그를 만드는 문제" 로 추상화 시킨 것이다
  - 분산 시스템의 모든 문제를 하나로 압축한 것이고, 로그만 잘 만들면 나머지는 공짜라는 주장이다
    - 좋은 로그를 만드는 핵심은 Code/Command를 쓰지 말고, Event(결과)를 쓰는 것이다
    - 그리고 머신에 따라 달라질 수 있는 log의 값(e.g. 시간)은 하나가 정하는 것이다(Leader)

### Change log 101

> There is a fascinating duality between a log of changes and a table.

- 로그 → 테이블: 로그를 순서대로 적용하면 최신 상태가 나옴
- 테이블 → 로그: 테이블에 발생하는 업데이트를 캡처하면 changelog가 나옴 (= CDC)

> "The magic of the log is that if it is a complete log of changes, it holds not only the contents of the final version of the table, but also allows recreating all other versions that might have existed."

- 로그와 테이블은 대칭이 아니며, 로그 ⊃ 테이블이다
  - 로그는 "현재 상태"뿐 아니라 "존재했던 모든 과거 상태"를 담는다
  - 테이블은 정보를 잃는 손실 압축(lossy)이다
- 이 비대칭이 이 글 전체의 주장(`로그를 system of record로 삼아라`)에 대한 근거가 된다
- 테이블을 원본으로 삼으면 되돌릴 수 없지만, 로그를 원본으로 삼으면 언제든 같은 형태의 테이블/인덱스를 다시 만들 수 있다
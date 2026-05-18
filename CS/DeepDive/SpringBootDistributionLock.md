# Spring boot distribution Lock

- 분산락은 결국 분산 트랜잭션이다
  - 분산락 내에 실행되야하는게 분산 트랜잭션이다
- SpringBoot에서 Redis를 이용한 Distribution Lock의 구현은 일반적으로 Lettuce/Redisson을 사용해 구현한다
- Lettuce는 단일 커넥션으로 Redis의 명령어를 직접 호출하는 방식에 가까워 멀티스레드 환경에서 성능이 좋다
- Redisson은 다수의 커넥션으로 락(분산)을 구현할때 많이 사용되고 자바 인터페이스를 사용해 자료구조를 사용할 수 있으며, 기능이 많은 만큼 라이브러리 자체가 무겁고 학습 곡선이 있다

## Redisson

### 왜 Redisson 락을 많이 사용할까?

- Lettuce의 경우 직접 락을 구현하기 위해서는 직접 구현이 필요하다
  - expire처리 등 많은 처리를 수동으로 해야한다
- Redisson은 Pub/Sub 방식을 사용해 락을 관리한다
  - Redis가 신호를 주는 Pub/Sub를 사용해 성능이 좋다
  - Redisson자체에서 구현된 기능을 사용하면 검증된 기능을 사용할 수 있다

### DB를 사용한 락도 가능한데, 굳이 관리 포인트를 늘리는 Redis를 사용해야 할까?

- Redis를 이용해 DB의 부하를 감소 시킬 수 있다
  - DB의 커넥션 풀이 마르는 것을 방지할 수 있다
  - 데드락 위험이 있다
- DB로 구현하면 결국 Spin Lock방식에 가깝기에 CPU와 네트워크 자원이 낭비된다
- 하지만 데이터 무결성이 최우선이고 트래픽이 낮다면 나쁜 방법은 아니다
  - 하지만 순간적인 트래픽이 몰린다면 단일 서버라도 Redis를 도입해 DB의 부담을 덜어주는 것이 안정성에서 좋은 판단이다

### 왜 Redis의 Pub/Sub가 효율적인 솔루션일까?

- Java Redisson의 분산락이 구현한 Pub/Sub은 락 대기 스레드들이 spin대기를 하지 않는다
- 락 대기 스레드들은 Redisson의 락 대기집합인 라이브러리 내부에서 구현된 Semaphor 내부의 AQS에서 kernel level thread blocking되어 CPU scheduling에서 제외된다
- Redis내부에 락이 잡힌 스레드들이 스케쥴링 되어있고, 신호를 보내 스레드를 깨운다(fair/unfair모드 존재)
  - 분산 환경에서 서버 내부의 Redisson이 Semaphore의 AQS 가장 앞에 대기 중인 스레드 하나만 통과시킨다
  - 분산 환경에서 선발된 '대표 스레드'들이 Redis에 락 획득 요청을 보낸다. Redis는 싱글 스레드로 동작하므로, 가장 먼저 도착해 원자적으로 스크립트를 실행한 단 하나의 스레드만 락을 차지한다
  - 락을 얻지 못한 나머지 대표 스레드들은 다시 본인의 서버 Semaphore의 AQS로 돌아가 잠들고, 다음 신호를 기다린다
- Redis 자체의 Custom protocol인 RESP(REdis Serialization Protocol)를 사용하며 이는 TCP/IP socket위에서 동작한다
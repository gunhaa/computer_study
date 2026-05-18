# Lettuce vs Redisson

- Lettuce
  - 구조적 특징
    - 하나의 커넥션을 이용해 어플리케이션의 모든 요청을 관리하는 스프링의 기본 라이브러리
    - Netty를 이용한 구조로, 하나의 커넥션을 사용해 클라이언트의 요청을 모아 batch로 req/res를 받는다
  - 분산 락 구현 시 한계
    - 자체적인 분산 락 API를 제공하지 않아 `SETNX` 명령어를 이용한 spin lock 형태로 직접 구현, 혹은 Redisson의 pub/sub를 사용한 복잡한 구현을 직접 해야함
      - 추가적인 네트워크 비용, `Thread.sleep()/yield()`등을 사용함으로써 컨텍스트 스위칭 비용의 증가
    - 락을 획득할 때까지 지속적으로 Redis에 재시도 요청을 보내므로, 트래픽이 몰릴 때 Redis와 application의 자원을 과도하게 소모함
- Redisson
  - 구조적 특징
    - 자바의 익숙한 인터페이스 형태로 분산 락 기능을 제공하는 라이브러리
    - 분산 락의 구현은 Redis의 pub/sub로 구현 되어있다
      - 락이 해제될 때 채널을 통해 통지를 받으므로, Lettuce와 달리 대기 중인 클라이언트가 Redis 서버에 지속적인 조회를 보내지 않아 부하가 현저히 적음
  - 연결 및 스레드 구조
    - 다중 커넥션 풀 사용: Pub/Sub 채널을 구독하고 유지하기 위해 Lettuce와 달리 여러 개의 커넥션을 맺고 관리
    - Netty EventLoop: 비동기 I/O 및 요청 처리를 위해 다수의 Netty Worker와 스레드를 활용함 (기본값: `Runtime.getRuntime().availableProcessors() * 2`)
      - netty event loop의 전용 스레드를 통해 network i/o 결과를 받아오며, 일반 스레드는 내부 라이브러리의 작업(watch dog 등)과 콜백 작업에 사용한다
        - webflux의 `Scheduler.boundedElastic()`처럼 event loop의 흐름을 깨는 cpu 작업을 위해 사용한다

---

# Why Redisson?

- Redis 서버 자원 보호
  - 대규모 트래픽 환경에서의 동시성 제어를 목표로 한다면 사용하는 편이 좋다
  - Lettuce의 스핀 락 방식은 경합이 심해질 때 application을 마비시킬 위험이 있어, Pub/Sub 기반으로 부하가 적은 Redisson을 채용
- 검증된 편의 기능 내장
  - 락 획득 타임아웃 처리, 비즈니스 로직 연장을 위한 Watch Dog 기능 등이 프레임워크 레벨에서 이미 안정적으로 구현
  - 검증에 필요한 시간낭비를 생략 할 수 있음
  - `바퀴를 다시 발명`하지 않고 비즈니스 로직에만 집중할 수 있음
- Redisson이 더 많은 커넥션 풀과 스레드를 관리해야 해서 가벼운 Lettuce에 비해 인프라 리소스를 더 쓰지만, 안정성을 위해 이 비용을 감수

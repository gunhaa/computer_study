# 트랜잭션

- 트랜잭션이란?
  - 작업의 완전성을 보장해주는 것이다
  - 논리적인 작업 셋을 모두 완벽하게 처리하거나, 처리하지 못할 경우 원 상태로 복구해 일부만 적용되는 현상의 발생을 막는 기능이다
  - Lock과 트랜잭션은 서로 비슷한 개념같지만 사실 잠금은 동시성 제어를 위한 기능이고, 트랜잭션은 데이터의 정합성을 보장하기 위한 기능이다
  - 하나의 회원 정보 레코드를 여러 커넥션에서 동시에 변경하려고하는데 잠금이 없다면 하나의 데이터를 여러 커넥션에서 동시에 변경할 수 있게 된다
    - 이 경우 레코드의 값은 예측할 수 없는 상태가 된다
- InnoDB테이블과 MyISAM테이블을 비교해보면 이 트랜잭션에 대해 이해하기가 쉽다
  - autocommit 모드에서 아래 쿼리를 실행시키면 엔진 트랜잭션에 대해 이해하기가 쉽다
  ```sql
  -- 3을 미리 넣어 놓은 상태
  mysql> INSERT INTO tab_myisam (fdpk) VALUES (1), (2), (3);
  mysql> INSERT INTO tab_innodb (fdpk) VALUES (1), (2), (3);
  ```
  - 해당 상황에서는 같은 에러가 발생한다
  ```sql
  ERROR 1062 (23000): Duplicate entry '3' for key 'PRIMARY'
  ```
  - 같은 에러가 발생하지만, MyISAM은 해당 상황에서 Partial Update를 실행해 1,2 가 DB에 들어가게 된다
    - 해당 현상은 데이터의 정합성을 맞추는데 상당히 어려운 문제를 만들어낸다
  - 이에 비해 InnoDB의 경우 DB에는 처음에 있던 3말고는 값이 들어가지 않는다
    - InnoDB의 경우 트랜잭션의 원칙대로 INSERT문장을 실행하기 전 상태로 그대로 복구한다
    - InnoDB는 Undo Log를 이용해 트랜잭션 실패 시 이전 상태로 복구한다.

# MySQL 엔진의 Lock

## 글로벌 락

- `FLUSH TABLES WITH READ LCOK`
  - MySQL에서 제공하는 잠금 가운데 가장 범위가 크다
  - 해당 명령은 실행과 동시에 MySQL 서버의 모든 데이터 테이블을 닫고 잠금을 건다
  - SELECT를 제외한 DDL 문장이나 DML 문장을 실행하는 경우 해당 문장이 대기 상태로 남는다

## 테이블 락

- `LOCK TABLES table_name [ READ | WRITE]` 명령으로 특정 테이블의 락을 획득할 수 있다
  - 이렇게 획득한 락은 `UNLOCK TABLES` 명령으로 잠금을 반납 할 수 있다
- 특별한 상황 아니면 어플리케이션에서 사용할 필요가 거의 없다
- 명시적으로 테이블을 잠그는 작업은 글로벌 락과 동일 하게 온라인 작업에 상당한 영향을 미치기 때문이다
- MyISAM, MEMORY 테이블에 데이터 변경 쿼리를 날릴 경우 묵시적 테이블 락이 걸린다
- InnoDB의 경우 스토리지 엔진 차원에서 레코드 기반의 잠금을 제공하기 때문에 단순 데이터 변경 쿼리로 인해 묵시적인 테이블 락이 설정되지는 않는다

## 네임드 락

- `GET_LOCK()` 함수를 이용해 임의의 문자열에 대해 잠금을 설정할 수 있다
- 이 잠금의 특징은 대상이 테이블이나 레코드 또는 AUTO_INCREMENT 같은 데이터베이스 객체가 아니라는 것이다
- 사용자가 지정한 문자열(String)에 대해 획득하고 반납하는 잠금이다
- 사용 예제: DB 하나에 5개의 웹 서버가 접근하는 상황에서, 5개의 웹 서버가 정보를 동기화하는 경우 네임드 락을 사용한다

```sql
-- mylock 문자열에 대해 잠금을 획득한다
-- 이미 잠금을 사용중이면 2초 동안만 대기한다(2초 이후 자동 잠금 해제)
mysql> SELECT GET_LOCK('mylcok', 2);

-- mylock 문자열에 대해 잠금 설정 확인
mysql> SELECT IS_FREE_LOCK('mylock');

-- mylock 문자열에 대해 획득했던 잠금을 반납
mysql> SELECT RELEASE_LOCK('mylock');

-- 3개 함수 모두 정상적으로 락을 획득하거나 해제한 경우 1, 아니면 NULL이나 0을 반환
```

- 많은 레코드에 복잡한 요건으로 레코드를 변경하는 트랜잭션에 유용하게 사용된다
- 동일 데이터 변경, 참조하는 쿼리끼리 분류해 네임드 락을 걸고 쿼리를 실행하면 아주 간단히 해결 할 수 있다

## 메타데이터 락

- 데이터 베이스 객체(테이블, 뷰)의 이름이나 구조를 변경하는 경우 획득하는 잠금
- 명시적으로 획득하거나 해제하는 것이 아닌, 테이블의 이름을 변경하는 경우 자동으로 획득하는 잠금이다
  - `RENAME TABLE tab_a TO tab_b` 시 사용됨

## 레코드 락(Record Lock)

- 레코드 단위로 걸리는 잠금이다.
- 주로 InnoDB 같은 트랜잭션 지원 스토리지 엔진에서 사용한다.
- 특정 행(row) 하나에만 잠금을 걸기 때문에, 동시에 여러 트랜잭션이 다른 행에 접근하는 것이 가능해져서 동시성(concurrency)을 높인다.
- 레코드 락은 크게
  - 공유 락(Shared Lock, S-lock): 읽기용 잠금, 다른 트랜잭션도 읽을 수 있지만 수정은 불가
  - 배타 락(Exclusive Lock, X-lock): 쓰기용 잠금, 다른 트랜잭션이 읽거나 쓰는 것도 막음
- 락 경합 시 대기하거나 데드락이 발생할 수 있어서 적절한 설계가 중요하다.

## InnoDB 스토리지 엔진 잠금

- MySQL에서 제공하는 잠금과는 별개로 스토리지 엔진 내부에서 레코드기반의 잠금 방식을 탑재하고 있다
- InnoDB는 여러 DB에서 사용되는, C,C++로 제공되는 오픈소스 코드셋이다
  - MySQL: 기본 InnoDB 사용
  - MariaDB : 포크된 InnoDB 변형
- 레코드 기반 잠금 방식을 이용해 뛰어난 동시성 처리를 할 수 있다

### InnoDB의 잠금은 "레코드"가 아닌 "인덱스 레코드"에 걸린다

- InnoDB에서는 데이터에 변화가 일어나는 모든 행위는 반드시 레코드(인덱스)에 락을 건 후 실행된다.
  - 정확히 말하면, 레코드 수준의 락(Row-level Lock)을 걸고 나서 해당 작업을 수행한다.
- InnoDB 스토리지 엔진은 레코드 자체가 아니라 인덱스의 레코드를 잠근다
  - 레코드 자체를 잠그느냐, 아니면 인덱스를 잠그느냐는 것은 상당히 크고 중요한 차이를 만들어 낸다
- InnoDB는 레코드 자체에 락을 거는 것이 아니라, 해당 레코드를 포함하는 인덱스의 엔트리에 락을 건다
  ```sql
  UPDATE employees SET salary = 6000 
  WHERE last_name = '황' AND first_name = '건하';
  ```
  - last_name 컬럼에만 세컨더리 인덱스가 걸려있을 경우, 해당 쿼리가 실행되면 세컨더리 인덱스를 통해 '황'으로 접근 후 '건하'를 찾는다
  - 이 트랜잭션의 실행 중일떄는 last_name이 '황'인 모든 인덱스 엔트리에 잠금이 걸린다
  - 세컨더리 인덱스의 엔트리(인덱스의 레코드)는 [인덱스 컬럼 정보 + PK 값]으로 구성되어 있다
  - InnoDB에서 '인덱스 레코드 락'이란, 조건에 처음에 걸린 세컨더리 인덱스 엔트리 자체를 먼저 잠그고, 이어서 그 안의 PK 값을 통해 클러스터 인덱스로 찾아가 최종 데이터 레코드까지 연속해서 잠그는 것을 의미한다
- 이는 InnoDB가 모든 데이터를 인덱스를 통해 접근하기 때문이다
  - 인덱스 → 해당 row 접근 → 정확한 인덱스 레코드에만 락
  - 인덱스가 없을 경우 → 테이블 풀 스캔 → 모든 row 잠금 가능성
- InnoDB에서 락의 단위는 레코드 자체가 아니라 인덱스의 엔트리
- 인덱스가 없으면 락 성능이 급격히 저하될 수 있음
- 따라서 정확한 락 범위를 위해 인덱스 설계가 매우 중요

### REPEATABLE READ격리 수준에서 발생하는 Next Key Lock

- Next key lock은 record lock과 gap lock이 합쳐진 형태이다
- InnoDB의 REPEATABLE READ 격리 수준은 phantom read를 발생시키지 않기 위해 특정 조건에서 record lock만이 아닌 gap lock을 추가적으로 발생시킨다
```java
    // Next Key Lock 예제 테스트
    // innodb_lock_wait_timeout = 3
    @Test
    @DisplayName("Next Key Lock: 비관적 배타락으로 범위 조회 시 해당 범위에 INSERT 자체가 차단되어 팬텀 리드를 원천 봉쇄한다(Point Search는 최적화에 따라 Gap락이 걸리지 않을 수 있다)")
    void nextKeyLockTest() throws InterruptedException {
        // given
        final ExecutorService es = Executors.newFixedThreadPool(2);
        final CountDownLatch startLatch = new CountDownLatch(1);
        final CountDownLatch endLatch = new CountDownLatch(1);
        final CountDownLatch insertLatch = new CountDownLatch(1);
        final CountDownLatch errorLatch = new CountDownLatch(1);

        final long startId = accountDbService.join();
        final int joinIterationInit = 0;
        final int joinIterationEnd = 5;
        final long greaterThanId = (startId + joinIterationEnd) / 2;

        final LongAdder exceptionCounter = new LongAdder();
        for (int i = joinIterationInit; i < joinIterationEnd; i++) {
            accountDbService.join();
        }

        // when
        es.submit(() -> {
            transactionTemplate.executeWithoutResult(status -> {
                try {
                    startLatch.await(DEFAULT_LATCH_TIMEOUT, TimeUnit.SECONDS);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    exceptionCounter.increment();
                }
                // Next Key Lock Acquire
                // for update는 insert/update시와 같은 비관적 배타락을 획득한다
                // 범위에 거는 락은 사이와 이후의 무한대의 공간에 갭 락(비관적 배타락)을 건다(팬텀 리드 방지)
                // 갭락은 Range Scan시 해당되는 곳에 걸리며, 마지막 레코드 이후가 조건에 있다면 INSERT가 차단된다
                // 이로 인해 해당 범위 내에 새로운 레코드의 Insert가 원천 차단된다
                accountDbRepository.findAllByIdGreaterThanWithLock(greaterThanId);
                insertLatch.countDown();
                try {
                    errorLatch.await(DEFAULT_LATCH_TIMEOUT, TimeUnit.SECONDS);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    exceptionCounter.increment();
                }
                endLatch.countDown();
            });
        });

        es.submit(() -> {
            transactionTemplate.executeWithoutResult(status -> {
                try {
                    insertLatch.await(DEFAULT_LATCH_TIMEOUT, TimeUnit.SECONDS);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    exceptionCounter.increment();
                }
                try {
                    // 위 스레드가 greaterThanId 이후 구간에 Next-Key Lock을 선점하고 있다
                    // 신규 INSERT는 해당 구간의 Gap Lock막혀 블로킹 된다
                    accountDbService.join();
                } catch (CannotAcquireLockException e) {
                    Thread.currentThread().interrupt();
                    exceptionCounter.increment();
                    errorLatch.countDown();
                }
            });
        });

        // then
        startLatch.countDown();
        boolean isEnd = endLatch.await(DEFAULT_LATCH_TIMEOUT, TimeUnit.SECONDS);

        assertThat(isEnd).isTrue();
        assertThat(exceptionCounter.sum()).isEqualTo(1);

        es.shutdownNow();
        es.close();
    }
```
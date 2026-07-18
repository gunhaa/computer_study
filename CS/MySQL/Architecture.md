# MySQL 아키텍쳐

## MySQL 전체 구조

![images1](images/mysql_architecture_1.png)

![images2](images/mysql_architecture_2.png)

![images3](images/mysql_architecture_3.png)

- MySQL은 사용 RDBMS와 같이 대부분의 프로그래밍 언어로부터 접근 방법을 모두 지원한다
- MySQL 서버는 크게 MySQL 엔진과 스토리지 엔진으로 구분할 수 있다
  - 이 둘을 합쳐 MySQL 또는 MySQL이라고 표현한다

### MySQL 엔진

- 클라이언트로부터 접속 및 쿼리 요청을 처리하는 커넥션 핸들러와, SQL parser & 전처리기, 쿼리의 최적화된 실행을 위한 옵티마이저가 중심을 이룬다
- 또한 MySQL은 표준 SQL(ANSI SQL)을 지원하기 때문에 표준 문법에 따라 작성된 쿼리는 타 DBMS와 호환되어 실행될 수 있다
- 전처리된 쿼리문은 SQL문과는 다소 차이가 있다. 쿼리 전처리는 SQL 문장이 실제로 실행되기 전에 수행되는 일련의 처리 과정으로, 쿼리의 최적화 및 구조 분석을 포함한다
  - java의 .class 파일과 유사하게 전처리된다고 생각하면 된다

### 스토리지 엔진

- 요청된 SQL 문장을 분석하거나 최적화하는 등 DBS의 두뇌에 해당하는 처리를 한다.
- 실제 데이터를 디스크 스토리지에 저장하거나 디스크 스토리지로부터 데이터를 읽어오는 부분은 스토리지 엔진이 전담한다
- MySQL서버에서 MySQL 엔진은 하나지만 스토리지엔진은 여러개를 동시에 사용할 수 있다
- 테이블이 사용할 스토리지 엔진을 지정하면 이후 해당 테이블의 모든 읽기, 변경 작업은 정의된 스토리지 엔진이 처리한다

```sql
CREATE TABLE test_table (fd1 INT, fd2 INT) ENGINE=INNODB;
```

- 스토리지 엔진은 mysql엔진에서 파싱되어 전처리된 쿼리문을 실행시키는 하드웨어적인 일만 수행한다

#### MVCC(Multi Version Concurrency Control)

- mysql의 특징적인 아키텍쳐이며, 언두로그를 이용해 구현된다
- lock이 없는 읽기를 가능하기 위해 만든 구조이며, 메모리를 희생해 속도를 빠르게 만든 구조이다
- 트랜잭션 격리 수준 별로 MVCC를 사용하는 방법이 다르며, READ_UNCOMMITED/SERIALIZABLE은 MVCC를 이용한 read_view를 사용하지 않는다
  - REPEATABLE_READ는 read_view를 트랜잭션 생성시 조회하며, 트랜잭션 진행시 read_view의 타겟 레코드의 cluster_index의 `db_trx_id` 사용해 read_view의 `m_creator_trx_id`(트랜잭션 생성 시점)보다 조회하는 레코드의 `db_trx_id` 높다면 `db_roll_ptr_id`을 이용해 트랜잭션 생성 시점 보다 낮은 정보를 찾아와 이를 이용한다
  - READ_COMMITED는 read_view를 select 시점마다 새로 생성하며, `db_trx_id`를 비교하며 다른 트랜잭션이 커밋하지 않은 상태의 레코드를 만나면, 해당 레코드의 `db_trx_id`를 보고 아직 커밋되지 않았음을 확인한 뒤 `db_roll_ptr을` 통해 언두 로그에서 가장 최근에 커밋된 버전의 데이터를 찾아와 읽는다
- `db_trx_id`의 디테일
  - db 전체의 채번이며, 각 `db_trx_id`는 유니크 하다
  - long 자료형(64bit)으로 구현되기에, 전부 소모될 일은 사실상 없다
  - Read-Write 트랜잭션에만 할당된다(INSERT, UPDATE, DELETE)
- `db_roll_ptr_id`의 디테일
  - 이전 트랜잭션의 정보를 사용해야할때 사용되는 포인터이며, 언두로그에서 해당 자료의 포인터가 저장되어 있다
  - 롤백 가능한 자료들은 링크드 리스트의 형태로 있어 더 이전의 결과가 필요할 경우 이전 포인터로 계속해서 접근한다
- read_view의 디테일
  - `m_creator_trx_id`, `m_ids`, `m_up_limit_id`, `m_low_limit_id` 4개의 필드로 구성되어있다
  - `m_creator_trx_id`를 통해 내가 레코드를 변경했을 경우 내가 볼 수 있는 것인지를 판단한다
  - `m_ids`를 통해 리드 뷰 생성 당시 활성화 중이던 다른 트랜잭션의 ID를 관리한다
  - `m_up_limit_id`(최소 활성 트랜잭션, m_ids 중 가장 작은 ID)를 통해 이 필드`보다`작은 트랜잭션의 ID는 이미 커밋됬던 안전한 트랜잭션이라는 것을 판단한다
  - `m_low_limit_id`(최대 트랜잭션 ID + 1)을 통해 리드 뷰 생성 당시에 발급된 가장 높은 트랜잭션 ID에 1을 더해 미래의 트랜잭션 인지를 판단한다
    - 즉 MVCC(REPEATABLE_READ)의 규칙은 아래와 같다
    - `m_up_limit_id` 이전의 트랜잭션은 언두 로그를 확인할 필요 없이 그대로 사용한다
    - `m_up_limit_id`와 `m_low_limit_id` 사이라면 `m_ids`에 속해있지 않다면 언두 로그를 확인하지 않는다(이미 완료됨)
    - `m_ids`에 속해있다면 언두로그를 통해 정보를 가져온다(진행 중)
    - `db_trx_id`가 `m_creator_trx_id`와 같다면, 내가 바꾼 정보라는 것을 인식하고 데이터의 가시성을 얻는다
    - 레코드의 `db_trx_id`가 `m_low_limit_id`보다 높다면 READ_VIEW가 만들어진 이후에 바뀐 정보이기에, 언두 로그를 통해 이전의 정보를 가져온다

#### InnoDB 버퍼 풀

- InnoDB 스토리지 엔진의 가장 핵심적인 부분으로, 디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시하는 공간이다
  - 쓰기 작업을 지연시켜 일괄 작업으로 처리할 수 있게하는 버퍼의 역할도 같이한다
- WAL을 효율적으로 사용하기 위해 사용되는 핵심 아키텍쳐이다
  - WAL을 통해 기록 후, 디스크에 바로 write를 하지 않고 Buffer pool에 메모리로 기록 후 disk 기록 여부를 flag로 표기해 DB의 속도를 유지한다
    - 아직 영속화 되지 않은 buffer pool의 공간을 `dirty page`라고 부른다
  - 영속화를 위해 flush는 비동기적으로 백그라운드 스레드를 통해 진행된다
- 언두/리두 로그와 InnoDB 버퍼 풀의 관계
  - DB는 클라이언트에 WAL(O(1))을 사용해 리두로그를 동기적으로 기록하고 `commit`을 반환한다
    - `commit`이 되었다면, 그 데이터는 반드시 리두로그를 통해 영속성을 보장받으며 이후 비동기적으로 buffer pool의 dirty page를 flush를 하는 백그라운드 스레드를 통해 디스크에 영속된다
    - 즉, 리두로그는 DB의 SSOT가 된다
    - 트랜잭션의 언두로그도 리두로그를 이용해 먼저 기록되고, 이후 백그라운드 스레드를 통해 언두로그 파일로 buffer pool의 dirty page를 flush시킨다
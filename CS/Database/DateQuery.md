# 날짜 조회 쿼리


- DATETIME
  - DATETIME 타입은 날짜와 시간을 모두 포함할 때 사용하는 타입
  - `YYYY-MM-DD HH:MM:SS`의 형태로 사용
  - 절대적인 시간을 표현할 때 사용된다
- TIMESTAMP 
  - `YYYY-MM-DD HH:MM:SS`형태로 사용
  - 상대적인 시간을 표현할 때 사용된다
  - 읽는 곳마다 해당 시간대로 보이도록 할 때는 TIMESTAMP를, 어디에서든 똑같이 보이도록 할 시간은 DATETIME을 사용하면 된다
    - 해당 이유로 TIMESTAMP 자료형을 많이 사용한다
- 저장된 것을 보면 형태 포맷팅된 String이라고 생각할 수 있지만, 이들은 실제 binary로 저장되어 있으며 String이랑 비교하기 위해서는 DB Engine자체 캐스팅이 필요하다

```sql
CREATE TABLE IF NOT EXISTS items (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
);
```

- (절대 하지 말아야할 방법)
  - `select * from created_date like '2025-12-05%'`
  - 직관적으로 봤을 때는 문제가 없어 보이지만, TIMESTAMP 자료형은 String이 포맷팅 된 자료형이 아니고 binary형태로 저장되어 있기에, 비교를 위해서 String으로 캐스팅하게되면 record마다 함수를 사용하는 것이기에 인덱스를 사용하지 못하고 데이터 풀스캔이 되어 굉장한 비효율이 발생한다
- (권장 방법)
  - `select * from created_date between '2025-12-05 00:00:00' and '2025-12-05 23:59:59'`
  - 해당 방식의 비교는 입력되는 파라미터를 캐스팅하기에, 인덱스를 사용할 수 있다
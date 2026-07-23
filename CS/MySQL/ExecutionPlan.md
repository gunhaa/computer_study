# Execution Plan, 실행 계획

- MySQL에서는 5.6버전 이전에는 메모리에 통계데이터를 보관했다
- 5.6버전 이후부터는 innodb_index_stats, innodb_table_stats 테이블에 통계 정보를 보관한다
- MySQL 8.0에서 도입된 히스토그램은 컬럼 값의 분포를 버킷(범위)단위로 알 수 있어, 옵티마이저가 정확한 판단을 가능하도록 한다
  - 인덱스가 있는 컬럼은 인덱스 통계를 활용하고, 히스토그램은 주로 인덱스가 없는 컬럼의 선택도를 추정하기 위해 사용된다
- MySQL의 실행 계획은 Cost기반이며, 기본 상수 값은 아래와 같으며, 관리자가 수동으로 변경할 수 있다
  - 실행 계획의 총 합은 `∑(연산별 동적 추정 수치 * 해당 연산의 기준상수)`로 계산된다 

#### mysql.server_cost

| cost_name | cost_value (기본값) | 설명 |
| :--- | :--- | :--- |
| `row_evaluate_cost` | 0.10 | 레코드 1건의 조건(WHERE)을 평가하는 비용 |
| `key_compare_cost` | 0.05 | 인덱스 키 값을 비교하는 비용 |
| `memory_temptable_create_cost` | 1.00 | 메모리 임시 테이블 생성 비용 |
| `memory_temptable_row_cost` | 0.10 | 메모리 임시 테이블에 1건 쓰는 비용 |
| `disk_temptable_create_cost` | 20.00 | 디스크 임시 테이블 생성 비용 |
| `disk_temptable_row_cost` | 0.50 | 디스크 임시 테이블에 1건 쓰는 비용 |

#### mysql.engine_cost

| cost_name | cost_value (기본값) | 설명 |
| :--- | :--- | :--- |
| `io_block_read_cost` | 1.00 | 디스크에서 데이터 블록 1개를 읽는 비용 |
| `memory_block_read_cost` | 0.25 | 메모리(버퍼 풀)에서 블록 1개를 읽는 비용 |

## 실행 계획

> 주요 사항 정리

### 1. 컬럼별 역할과 성능 체크 포인트

| 컬럼명 | 주요 역할 | 성능 체크 포인트 |
| :--- | :--- | :--- |
| `id` | SELECT 구문의 실행 순서 | 숫자가 클수록, 같은 숫자면 위쪽이 먼저 실행됨 |
| `select_type` | SELECT 구문의 성격/역할 | `DERIVED`, `UNCACHEABLE` 유무 확인 |
| `table` | 접근하는 테이블 (또는 별칭) | `<derived{id}>` 등 임시 테이블 스캔 여부 |
| `partitions` | 접근한 테이블 파티션 | 파티션 프루닝 작동 여부 |
| `type` | 데이터 접근(스캔) 방식 | `ALL` (풀 스캔) 회피 필요 |
| `possible_keys` | 옵티마이저가 검토한 후보 인덱스 | 적절한 인덱스가 후보에 올라왔는지 확인 |
| `key` |  실제 선택된 인덱스 | `NULL`이면 인덱스 미사용 |
| `key_len` | 사용된 인덱스 키의 길이 (Bytes) | 복합 인덱스 중 몇 개의 컬럼까지 탔는지 계산 |
| `ref` | 인덱스 비교에 사용된 값/컬럼 | `const`, `func`, 타 테이블 컬럼명 확인 |
| `rows` | 읽을 것으로 예상되는 레코드 수 | 수치가 터무니없이 높지 않은지 확인 |
| `filtered` | 필터링 후 남을 것으로 예상되는 비율(%) | `rows` × `filtered` = 최종 처리 건수 |
| `Extra` | 실행 방식의 부가 정보 | `Using filesort`, `Using temporary` 주의 |

### 2. 컬럼별 상세 설명

#### `id` (실행 순서)
- SELECT 구문에 부여되는 식별 번호
  - 숫자가 클수록 먼저 실행된다
  - 숫자가 같으면 위에서 아래 순서로 실행된다

#### `select_type` (SELECT 구문의 성격)
- `SIMPLE`: UNION이나 서브쿼리가 없는 가장 단순한 SELECT
- `PRIMARY`: 서브쿼리나 UNION이 포함된 복잡한 쿼리에서 가장 최상위 메인 SELECT
- `DERIVED`: FROM 절 내부에 사용된 서브쿼리 (메모리/디스크에 파생 임시 테이블 생성)
- `SUBQUERY`: WHERE / SELECT 절 등에 사용된 일반 서브쿼리 (상관관계 없음)
- `DEPENDENT SUBQUERY`: 바깥쪽 쿼리의 값을 전달받아 반복 실행되는 서브쿼리 (성능 주의)
- `UNION`: UNION 키워드로 연결된 두 번째 이후의 SELECT

#### `table` (대상 테이블)
- 실행 계획 단계에서 접근하는 테이블 이름 또는 별칭
- `<derived2>` 처럼 표기되면 `id=2`번 단계에서 생성된 파생 테이블을 읽고 있음을 의미한다

#### `type` (데이터 접근 방식 - 성능 핵심)
- 데이터를 가져오기 위해 테이블을 어떤 방식으로 읽는가를 나타내며, 오른쪽으로 갈수록 성능이 나쁘다

`const` → `eq_ref` → `ref` → `range` → `index` → `ALL`

- `const`: Primary Key나 Unique Key로 단 1건만 조회하는 최상 성능 방식.
- `eq_ref`: 조인 시 드라이빙 테이블의 값을 받아 피드라이빙 테이블의 PK/Unique Key로 단 1건만 찾을 때
- `ref`: Unique가 아닌 일반 인덱스(=) 조인 시 사용
- `range`: 인덱스를 사용한 범위 검색 (`<`, `>`, `BETWEEN`, `IN`, `LIKE 'a%'`)
- `index`: 인덱스 전체를 풀 스캔 (인덱스 파일 자체를 처음부터 끝까지 읽음)
- `ALL`: 풀 스캔, 테이블 전체 데이터를 디스크에서 다 읽음 (최우선 튜닝 대상)

#### `possible_keys` & `key` & `key_len` (인덱스 정보)

- `possible_keys`: 옵티마이저가 인덱스 사용을 고려한 후보 인덱스들
- `key`: 실제로 사용된 인덱스 (`NULL`이면 인덱스를 타지 않은 것)
- `key_len`: 채택된 인덱스에서 몇 Byte까지 활용되었는지를 나타낸다

#### `ref` (인덱스 조인 조건)

인덱스를 조회할 때 어떤 값과 비교했는지를 보여준다
- `const`: 상수 값과 비교 (PK, 혹은 Unique 컬럼 사용)
- `db_name.table.col`: 다른 테이블의 특정 컬럼 값과 조인 비교

#### `rows` & `filtered` (예상 데이터량)
- `rows`: 쿼리 조건을 만족하는 행을 찾기 위해 MySQL이 조사해야 할 것으로 예상하는 레코드 건수 (실제 결과 건수가 아님)
- `filtered`: `rows` 중에서 WHERE 조건 등으로 최종 채택될 레코드의 비율
  - e.g. `rows = 1000`, `filtered = 10.00`이라면, 1,000건을 읽어서 조건 필터링 후 100건만 남을 것으로 예상 된다는 뜻

### `Extra` (부가 정보 - 성능 핵심)
쿼리가 실행될 때 내부적으로 어떤 작업이 일어나는지 보여주는 지표

- `Using index` (커버링 인덱스 - 긍정)
  - 테이블 본문(데이터)에 접근할 필요 없이 인덱스 블록만 읽고도 쿼리를 완벽히 처리한 경우
- `Using index condition` (ICP - 긍정)
  - Index Condition Pushdown이 적용되어 스토리지 엔진 단에서 필터링을 미리 마친 경우
- `Using where` (주의/일반)
  - 스토리지 엔진에서 가져온 데이터를 MySQL 에서 추가로 필터링한 경우
  - MySQL은 스토리지 엔진에서 읽거나 저장을, MySQL 엔진에서 가공 또는 연산을 하는데, 여기서 MySQL단의 연산이 진행된다면 `Using Where`이 되는 것이다
- `Using temporary` (경고 - 튜닝 대상)
  - 쿼리 처리를 위해 임시 테이블을 메모리나 디스크에 생성한 경우 (주로 `GROUP BY`, `DISTINCT` 등에서 발생)
- `Using filesort` (경고 - 튜닝 대상)
  - 인덱스를 활용해 정렬하지 못해, 메모리나 디스크에 별도의 정렬 알고리즘을 수행한 경우
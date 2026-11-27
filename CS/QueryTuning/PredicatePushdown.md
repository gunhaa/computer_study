# Predicate Pushdown

- 조건절을 더 아래 단계의 쿼리에서 처리하도록 Pushdown하여 최적화하는 쿼리 튜닝 기법

## 1. 개요

`UNION` 연산이 포함된 권한 조회 쿼리에서 발생한 약 70초의 응답 지연을 분석하고, 옵티마이저가 자동으로 처리하지 못하는 Predicate Pushdown을 각 브랜치로 수동 적용하여 **0.1초**로 성능을 개선한 사례

- 개선 성과: 70s → 0.1s (약 700배 성능 향상)
- 주요 기법: 수동 Predicate Pushdown, Execution Plan 분석, Driving Table 전환(`org`) 및 Index Scan(`wg_id_idx`) 유도
- 참고: 테이블·컬럼명은 마스킹했고, 쿼리는 병목과 직접 관련된 부분만 남기고 나머지 컬럼·조건·조인은 제거함. 실행 계획도 결정적인 행만 발췌했으며 `rows`는 옵티마이저 추정치

---

## 2. 배경

### 2.1 시스템 구조 및 현상
- 사용처: Bootstrap 및 권한 캐싱 시점에 권한 데이터를 조회하는 핵심 쿼리
- 문제점: 로컬 및 운영 환경에서 시작 시마다 60~90초(평균 70초)의 병목 발생
- 구조적 한계: `admin_map`과 `member_map`이 각각 `account`와 조인된 뒤 `UNION`으로 묶이고, 그 결과(`map`)와 `org`가 `UNION` 병합 이후 외부에서 조인 및 필터링되는 구조

### 2.2 운영 및 비즈니스 제약
- 과도한 DB 부하로 인해 런타임 중 권한 캐시 동적 갱신 불가능
- 실시간 권한 반영이 불가능하여 서버 재시작/점검 시에만 변경 사항을 반영해야 하는 비효율 존재

---

## 3. 원인 분석

### 3.1 원본 쿼리 (Before, pseudo code)

```sql
select login_id, org_id
from ( admin_map ⨝ account
  union
  member_map ⨝ account ) as map
  ⨝ org o on map.workgroup_id = o.ext_key
where o.ext_key is not null
```

### 3.2 실행 계획 분석 (Before Execution Plan)

| id | table | type | possible_keys | key | rows | filtered | Extra |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | org o | ALL | - | - | 10 | 100% | where; temp |
| **1** | \<derived2\> | ref | auto_key1 | auto_key1 | 10,000 | 100% | using where |
| **2** | admin_map | ALL | login_id_idx | - | 30,000 | 100% | - |
| **2** | account | eq_ref | username_idx | username_idx | 1 | 100% | using index |
| **4** | account | index | username_idx | username_idx | 70,000 | 100% | using index |
| **4** | member_map | ref | login_id_idx | login_id_idx | 10 | 100% | index cond. |
| **6** | \<union2,4\> | ALL | - | - | - | - | temporary |

#### 병목 원인
1. **조건이 `UNION` 병합 이후에 걸림**: `org o`와의 조인 및 `where o.ext_key is not null` 조건이 `UNION` 내부 브랜치로 전진 배치되지 못함
2. **대량 탐색 및 임시 테이블/동적 인덱스 생성**:
   - `account` 약 70,000 rows를 기준으로 1:N(10) 관계인 `member_map`을 탐색하며 대량 데이터 발생
   - `UNION` 결과 약 10,000 rows를 임시 테이블(`<union2,4>`)로 만들고 `<auto_key1>`을 즉석 생성한 뒤에야 조건이 걸림
3. **탐색 규모**: Driving Table이 `admin_map/account` 기준 `[30,000 rows * 1 rows + 70,000 * 10 rows ( - 중복제거)]` 구조로 동작

---

## 4. 해결 방안: 수동 Predicate Pushdown

- 옵티마이저에 의존하지 않고 조인과 조건절을 각 `UNION` 브랜치 안으로 전진 배치
- 핵심 원리: I/O가 사라진 것이 아니라 **읽는 양이 `org(10 rows)`에서 시작하는 구조**로 전환
- UNION의 합집합적 특성 및 조인/조건 위치 변경에 따른 결과/비용 리스크로 옵티마이저가 원하는 형태로 Pushdown을 적용하지 않기에 직접 브랜치 안으로 내림
- 변경 전/후 쿼리의 결과 동등성을 확인하기 위해 dev 환경에서 일주일 동안 다양한 데이터를 넣어 기존 쿼리와 결과를 비교 검증함

### 4.1 개선 쿼리 (After, pseudo code)

```sql
select login_id, org_id
from (
  admin_map ⨝ account
  ⨝ org o on admin_map.workgroup_id = o.ext_key
  where o.ext_key is not null
  union
  member_map ⨝ account
  ⨝ org o on member_map.workgroup_id = o.ext_key
  where o.ext_key is not null
) as map

```

---

## 5. 튜닝 후 실행 계획

| id | table | type | possible_keys | key | rows | filtered | Extra |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | \<derived2\> | ALL | - | - | 200 | 100% | - |
| **2** | org o | ALL | - | - | 10 | 100% | where; temp |
| **2** | admin_map | ref | wg_id_idx, login_id_idx | wg_id_idx | 10 | 100% | index cond. |
| **2** | account | eq_ref | username_idx | username_idx | 1 | 100% | using index |
| **3** | org o | ALL | - | - | 10 | 100% | where; temp |
| **3** | member_map | ref | wg_id_idx, login_id_idx | wg_id_idx | 200 | 100% | index cond. |
| **3** | account | eq_ref | username_idx | username_idx | 1 | 100% | using index |
| **4** | \<union2,3\> | ALL | - | - | - | - | temporary |

- **Driving Table 전환**: 약 10행인 `org`가 Driving Table이 되어 두 핵심 테이블(`admin_map`, `member_map`)이 `wg_id_idx` `ref` 조회 수행
- **Driven Table 처리 축소**: `member_map`/`admin_map`/`account`는 Driven Table로서 앞선 조인 결과를 받아 조인을 수행하며 처리 건수 대폭 감소
- **탐색 규모**: Driving Table이 `org` 기준 `[10 rows * 10 rows + 10 * 200 rows ( - 중복제거) ]` 구조로 축소

---

## 6. 최종 성과 및 결론

- 성능 개선: `70s` → `0.1s` (약 700배 성능 향상)
- 핵심 변화: I/O가 사라진 것이 아니라 읽는 양이 `org(10 rows)`에서 시작하는 구조로 개선
- 비즈니스 가치: 런타임 중 캐시 동적 갱신이 가능해져 실시간 권한 반영 요구사항 해결
# MapReduce

> paper: https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf

> lab: https://pdos.csail.mit.edu/6.824/labs/lab-mr.html

> git: https://github.com/gunhaa/map_reduce

- 6.824 distributed systems 강의의 lab1 map_reduce 실습

## 핵심

- 구글에서 대용량 데이터를 병렬처리 하기 위한 논문으로써, 수 천대의 컴퓨팅 자원을 낭비없이 쓰기위한 방법론
  - 특정 노드의 문제 발생시 해결, 노드 간의 네트워크 대역폭 문제, 머신 내부의 동시성/전체 네트워크의 동시성 모두를 다뤄야한다

## 구현

### Map, Reduce

- 대규모 분산 클러스터에서 복잡한 병렬 계산을 개발자가 직접 신경 쓰지 않고, 단순한 두 개의 함수(Map과 Reduce)만 구현하여 처리할 수 있도록 추상화한 것
  - map: 입력 데이터를 받아서 (Key, Value) 형태의 중간 데이터로 변환
    - e.g. 문서들을 읽어 단어별 카운트를 추출
  - reduce: 동일한 키(Key)를 가진 데이터들을 모아서 공통된 연산을 수행하고 최종 결과를 도출
    - e.g. 같은 단어끼리 모두 더해 최종 빈도수 계산

### Lab

- TODO(bitcopark)
# I/O 튜닝 심화

- I/O 튜닝이 곧 SQL 튜닝이라고 불릴만큼 중요하다
- DB가 결과를 얻기 위한 과정은 간단하다
  1. 입력 받은 SQL을 파싱한다(soft/hard)
  2. 1의 결과를 바탕으로 Memory를 탐색 후 결과 집합을 사용자에게 출력한다
- 여기서 핵심은 2의 방법이다
  - 저장된 file system을 탐색하기전에 in-memory 탐색을 한다
  - buffer cache를 탐색해 결과가 있다면 해당 포인터를 사용해 빠른 I/O를 통해 결과를 가져온다
    - cache는 hashmap, 충돌 방지를 위한 linkedList로 구성되어 있으며 각 버킷마다 충돌 방지를 위한 latch가 존재한다(버킷 변경 lock)
  - full scan의 경우 multiblock I/O를, idx scan의 경우 singleblock I/O를 한다
    - 이름에서 알 수 있듯이 multiblock은 연속적인 많은 수의 record를 가져올때 강점을 발휘하지만, 적은 수 I/O에서는 비효율적이다
      - multiblock은 (주변의) 데이터를 한번에 가져오는 것이기에, 필요 없는 데이터를 가져오면 cache가 오염되며, 연속적이지 않은 데이터를 가지고오지 못한다
    - singleblock은 적은 수를 가져올 때 효율적이지만, (연속적인)많은 수에서는 불리하다
  - 캐싱된 포인터를 사용하고 없는 record들은 I/O를 통해서 가져온다
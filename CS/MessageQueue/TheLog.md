# The Log

> 원본: https://www.linkedin.com/blog/engineering/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying<br>
> 관련 영상: https://www.youtube.com/watch?v=v2Eab3U0W9s

- 원본 글은 kafka를 만들기 이전, 이벤트 스트리밍에 대한 핵심 개념을 담은 포스트이다
  - 이 글에서 핵심은 상태(state)는 1급 개념이 아니며, 상태는 변경 이벤트(log)의  결과(fold)일 뿐이다
  - 즉, log만 가지고 state를 복구할 수 있다
- Log란 `A log is perhaps the simplest possible storage abstraction. It is an append-only, totally-ordered sequence of records ordered by time`이다
  - 가장 단순한 저장 추상화이다
  - 어플리케이션에서 생상되는 로그와는 별도의 개념이다
    - `The application log is a degenerative form of the log concept I am describing. The biggest difference is that text logs are meant to be primarily for humans to read and the 'journal' or 'data logs' I'm describing are built for programmatic access.`
    - 핵심은 Log는 상태 재구성의 원본으로 사용이 가능하다는 것이다 
-  `The log is the record of what happened, and each table or index is a projection of this history into some useful data structure or index.`
  - 일반적으로 우리는 테이블이 진짜 데이터이며, 로그는 보조장치라고 생각하지만 저자는 이를 뒤집는다
```
전통적 관점:            이 글의 관점:

  [테이블] ← 진실         [로그] ← 진실 (history of what happened)
     │                      │
     ↓ (보조)               ├──→ [테이블]      (projection)
  [WAL]                     ├──→ [B-tree 인덱스] (projection)
                            ├──→ [전문검색 인덱스] (projection)
                            └──→ [머티리얼라이즈드 뷰] (projection)
```
# Clox 

## 특징

- 하이레벨 언어(javascript, lua, 스킴의 두가지 측면을 공유한다)
- 동적 타입 언어
  - 변수에 어떤 값이라도 담을 수 있고, 하나의 변수는 임의의 시간에 타입이 다른 값을 보관할 수 있다
  - 구현이 정적타입 언어에 비해 쉽다
- 자동 메모리 관리(GC, garbage collection)
  - 하이레벨 언어의 존재의의는 에러가 발생하기 쉬운 로우레벨의 일을 덜어주는 것이다
- 타입
  - Boolean, Number, String, Nil
  - Number: 배정도 부정소수점(double-precision floating point)한 종류의 숫자만 사용한다
  - String: 다른 언어처럼 큰따옴표로 묶는다
  - Nil: 다른 언어의 null과 같다, 정적 타입 언어는 없다면 더 이득이겠지만, 동적 타입 언어에서 null을 제거하면 그냥 갖고 있는 것보다 더 성가실때가 많다
- 표현식(expression)
  - &&, || 연산에서 short circuit이 존재한다
  - &, | 비트 연산자는 사용하지 않는다
  - ()를 이용한 우선순위 연산을 사용한다
- 문장(statement)
  - 뒤에 세미콜론(;)을 붙이면 문장으로 승격된다(promote)
    - 이를 표현문(expression statement)라고 한다
  - 여러 문장처럼 작동시키려면 block으로 감싼다
- 변수
  - var문으로 시작한다
    - 초기자(initialize)를 생략하면 변수의 기본값은 nil이다
    - nil(null)이 존재하지 않는다면 어떤 값으로 변수를 초기화해야하는 문제가 있다
  - C/Java의 스코핑 규칙이 적용된다
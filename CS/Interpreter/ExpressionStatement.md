# 프로그래밍 언어에서 Statement/Expression/Token/Lexeme/Literal

- Statement -> Expression -> Token -> Lexeme/Literal
  - 상위 개념이 하위 개념을 포함하는 구조로 되어있다
- Literal 
  - 리터럴
  - Value가 있는 Lexeme을 다르게 부르는 명칭이다
  - 렉심이 나타내는 실제 값
    - 소스코드에 0x10이라고 써있다면 렉심은 `0x10`이지만, 리터럴(값)은 16이 된다
    - 코드에 직접 하드코딩된 값
- Lexeme
  - 렉심(어휘소)
  - 문자를 스캐닝해서 가장 작은 시퀀스로 묶는 작업의 핵심이다
  - 소스 코드의 "생모습" 그 자체인 문자열
  - 렉심을 가져와 다른 정보와 함께 묶으면 토큰이 된다
- Token
  - 프로그래밍 언어의 최소 단위이며, 아래 정보를 모두 포함한다
  - 렉심에 의미(Type)를 부여한 객체
  ```java
  // 토큰의 타입(컴파일러를 위한 정보)
  final TokenType type;
  // 렉심(어휘소) e.g. var, print, =, !, a(identifier)..
  final String lexeme;
  // number 같은 lexeme 타입이라면 값이 필요하다(lexeme = 1, literal = 1.0)
  final Object literal;
  ```
- Expression
  - 표현식
  - 값이 존재하는 토큰의 묶음이다(Literal- 값/ Grouping- 값의 묶음/ Binary- 두 개의 값/ Unary - 하나의 값)
  - 평가되어 하나의 값이 된다
- Statement
  - 프로그래밍 언어를 이루는 문장
  - Expression이 모여서 문장이 된다(var a = 1;/ print 213; 같은 형태)
  - 실행할 수 있는 최소의 단위이다
  - 모든 Expression은 Statement의 일부가 될 수 있지만(Expression Statement), 모든 Statement가 Expression인 것은 아니다

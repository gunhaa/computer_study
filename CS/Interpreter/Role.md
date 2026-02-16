# Lox Interpreter 객체의 역할

## 구성

- Lox
  - Lox언어의 엔트리 포인트로써, Prompt/File 모드에 따른 실행방법을 분기한다
  - 실행에 대한 환경, 결과의 상태를 저장한다
- Scanner
  - script/prompt를 읽어 `List<Token>`을 생성한다
- Parser
  - Scanner가 만든 `List<Token>`을 이용해, Expression들을 이용해 Abstract Syntax Tree(AST)를 생성한다
  - 올바름을 판정하지 않으며, 트리로 만들고 에러처리와 synchronize에 집중한다
- Token
  - 토큰에 대한 정보를 저장한다(토큰 타입, lexeme(형태소/ text), 리터럴(value), 에러 발생 라인)
- **Expr(표현식)
  - 추상 클래스 Expr에 행동을 하는 Visit Interface로 구성되어있으며, 데이터를 보관하고 있는 Binary(operator로 연결된 Expr), Unary(`+,-`), Grouping(`()`그룹핑), Literal(`value`)이 존재한다
  - Expression을 원하는 클라이언트는 Visit Interface를 구현해서 Expr의 데이터를 사용한다
  - 자신이 어떤 종류의 식인지 알고 있으며, 손님(Visitor)을 맞이할 준비가 된 구조체이다
    - Expr의 Visitor 패턴은 데이터의 구체적인 타입(Binary인지 Literal인지 등)을 노출하지 않고도 각 타입에 맞는 로직을 수행할 수 있게 해준다
    - Expr 클래스 안에 Expr을 넣음으로써, 재귀적 구조를 표현한다
  - Expression은 값이 존재하는 토큰의 묶음이며, 평가되어 하나의 값이 된다(Literal- 값/ Grouping- 값의 묶음/ Binary- 두 개의 값/ Unary - 하나의 값)
- Statement
  - Expr이 모여 Statement가 된다
  - Expr은 값이며, Statement는 동작(Action)이다
  - Expr은 평가되어 하나의 값을 생성하는 토큰의 묶음이며, Statement는 그 값을 이용해 시스템에 변화를 주거나 프로그램의 흐름을 제어하는 실행 단위이다
- TokenType
  - Token의 모든 타입을 Enum으로 표현한다
- AstPrinter
  - Expr의 행동(Visit interface)를 구현해서 expression 디버깅용 Print를 담당한다
- Interpreter
  - Expr의 행동(Visit interface)를 구현해서 parser가 만든 expression ast를 해석(interpret)한다
  - Runtime 환경을 관리한다
- Resolver
  - 변수의 참조 범위(Scope)를 확정한다
    - Stack과 Map을 이용해 변수의 재사용을 감지한다
    - 정적 시맨틱 분석을 시행해 문법을 검사한다
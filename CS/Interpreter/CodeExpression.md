# 코드 표현

- Scanner를 통해 lexeme로 token화 시킨 후, 이를 이용해 코드 표현을 만든다
  - 코드 표현은 파서가 만들기 간단하고 인터프리터가 소비하기 쉬워야한다
  - 1+2*3-4
    - 해당 식은 트리구조로 만들어져, 후위 순회로 처리된다(우선 순위에 대한 설정 필요, 산술의 규칙은 PEMDA)

## Lox Expression Grammar

```plaintext
expression -> literal | unary | binary | grouping;
literal -> NUMBER | String | "true" | "false" | "nil";
grouping -> "(" expression ")";
unary -> ( "-" | "!" ) expression;
binary -> expression operator expression;
operator -> "==" || "!=" | "<" | "<=" | ">" | ">=" | "+" | "-" | "*" | "/";
```
- unary expression(단항식): !는 not, -는 숫자를 음수로
- binary expression(다항식): 중위 연산자(+,-,*,/), 논리 연산자(==, !=, <=, <, >=, >)
- 문법은 재귀적이다
- 결국 표현의 자료구조는 트리 모양이되며, 이를 syntax tree(구문 트리)라고 한다

# 재귀 하향 파싱, Recursive descent

- 재귀 하향은 파서를 만드는 가장 간단한 수단이다
- 재귀 하향 파싱 예제 코드
  - while문을 이용해 greedy하게 좌항 결합을 시도한다
  - equal(동등)이 comparison(비교) 보다 우선순위가 높음을 표현한 코드이다
  - 좌결합성 우선순위(동일한 우선순위를 가졌다면 좌측부터 처리)가 적용된 AST트리를 만들어낸다
  - Parser의 책임은 Abstract Syntax Tree(AST)생성이 끝이다

```java
private Expr equality() {
    Expr expr = comparison();
    // !=, ==
    while(match(BANG_EQUAL, EQUAL_EQUAL)) {
        Token operator = previous();
        Expr right = comparision();
        expr = new Expr.Binary(expr, operator, right);
    }

    return expr;
}

private Expr comparison() {
    Expr expr = term();

    while (match(GREATER, GREATER_EQUAL, LESS, LESS_EQUAL)) {
        Token operator = previous();
        Expr right = term();
        expr = new Expr.Binary(expr, operator, right);
    }
}

// 특정 상황, 토큰을 소모한다(advance)
private boolean match(TokenType... types) {
    for (TokenType type : types) {
        if(check(type)) {
            // 토큰 소모
            advance();
            return true;
        }
    }
    return false;
}
```


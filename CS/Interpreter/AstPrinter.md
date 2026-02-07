# AstPrinter

- 데이터와 로직을 분리시키는 잘 만든 예제이다
- Visitor패턴은 프레임워크, 컴파일러에서 사용되는 데이터/로직을 분리시키는 디자인 패턴이다
  - OOP 언어에서 함수형 언어의 컬럼 확장 방식을 구현하는 패턴이다

```java
// Expr.java
package interpreter.lox;

abstract class Expr {

    abstract <R> R accept(Visitor<R> visitor);

    interface Visitor<R> {
        R visitBinaryExpr(Binary EXPR);

        R visitGroupingExpr(Grouping EXPR);

        R visitLiteralExpr(Literal EXPR);

        R visitUnaryExpr(Unary EXPR);
    }

    static class Binary extends Expr {
        Binary(Expr left, Token operator, Expr right) {
            this.left = left;
            this.operator = operator;
            this.right = right;
        }

        @Override
        <R> R accept(Visitor<R> visitor) {
            return visitor.visitBinaryExpr(this);
        }

        final Expr left;
        final Token operator;
        final Expr right;
    }

    static class Grouping extends Expr {
        Grouping(Expr expression) {
            this.expression = expression;
        }

        @Override
        <R> R accept(Visitor<R> visitor) {
            return visitor.visitGroupingExpr(this);
        }

        final Expr expression;
    }

    static class Literal extends Expr {
        Literal(Object value) {
            this.value = value;
        }

        @Override
        <R> R accept(Visitor<R> visitor) {
            return visitor.visitLiteralExpr(this);
        }

        final Object value;
    }

    static class Unary extends Expr {
        Unary(Token operator, Expr right) {
            this.operator = operator;
            this.right = right;
        }

        @Override
        <R> R accept(Visitor<R> visitor) {
            return visitor.visitUnaryExpr(this);
        }

        final Token operator;
        final Expr right;
    }

}
```

```java
// AstPrinter.java
package interpreter.lox;

public class AstPrinter implements Expr.Visitor<String>{
    @Override
    public String visitBinaryExpr(Expr.Binary expr) {
        return parenthesize(expr.operator.lexeme, expr.left, expr.right);
    }

    @Override
    public String visitGroupingExpr(Expr.Grouping expr) {
        return parenthesize("group", expr.expression);
    }

    @Override
    public String visitLiteralExpr(Expr.Literal expr) {
        if (expr.value == null) return "nil";
        return expr.value.toString();
    }

    @Override
    public String visitUnaryExpr(Expr.Unary expr) {
        return parenthesize(expr.operator.lexeme, expr.right);
    }

    private String parenthesize(String name, Expr... exprs) {
        StringBuilder builder = new StringBuilder();

        builder.append("(").append(name);
        for (Expr expr : exprs) {
            builder.append(" ");
            builder.append(expr.accept(this));
        }
        builder.append(")");
        return builder.toString();
    }

    String print(Expr expr) {
        return expr.accept(this);
    }

    public static void main(String[] args) {
        Expr expression = new Expr.Binary(
                new Expr.Unary(
                        new Token(TokenType.MINUS, "-", null, 1),
                        new Expr.Literal(123)
                ),
                new Token(TokenType.STAR, "*", null, 1),
                new Expr.Grouping(
                        new Expr.Literal(45.67)
                )
        );
        System.out.println(new AstPrinter().print(expression));
    }
}
```

- 오버로딩은 명확하지 않아 피한다
  - 하지만 일반적인 Visitor 패턴은 static dispatch를 이용해 구현한다
  - 오버 라이딩을하면 dyn dispatch를 사용해, 런타임에 타입을 결정해서 가져올 수 있다(java)
- this를 넘겨 실행시킬 메서드의 힌트를 준다
- 가지고있는 데이터 내부 클래스를 사용해 구현된 로직을 실행시켜 OCP와 SRP를 완벽하게 만족시킨다
- recursive call로 print를 한다
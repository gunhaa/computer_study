# PECS (Producer Extends, Consumer Super)

> java의 super, extends에 관한 공식이다(from effective java item31)

- 생산자(Producer)는 extends를 사용하고, 소비자(Consumer)는 super를 사용한다
  - 이것이 PECS(Producer-Extends, Consumer-Super)이다
- Producer
  - 읽기(Read / Produce)를 위해 컬렉션에서 꺼낸 객체는 최소한 T 타입의 객체(혹은 그 상위 타입)임이 보장된다
  - 따라서 안전하게 T 타입으로 참조할 수 있다
  - 데이터를 외부에 제공(생산, readonly이기에 데이터를 제공)
- Consumer
  - 컴파일러가 컬렉션의 실제 타입을 알 수 없기 때문에, 어떤 하위 타입의 객체를 넣어도 타입 불일치로 인한 런타임 오류가 발생할 위험이 있다
  - 자신이 다룰 수 있는 데이터의 범위를 넓혀 외부에서 들어오는 데이터를 안전하게 받아 저장(소비)

## 예제

```java
package org.example.item31;

// PECS
public class Main {

    private static class Parent {
        public void doParent() {
            System.out.println("parent doing");
        }
    }

    private static class FirstChild extends Parent{
        public void doFirstChild() {
            System.out.println("first child doing");
        }
    }

    private static class SecondChild extends Parent{
        public void doSecondChild() {
            System.out.println("second child doing");
        }
    }

    static void main() {
        // of 는 생성이기에, java extends가 producer를 readonly로 만드는 문제를 해결하지 못하고 컴파일되는게 정상 동작이다
        List<? extends Parent> producer = List.of(new FirstChild(), new SecondChild());
        for (Parent p : producer) {
            p.doParent();
            try {
                FirstChild fc = (FirstChild) p;
                fc.doFirstChild();
            } catch (ClassCastException e) {
                e.printStackTrace();
            }
        }

        List<? extends Parent> extendsList = new ArrayList<>();
//        extendsList는 어떤 타입의 추가도 불가능하다(null은 가능하다)
//        오직 readonly를 생성해내는 타입이 ? extends Parent이다
//        extendsList.add(new Parent());
//        extendsList.add(new FirstChild());
//        extendsList.add(new SecondChild());

        List<? super Parent> superList = new ArrayList<>();
        // parent를 가지는 모든 타입을 추가해도 문제가 생기지 않는다
        superList.add(new Parent());
        superList.add(new FirstChild());
        superList.add(new SecondChild());

        /*
        <? super T>의 '쓰기 전용' 원칙은 add()를 제외한 모든 읽기 작업을
        Object로 제한하여 타입 안정성을 지키는 것이며,
        프로그래머가 강제로 캐스팅하는 것까지는 막지 못한다
        이 강제 캐스팅은 제네릭 시스템의 문제가 아니라 일반적인 객체 지향의 타입 예측 실수에 해당한다
        */
        // 문제가 있지만 컴파일 시점에 막지 못한다, 이 오류는 런타임 시점에 잡히는 프로그래머의 실수이다
        // 컴파일 상으로 막지는 않지만 계약만 준수해서 사용한다면 사용 가능하다
        Parent castingParent = (Parent) superList.get(0);
        try {
            SecondChild castingChild = (SecondChild) superList.get(0);
        }catch (ClassCastException e) {
            e.printStackTrace();
        }
    }
}
```


# 공변, 불공변

> from effective java item28

- 변성(Variance)은 컴퓨터 과학의 타입 이론에서 나온 개념이다 

|개념|설명|
|------|---|
|공변|Child가 Parent의 하위 타입이면, Child[]는 Parent[]의 하위 타입이다|
|불공변|List\<Child> 와 List\<Parent>는 관계가 없다|

- 자바에서 배열은 공변이고, 제네릭은 불공변이다

```java
// java의 배열은 공변이기에 해당 메서드가 문제없이 컴파일된다
public static void print(Object[] arr) {
    for(Object i : arr) {
        System.out.println(i);
    }
}

public static void main(String[] args) {
    String[] strings = {"1", "2", "3"};
    print(strings);

    Integer[] integers = {1, 2, 3};
    print(integers);
}
```

```java
// java의 제네릭은 불공변이기에 아래 코드는 컴파일 에러가 발생한다
public static void print(List<Object> arr) {
    for(Object i : arr) {
        System.out.println(i);
    }
}

public static void main(String[] args) {
    List<String> strings = List.of("1", "2", "3");
    print(strings); // 컴파일 에러

    List<Integer> integers = List.of(1, 2, 3);
    print(integers); // 컴파일 에러
}
```

## java가 배열과 제네릭에서 공변성이 다른 이유

- 원래 java에는 제네릭이 존재하지 않았기에 발생한 문제이다
  - 배열은 C++ 스타일의 유연성을 제공하기 위해 공변(Covariant)으로 설계되었다
  - 이 설계는 컴파일 시점에는 타입 오류를 잡아내지 못하고, 런타임에 `ArrayStoreException`을 발생시켜야 하는 근본적인 타입 안정성 문제를 안고 시작했다
    - C++의 배열도 같은 문제를 가지고있어 C++ 개발자도 배열의 사용을 매우 위험하게 간주하고 `std::vector`와 같은 제네릭 컨테이너(불공변)를 주로 사용한다
- Java는 시간이 지나면서 배열의 런타임 타입 오류 문제와 컬렉션 사용의 불편함(`Object`로 꺼내서 강제 형변환 필요)을 인식했다(`new ArrayList()`로 생성하기에, 타입을 알 수 없으며 다른 타입을 넣더라도 모두 들어가고, 런타임에 알 수 있다)
  - 제네릭 설계자들은 배열의 실수를 반복하지 않기 위해 타입 안정성을 최우선으로 두었다
  - 그 결과, 제네릭 컬렉션(List\<T>)은 불공변(Invariant)으로 설계되어 컴파일 시점에 모든 타입 오류를 포착하도록 했다
# Chapter2. 프로그램이 실행되었지만, 뭐가 뭔지 하나도 모르겠다

## Section 2.1. 운영체제, 프로세스, 스레드의 근본 이해하기

### CPU

- 모든 것은 CPU에서 시작된다
  - CPU의 동작은 두 가지 밖에 없다
    1. PC Register를 통해서 메모리에서 명령어(instruction)를 하나 가져온다(dispatch)
    2. 이 명령어를 실행(execute)한 후 다시 1로 돌아간다
  - PC Register의 초기값은 entry point에서 결정된다(일반적으로 main함수)
  - CPU는 프로세스/스레드가 무엇인지 알지 못한다

### 운영체제

#### Operation System이 없던 시절의 문제

- 운영체제가 없이 순수하게 프로그램을 실행한다면 다음의 문제가 생긴다
  1. 프로그램을 적재할 수 있는 크기의 메모리를 직접 찾아야하며, CPU Register를 직접 초기화하고 함수의 entry point를 찾아서 PC Register를 설정해야한다
  2. 한 번에 하나의 프로그램만 실행할 수 있으며, 다중 코어의 사용이 사실상 불가능하다
  3. 모든 프로그램이 사용할 하드웨어를 직접 특정 드라이버와 연결해야하며, 그렇지 않으면 프로그램이 외부 장치를 전혀 사용할 수 없다
    - 사운드를 사용한다면 사운드 카드 드라이버, 네트워크 카드를 사용한다면 네트워크 카드 드라이버, 네트워크로 통신하려면 TCP/IP 스택 소스코드에도 연결해야 한다
    - print함수를 이용해 helloworld를 사용하기 위해서도 print함수를 직접 구현해야한다
- 이는 운영체제가 없던 1950-60년대 프로그램 작성 방식이며, 당시의 UX는 매우 끔찍했다

#### Operation System이 해결하는 문제

- operation system은 프로그램 여러개를 동시에 실행 시킬 수 있으며, 다중 코어를 사용 가능하도록 만든다
  - 프로그램에서 필요한 연산은 CPU의 연산속도보다 느리기에 CPU의 idle시간을 이용해 여러 프로그램의 동시 실행을 한다
  - 이 처리를 위해서 process라는 이름의 구조체를 만들며, context라는 필드를 통해 동시 실행시 자신의 상황을 저장한다
  - 이제 모든 프로그램은 실행된 후 process라는 이름으로 관리된다
  - CPU가 process사이를 충분히 빠르게 전환하는 한, CPU가 하나 뿐인 시스템에서도 수 많은 프로세스를 동시에 실행하거나 적어도 동시에 실행 중인 것 처럼 보일 수 있다(multi-tasking)
  - 프로그램을 자동으로 적재해주는 도구와 multi-tasking을 실현해주는 process관리 도구의 이름을 operation system이라고 한다
- 즉 고급 프로그래밍 언어, 컴파일러, 링커, 운영체제는 모두 프로그래머의 생산성을 높이기 위한 소프트웨어이다

#### Thread가 없던 시절의 문제

- 오래걸리는 function A, function B를 처리해야 후 값을 더해야 할 경우 하나의 프로세스에서는 문제가 생긴다
  - 병렬 실행이 불가능하기에, 모든 코드의 실행을 대기해야 한다
  - 이를 극복하기 위해선 2개의 process가 생성된 후 IPC를 통해 정보를 교환해야 한다
- 이 다중 process programming에는 단점이 있다
  1. process를 생성할 때 비교적 큰 overhead가 있다
  2. process마다 자체적인 주소 공간이 있기 때문에 process간의 통신은 복잡하다

### Thread가 해결하는 문제

- process의 주소 공간을 공유하는 독립적인 실행 흐름을 만들어내는 방법이 필요 했고, 이 단위가 thread가 되었다
  - thread는 process보다 훨씬 가겹고 생성속도가 빠른 이유이다(주소 공간의 공유)
  - 이런 이유로 thread를 light weight process(LWP)라고 부르기도 한다
- thread를 사용해 다중 코어로 CPU를 최대한 사용할 수 있다
  - 이 것이 고성능과 높은 동시성의 기초가 된다
  - 단일 코어에서도 multi-thread를 사용할 수 있다
    - thread가 운영체제 계층에서 구현되며, 코어 갯수와 무관하기 때문이다
  - CPU가 기계 명령어를 실행할 때도 실행중인 기계 명령어가 어떤 스레드에 속해있는지 인식하지 못한다
- thread를 통해 실행흐름의 여러 execution entry point가 생기게 된다
  - 즉, 모든 thread의 실행흐름을 저장하기 위해 stack area가 여러개 필요하다

## Section 2.4. 프로그래머는 코루틴을 어떻게 이해해야 할까?

### Coroutine란 무엇인가?

```python
# 정의
def func():
    print("A")
    yield
    print("B")
    yield
    print("C")
# 사용
def A(): 
    co = func()
    next(co)
    print("in functionA")
    next(co)
# 결과
# A
# in functionA
# B
```

- 함수의 제어권 이용방법이며, 일반적으로 언어에서 `yield`라는 예약어로 구현된다
- 프로그래머가 운영체제와 유사한 역할을 할 수 있게 하는 방법이다
  - 함수를 사용자가 스케줄링한다
- Coroutine이 특별한게 아닌, 함수자체가 Coroutine의 특별한 예에 불과하며, 연결 시작 지점이 하나만 존재하는 Coroutine이다
- 함수를 저장할때 Coroutine Context를 같이 저장해 놓는 방법으로 구현한다
  - 메모리가 무한하다면 Coroutine 무한히 구현할 수 있다

### Coroutine을 통해 무엇을 할 수 있는가?

- Coroutine을 사용해 동기식 코드 스타일로 직관적이게 코드를 작성하면서도, 내부적으로는 고성능의 비동기 Event 기반 시스템을 구현할 수 있다.
  - e.g. Python의 asyncio, JavaScript의 async/await, Go의 Goroutine 등 현대 비동기 프레임워크의 근본 구조가 됨

### Section 2.8. 높은 동시성과 고성능을 갖춘 서버 구현

- 사용자의 요청을 처리하기 위한 서버의 발전 순서와 장/단점

#### 1. 부모 프로세스가 요청을 수락하고, fork()를 통해 자식 프로세스를 생성

- 장점
  - 프로그래밍이 간단해서 이해가 쉽다
  - 개별 프로세스의 주소 공간이 격리되어 있어 하나의 프로세스에 문제가 발생하더라도 다른 프로세스에 영향을 미치지 않는다
  - 다중 코어 리소스를 최대한 활용할 수 있다
- 단점
  - 프로세스의 주소 공간이 격리 되어 있다는 것은 장점이지만, 단점도 될 수 있다. 프로세스 간 통신이 필요할떄 난이도가 올라간다
  - 프로세스를 생성할때 부담이 크고, 프로세스의 빈번한 생성과 종료는 시스템 부담을 증가시킨다

#### 2. 다중 스레드

- 1thread - 1request 모델이다(thread-per-connection)
- 연결은 I/O multiplexing을 사용(Java NIO)하고 어플리케이션에선 다중 스레드(스레드풀)을 사용한 방식으로 구현된 것이 spring framework이다
- 장점
  - 사용자 규모가 크지 않고 동시 접속자가 적은 경우, 구조가 직관적이며 충분히 안정적으로 처리가 가능하다
- 문제
  - thread는 주소 공간을 공유하기 대문에 thread간 통신에 있어서 편리함을 제공하지만 많은 문제를 일으킨다
    - thread가 문제가 생긴다면 모든 스레드와 프로세스가 한번에 강제 종료된다
    - 공유 리소스를 잘못 사용한다면 race condition이 발생한다
  - 많은 요청이 들어올 경우 다중 스레드의 recv()로는 감당이 어렵다
    - 만약 클라이언트가 연결만 유지하고 데이터를 안 보내면, 그 스레드는 recv()에 묶여서 다른 일을 전혀 못 한다

#### 3. I/O Multiplexing - select을 이용한 single thread eventloop

- select system call이란?
  - 많은 request를 recv()로 감당이 어렵다
  - 이를 해결하기 위해 I/O Multiplexing을 활용한다
    - 요청이 완료됬을시, kernel에서 준비 통지를 받는다
- 장점
  - 적은 수(단일)의 스레드만으로도 수 많은 클라이언트 요청을 효율적으로 관리할 수 있다
- 문제
  - select system call은 C10K 문제가 생기기 이전의 방식이다
    - 많은 요청을 처리하기 위해 설계되었지만, 10K의 요청을 감당하기 위해 설계 된 방식이 아니다
  - 관리하는 socket을 list로 관리하는 방식이기에, 완료가 발생한 socket을 찾기 위해 관리 중인 socket을 O(N)으로 순회하는 문제가 있다
  - 구조적으로 한 번에 감시할 수 있는 socket의 최대 개수가 고정되어 있다(1024)

#### 4. I/O Multiplexing - epoll을 이용한 single thread eventloop

- epoll system call이란?
  - linux의 I/O multiplexing system call이다
  - select에서 발전된 형태이며, 준비 된 socket을 첨부해 O(1)으로 탐색이 가능하도록 한다
- 장점
  - 적은 수(단일)의 스레드만으로도 수많은 클라이언트 요청을 효율적으로 관리할 수 있다
- 단점
  - CPU bound작업이 있을 경우 loop 전체가 block된다
  - 다중 코어를 사용할 수 없다

#### 5. I/O Multiplexing - epoll을 이용한 multi thread eventloop

- 해당 설게 방식은 reactor pattern이라는 이름이 붙어있다
- eventloop로 사용되는 worker thread가 여러 개 존재(Runtime.getRuntime().availableProcessors() * 1~2)하며 sync, non-blocking을 사용한다
  - 요청의 결과는 event로 발행되며 동기적으로 eventloop를 통해 처리된다
  - 비동기 요청(network I/O)은 kernel이 작업 후 준비 완료 상태가 반환된다(O(1))
  - blocking 요청이 섞여있다면 eventloop자체가 마비된다
  - coroutine을 이용하면 동기적인 프로그래밍 방식으로 비동기 실행과 같은 효과를 얻을 수 있다
    - e.g. Spring WebFlux에서 flatMap 등을 이용해 복잡하게 체이닝하던 비동기 로직을, Kotlin Coroutine을 도입하면 평범한 동기식 코드로 작성할 수 있게 된다
    ```java
    public Mono<UserOrderDto> getUserOrderDetails(String userId) {
        return userService.getUser(userId) // 1. 유저 가져오기
            .flatMap(user -> orderService.getOrders(user.getId()) // 2. 주문 가져오기 (user 변수 필요)
                .flatMap(order -> detailService.getDetails(order.getId()) // 3. 상세 정보 가져오기
                    .map(detail -> new UserOrderDto(user, order, detail)) // 4. 조립 (user와 order이 다 필요)
            )
        );
    }
    ```
    ```kotlin
    suspend fun getUserOrderDetails(userId: String): UserOrderDto {
        val user = userService.getUser(userId)         // suspend 함수, non-blocking 대기
        val order = orderService.getOrders(user.id)     // suspend 함수, non-blocking 대기
        val detail = detailService.getDetails(order.id) // suspend 함수, non-blocking 대기
    
        return UserOrderDto(user, order, detail)
    }
    ```
# ReentrantLock

- Lock.support를 통해 Lock을 다양하게 구현할 수 있지만, 너무 저수준이라 작성할 코드가 많아진다
- 이를 극복하기 위해 java에서 Lock인터페이스와 그 구현체인 ReentrantLock를 제공한다
  - ReentrantLock의 경우 공정/비공정 모드가 존재한다(순서가 보장/비보장)
  - 비공정도 내부가 큐로되어 있어서 어느정도 순서가 일치하지만, 갑자기 등장한 스레드에게 순서를 뺏기는 경우가 있어 비공정이라고 정의하는 것이다
- Lock 인터페이스를 구현한 다른 여러 구현체가 존재하고, 이를 이용해 다양한 락 구현이 가능하다

## Lock 인터페이스

```java
 package java.util.concurrent.locks;

 public interface Lock {
    // lock을 무한 대기한다
    void lock();
    /**
     * 현재 스레드가 인터럽트된 상태가 아니라면 락을 획득
     * Acquires the lock unless the current thread is interrupted.
     *
     * 락이 비어있으면 즉시 획득
     * Acquires the lock if it is not held by another thread and returns immediately...
     *
     * 다른 스레드가 락을 잡고 있어 대기하는 도중 인터럽트가 발생하면 대기를 중단
     * If the lock is held by another thread then the current thread lies dormant until:
     * - The lock is acquired by the current thread; or
     * - Some other thread interrupts the current thread.
     *
     * 락을 얻으려고 대기하는 동안 인터럽트가 발생하면 예외가 발생하고 인터럽트 상태는 초기화
     * If the current thread is interrupted while acquiring the lock, 
     * then InterruptedException is thrown and the current thread's interrupted status is cleared.
     *
     * 정상적인 락 획득이나 재진입 과정보다 인터럽트에 반응하는 것을 최우선으로 처리
     * Preference is given to responding to the interrupt over normal or reentrant acquisition of the lock.
     *
     * @throws InterruptedException if the current thread is interrupted
     */
    // lock을 잡고있는 중의 취소를 하기 위한 것이 아니라, 락을 잡기위해 대기중(AQS)에서 외부에서 취소(interrupt)할 수 있게 열어두는 메서드이다
    void lockInterruptibly() throws InterruptedException;
    // 시도 후 실패시 false
    boolean tryLock();
    // 지정된 시간 대기 후 실패시 false, interrupt 발생시 error throw
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
 }
```

- Lock 인터페이스는 동시성 프로그래밍에서 쓰이는 안전한 임계 영역을 위한 락을 구현하는데 사용된다
  - 인터페이스는 다음과 같은 메서드를 제공한다. 대표적인 구현체가 ReentrantLock이다
- 이 인터페이스를 사용하면 고수준의 동기화가 구현 가능하다
  - Lock인터페이스는 synchronized 블록보다 더 많은 유연성을 제공하며, 특히 락을 특정 시간만큼 시도하거나 인터럽트 가능한 락을 구현할때 유용하다
  - 인터페이스의 메서드명만 봐도 synchronized의 무한대기 문제도 깔끔하게 극복할 수 있다

## java.util.concurrent.locks.AbstractQueuedSynchronizer (AQS)

- 자바의 고수준 동기화 프레임워크(ReentrantLock, Semaphore, CountDownLatch, ReentrantReadWriteLock 등)의 뼈대가 되는 핵심 엔진
- 내부에 동기화 상태를 나타내는 volatile int state 변수와, 락을 얻지 못한 스레드들이 줄을 서는 FIFO Wait Queue를 가지고 있다
- 락 획득에 실패한 스레드는 자신을 대표하는 Node 객체를 생성해 AQS의 대기 큐(더블 링크드 리스트) 맨 뒤에 등록
- 큐에 등록된 스레드는 앞서 다룬 LockSupport.park()를 통해 spin대기 하지 않고 커널 레벨에서 안전하게 블로킹
- 락을 해제하는 스레드가 발생하면, AQS는 큐의 다음 노드에 있는 스레드를 지목하여 LockSupport.unpark(nextThread)로 깨운다
- 사용 방법
  - AQS를 상속받는 클래스 내부에서 protected로 열려 있는 템플릿 메서드들을 오버라이딩하여 "자원을 어떻게 획득하고 해제할 것인가"의 규칙만 정의
    - tryAcquire(int arg) / tryRelease(int arg): 배타적 모드(1개만 허용) 락 획득/해제 로직 구현
    - tryAcquireShared(int arg) / tryReleaseShared(int arg): 공유 모드(여러 개 허용) 자원 획득/해제 로직 구현
    - isHeldExclusively(): 현재 스레드가 락을 쥐고 있는지 여부 확인

```plaintext
[ AbstractQueuedSynchronizer (AQS) 내부 구조 ]
┌────────────────────────────────────────────────────────┐
│ 🔹 1. state (동기화 상태 필드)                           │
│    └─ 타입: volatile int                               │
│    └─ 의미: 락 획득 여부(0 또는 1+α), 재진입 횟수,          │
│           또는 남은 퍼밋(Permit) 개수를 나타내는 공유 값    │
│ ────────────────────────────────────────────────────── │
│ 🔹 2. FIFO Wait Queue (대기 큐)                        │
│    └─ head와 tail 포인터로 관리되는                      │
│       더블 링크드 리스트 (Doubly-Linked List)           │
└────────────────────────────────────────────────────────┘
                           ▲
                           │  (상태 변경은 CAS, 스레드 제어는 LockSupport 활용)
┌──────────────────────────┴─────────────────────────────┐
│             Node (큐의 각 요소를 구성하는 내부 클래스)   │
│                                                        │
│  - volatile Thread thread;    // 이 노드가 대기시키는 스레드 │
│  - volatile Node prev, next;  // 앞사람, 뒷사람 링크       │
│  - volatile int waitStatus;   // 노드의 상태 (SIGNAL 등)   │
└────────────────────────────────────────────────────────┘
```

```java
// AQS 내부의 핵심 획득(Acquire) 흐름 개념
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&  // 1. 락 획득을 먼저 시도해보고 실패하면,
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) { // 2. 큐에 노드를 등록하고 park 루프 진입
        selfInterrupt();
    }
}

// AQS 내부의 큐 대기 루프
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) { // 각 스레드가 자신의 스택 안에서 도는 개별 루프
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) { // 내 앞사람이 head이고 락을 잡을 수 있다면
                setHead(node); // 락을 잡음
                p.next = null; 
                failed = false;
                return interrupted;
            }
            // 조건이 안 맞으면 park로 잠들고, 깨어난 뒤 인터럽트 여부 체크
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

## java.util.concurrent.locks.LockSupport

- 스레드마다 permit 하나(0 또는 1, 1이면 통행 가능 상태)가 있는 바이너리 세마포어이다
  - 뮤텍스는 ‘소유권(Ownership)’ 개념이 있어 락을 획득한 스레드만 해제할 수 있다
  - 반면 바이너리 세마포어는 소유권 개념이 없다
  - 즉, 락을 쥔 주체나 특정 객체의 소유자가 아니더라도, 대상 스레드의 참조만 알고 있다면 외부의 전혀 다른 스레드나 관리자 스레드가 언제든 LockSupport.unpark(targetThread)를 호출해서 깨워줄 수 있다
- Lock 인터페이스와는 관련이 없다
- java에서 락을 다루는 가장 저수준 API이다
  - park()는 "permit이 1이 아니면 여기서 멈춰라"는 뜻이며, 내가 왜 멈춰 있는지(parkBlocker)를 기록한다(트레이스 용)
  - unpark()는 특정 스레드에게 통행증(1)을 쥐어주며, 멈춰 있다면 깨우고 아니면 지나갈 수 있게 해 준다
- 즉, LockSupport는 결국 각 스레드가 품고 있는 바이너리 상태(0 또는 1)를 조작하고 확인하기 위한 가장 저수준(Low-level)의 API이다
  - permit은 단일 진입과는 연관이 없으며, 1인 스레드라면 모두 락에 접근하는 것을 허용한다

```plaintext
[ JVM 내부 / Java Thread 객체 (Instance) ]
┌────────────────────────────────────────────────────────┐
│ threadName  : "Thread-1"                               │
│ target      : Runnable                                 │
│ priority    : 5                                        │
│ ────────────────────────────────────────────────────── │
│ 🔹 1. parkPermit (내부 퍼밋 상태)                        │
│    └─ 타입: int / boolean (0 또는 1)                     │
│    └─ 의미: 이 스레드만의 독립적인 '통행증' 보유 여부        │
│ ────────────────────────────────────────────────────── │
│ 🔹 2. parkBlocker (블로킹 원인 객체)                     │
│    └─ 타입: Object                                     │
│    └─ 의미: "나 지금 이 객체 때문에 멈춰있어"라고 기록하는 필드 │
└────────────────────────────────────────────────────────┘
                           ▲
                           │  (LockSupport가 이 필드들을 읽고 씀)
┌──────────────────────────┴─────────────────────────────┐
│             LockSupport (정적 유틸리티 클래스)            │
│                                                        │
│  - LockSupport.park(blocker)                           │
│    ↳ 내(현재 스레드)의 parkBlocker에 객체 기록 후,        │
│      parkPermit이 0이면 OS 커널 레벨에서 블로킹           │
│                                                        │
│  - LockSupport.unpark(Thread t)                        │
│    ↳ 대상 스레드(t)의 parkPermit을 1로 강제 설정           │
│                                                        │
│  - LockSupport.getBlocker(Thread t)                    │
│    ↳ 대상 스레드(t)의 parkBlocker 필드 값을 읽어서 반환     │
└────────────────────────────────────────────────────────┘
```

```java
// LockSupport javadoc
// 해당 코드는 spin 대기를 하지 않는다
// LockSupport.park()는 커널 레벨에서의 블로킹을 수행한다
// 해당 루프는 각 스레드가 하나의 경쟁 지점을 진입할때 사용한다
// 멀티스레드 환경에서 락을 얻지 못한 각 스레드는 저마다 자신의 스레드 스택 안에서 이 while 루프를 개별적으로 실행
while (queue.peek() != current || !locked.compareAndSet(false, true)) {
    LockSupport.park(this);
    if (Thread.interrupted()) wasInterrupted = true;   // 인터럽트는 기록만
}
```

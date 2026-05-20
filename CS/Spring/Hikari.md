# Hikari Connection Pool(bitcopark 26_4_25)

> Hikari Connection Pool은 스프링이 채택하고 있는 JDBC전용 커넥션 풀이다

- `HikariConfig` 클래스에 DB Connection을 위한 설정을 저장 후 Connection 관리 클래스인 `HikariDataSource`를 통해 커넥션을 가져와서 connectionPool을 생성, 관리한다
  - `fastPathPool`필드를 통해 connection Pool을 관리하며, 스레드 풀 처럼 생성에 많은 코스트가 드는 커넥션을 생성 후 Pool로 관리한다
  - Postgre에서 추천하는 적정 커넥션 수는 `Runtime.getRuntime().availableProcessors()` *2 + harddisk 수(일반적으로 1)이며, Spring 기본 값은 10이다
- HikariCP는 이름처럼(일본어 빛) 빠른 커넥션이 되는 것을 목표로 많은 최적화가 들어가 있다

## HikariCP의 최적화들

```java

public class ConcurrentBag<T extends IConcurrentBagEntry> implements AutoCloseable {
    // ..

   /**
    * The method will borrow a BagEntry from the bag, blocking for the
    * specified timeout if none are available.
    *
    * @param timeout how long to wait before giving up, in units of unit
    * @param timeUnit a <code>TimeUnit</code> determining how to interpret the timeout parameter
    * @return a borrowed instance from the bag or null if a timeout occurs
    * @throws InterruptedException if interrupted while waiting
    */
   public T borrow(long timeout, final TimeUnit timeUnit) throws InterruptedException
   {
      // ThreadLocal 캐시에서 사용한 커넥션 탐색
      // Try the thread-local list first
      final var list = threadLocalList.get();
      for (var i = list.size() - 1; i >= 0; i--) {
         final var entry = list.remove(i);
         @SuppressWarnings("unchecked")
         final T bagEntry = useWeakThreadLocals ? ((WeakReference<T>) entry).get() : (T) entry;
         if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            return bagEntry;
         }
      }

      // sharedList에서 사용 가능 커넥션 탐색
      // Otherwise, scan the shared list ... then poll the handoff queue
      final var waiting = waiters.incrementAndGet();
      try {
         for (T bagEntry : sharedList) {
            if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
               // 대기자가 더 있다면 busy로 판단하고 추가 커넥션 생성 요청
               // If we may have stolen another waiter's connection, request another bag add.
               if (waiting > 1) {
                  listener.addBagItem(waiting - 1);
               }
               return bagEntry;
            }
         }

         listener.addBagItem(waiting);

         timeout = timeUnit.toNanos(timeout);
         do {
            final var start = currentTime();
            // SynchronousQueue<T> handoffQueue 에서 즉시 요청
            // 락의 경합을 피하고 cpu cache, context switching을 위하여
            final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
            if (bagEntry == null || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
               return bagEntry;
            }

            timeout -= elapsedNanos(start);
         } while (timeout > 10_000);

         return null;
      }
      finally {
         waiters.decrementAndGet();
      }
   }
```

### ThreadLocal을 이용한 Lock-Free 방식의 최적화

- `HikariPool`의 필드인 `ConcurrentBag<PoolEntry> connectionBag`필드를 이용한다
  - PoolEntry: 실제 물리적인 JDBC Connection과 그 상태(사용 중, 대기 중 등)를 담고 있는 래퍼 객체이다
  - ConcurrencyBag: HikariCP의 전용 저장소이며, Lock-free 기법을 활용하여 여러 스레드가 동시에 커넥션을 요청할 때 경합을 최소화하도록 설계된 고성능 컬렉션
- `ConcurrencyBag`클래스의 `ThreadLocal<List<Object>> threadLocalList`를 사용해 이전에 사용된 스레드를 캐싱한다
  - 이 리스트를 역으로 순회하면서 기록된 최신 순으로 사용했던 Connection의 상태를 조사해 사용 가능한 Connection이 있으면 바로 사용한다
  - `connectionBag`에서 사용 가능한 Connection을 찾을 수 없다면 `CopyOnWriteArrayList<T> sharedList`를 사용해 `threadLocalList`필드에서 사용가능한 Connection이 없을 경우 현재 사용가능한 Connection을 찾는다
  - 사용한 Connection의 참조를 ThreadLocal에 넣어놓는 방식으로 Connection선택시 가장 많은 비용을 치루는 Lock의 경합을 회피한다


### ConcurrentBag의 Hand-off

- threadLocalList와 sharedList에서도 사용 가능한 커넥션을 찾지 못했을 때의 전략
- `handOffQueue` (SynchronousQueue): 다른 스레드가 커넥션을 반납하기를 기다리는 지점
- 단순 대기가 아니라, 반납하는 스레드가 대기 중인 스레드에게 직접 커넥션을 넘겨주는(Hand-off) 방식을 취해 불필요한 컨텍스트 스위칭을 줄인다

### 클래스의 필드 값 복사

- 클래스의 필드값을 복사해 JVM의 최적화를 돕는다
  - 필드의 값을 사용할 Compiler는 인라인화 결정을 하기 힘들다
  - 필드의 값을 final로 복사해 사용할 경우 Compiler는 값의 불변의 인식이 쉬워져 인라인화를 시도한다
    - 해당 메서드 스택 프레임 안에서만 존재하며, 값이 고정됨이 확실해진다. 즉 컴파일러는 이 값을 CPU 레지스터에 직접 올려두고 재사용할 수 있으며, 값이 변하지 않으니 이 값을 사용하는 메서드 호출 등을 인라인화할 수 있다
  - 알면 쓸모가 많은 최적화 방식 중 하나이다

### FastList

- ArrayList의 오버헤드 제거
- HikariCP는 자바 표준 ArrayList를 쓰지 않고 자체 구현한 FastList를 사용
  - 인덱스 체크 생략: ArrayList는 get(int index) 호출 시 매번 범위 체크를 하지만, FastList는 이를 생략
  - 삭제 최적화: JDBC의 특성상 Statement나 ResultSet은 최근에 열린 순서대로 닫히는 경우가 많아 FastList는 요소를 삭제할 때 끝에서부터 스캔하여 삭제

### Statement Caching

- 많은 커넥션 풀이 자체적으로 PreparedStatement 캐싱 기능을 제공하지만, HikariCP는 이를 생략
  - 이유: 현대의 JDBC Driver(PostgreSQL, MySQL 등)가 이미 드라이버 레벨에서 더 효율적인 캐싱을 제공하기 때문
  - JAVA 계층에서의 중복 캐싱을 제거해 메모리 점유율을 낮추고 코드 복잡도를 줄였다

### invokeVirtual 인라인화를 이용한 최적화

```java
// 수정 전
public final PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException
{
    return PROXY_FACTORY.getProxyPreparedStatement(this, delegate.prepareStatement(sql, columnNames));
}
// 수정 후
public final PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException
{
   return ProxyFactory.getProxyPreparedStatement(this, delegate.prepareStatement(sql, columnNames));
}
```

- invokeVirutal은 vtable을 탐색해야 하므로 추가 비용이 든다
  - 구현체를 런타임에 특정해 invokeStatic으로 호출할 수 있다면 JIT이 인라인화를 시도해 최적화 할 수 있다
  - 구현체가 많으면(3개 이상) JIT은 최적화를 포기하는 경우가 많다
  - 이를 극복하기 위해 런타임 바이트 코드 생성 라이브러리인 Javassist를 활용해 byte code를 삽입해 인스턴스(필드) 거치지 않고 정적 메서드가 정의된 final 클래스를 직접 지칭하여 invokestatic 호출을 유도한다
  - 즉, 여러 구현체가 있는 팩토리 메서드(MySQL, Oracle의 delegate) 대신 final 클래스의 static메서드를 이용한 직접 호출로 최적화 한 것이다(한개의 구현체) 
  - 즉, 동적 프록시 클래스(팩토리)의 내부 메서드는 벤더의 메서드를 바이트코드로 채워넣는다(버전이 바뀌며 구현이 바뀌어도 코드 변경이 필요 없기 때문) / 수정 후에는 ProxyFactory가 final class이고 static method이기에 JVM이 확신할 수 있어 invokeStatic이 호출된다

### 권장 설정

> https://github.com/brettwooldridge/HikariCP/wiki/MySQL-Configuration

```plaintext
jdbcUrl=jdbc:mysql://localhost:3306/simpsons
username=test
password=test
dataSource.cachePrepStmts=true
dataSource.prepStmtCacheSize=250
dataSource.prepStmtCacheSqlLimit=2048
dataSource.useServerPrepStmts=true
dataSource.useLocalSessionState=true
dataSource.rewriteBatchedStatements=true
dataSource.cacheResultSetMetadata=true
dataSource.cacheServerConfiguration=true
dataSource.elideSetAutoCommits=true
dataSource.maintainTimeStats=false
```
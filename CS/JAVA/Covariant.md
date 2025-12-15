# 공변성 반공변성 실제 예시


```java
public interface CacheManager<E> {
    boolean isTTL();
    Collection<E> getCache();
    void updateCache(Collection<E> newCache);
}

@Component
public final class EngravingCacheManager implements CacheManager<Item> {

    @Override
    public List<Item> getCache() {
        return this.relicEngravingsCache;
    }

    @Override
    public void updateCache(Collection<Item> items) {
        this.lastUpdatedTime = LocalDateTime.now();
        relicEngravingsCache.clear();
        relicEngravingsCache.addAll(items);
    }
}
```

- 위 예제의 구현, updateCache에서 `List<Item>`을 파라미터로 받는 것은 불가능하다(반공변성)
  - List를 이용해 인터페이스를 사용한다면 List에서만 구현된 메서드만 사용하면 컴파일이 불가능하기 때문이다
- 하지만 `getCache()`의 반환값은 List를 사용할 수 있다
  - 반환시키는 것은 구체적인 구현체여도 아무런 차이가 없기 때문이다(공변성)
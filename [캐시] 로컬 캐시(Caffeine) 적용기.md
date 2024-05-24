# 로컬 캐시 Caffeine 적용기

## 1. 적용 이유
저는 Pickple 프로젝트에서 지도를 담당하고 있습니다. 지도는 사실 특정 장소에 대한 위도, 경도 값만 클라이언트에게 넘겨줘도 간단한 기능은 구현 가능한데요. 좀 더 심화해서 적용해보고 싶어 서울시의 모든 구 Polygon 데이터를 구해서 데이터베이스에 넣어둔 상태입니다. 지도에서의 Polygon은 어떠한 특정 구역을 의미하고, 그 특정 구역은 수 많은 좌표로 이루어져있습니다. 여러개의 좌표가 모여 지도 특정 구역의 테두리가 되는 것이죠.

![Untitled](/image/local_cache/caffeine_cache.png)

저희 프로젝트에서도 위 사진과 같이 서울에서만 진행하는 서비스인데요. 만약 클라이언트가 위치 정보 수집을 거부할 경우 클라이언트가 로그인 했을 때 가지고 있는 주 활동지역(서울시 영등포구)을 이용해서 Polygon(서울시의 특정 구)안에 있는 데이터를 모두 불러올 수 있게 기능을 구현했습니다.

하지만 앞서 말씀드렸듯 Polygon 데이터는 수 많은 좌표가 모여 하나의 Polygon을 이루고 있고 서울시 하나의 구는 수천개의 Polygon으로 이루어져있어 데이터가 굉장히 방대합니다. 클라이언트가 요청할 때마다 Polygon의 그 수 많은 데이터를 불러와야하는 것이죠. 

서울시 특정 구 Polygon 데이터는 실제로 거의 변하지 않는 값이기 때문에 똑같은 데이터를 계속 READ하는 작업이고, 변하지 않는 데이터에다가 좀 무거운 데이터라면 캐시에 넣어서 사용하면 좋을 것 같다고 생각이 들었습니다.

### 1.1 왜 로컬 캐시인가요?

현재 사용하는 서버가 여러 대여서 레디스를 고려해보기도 했지만 모든 서버가 공유하면서 실시간으로 변경된 것을 가져다가 사용해야하는 그런 데이터도 아니었고 네트워크를 타지 않는 로컬 캐시이다보니 오히려 더 빠른 장점이 있었습니다. 또 굳이 엄청나게 중요하지 않은 데이터인데 Redis를 서버로 올려서 진행하는 것이 오버 스펙이라고 느껴졌습니다.

### 1.2 다양한 로컬 캐시 중 왜 Caffeine 인가요?

[Benchmarks](https://github.com/ben-manes/caffeine/wiki/Benchmarks)

물론 로컬 캐시도 각각의 캐시마다 장, 단점이 있지만 위 글에서 보기를 READ 100%일 때 타 로컬 캐시들보다 훨씬 더 좋은 성능을 보이고 있습니다. 저희 프로젝트도 READ 100%이기에 최대한 좋은 성능을 이끌어 내고 싶어서 선택하게 되었습니다.

## 2. 적용 해보기

### 2.1 build.gradle에 추가

```sql

implementation 'org.springframework.boot:spring-boot-starter-cache'
implementation "com.github.ben-manes.caffeine:caffeine:3.1.8"
```

### 2.2 캐시 활성화 및 캐시 관리 설정

**캐시 활성화**

```java
@Configuration
**@EnableCaching** 
public class CacheConfig {

}
```

- `@EnableCaching`을 적어서 캐시를 활성화 합니다. → Spring을 Run하는 곳에 적어도 되나 테스트 코드를 위해 별도로 분리
- Spring Cache 기능을 활성화 하는 어노테이션입니다.
- 해당 어노테이션은 캐싱을 활성화만 하는 것이지 캐시 저장소에 대한 지정을 별도로 하지 않습니다.

**캐시 정보 분리**

```java
@Getter
@RequiredArgsConstructor
public enum CacheType {

    POLYGON("polygon", 60 * 60 * 24 * 7, 1);

    private final String cacheName;
    private final int secsToExpireAfterWrite;
    private final int entryMaxSize;
}
```

- 하나의 캐시만 적용하는 것이 아닌 앞으로 개발하며 많은 데이터들을 캐싱할 수 있으므로 Enum으로 관리하는 것이 좋습니다.

**캐시 관리 설정**

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        final List<CaffeineCache> caches = Arrays.stream(CacheType.values())
                .map(cacheType ->
                        new CaffeineCache(
                                cacheType.getCacheName(),
                                Caffeine.newBuilder()
                                        .recordStats()
                                        .expireAfterWrite(cacheType.getSecsToExpireAfterWrite(), TimeUnit.SECONDS)
                                        .maximumSize(cacheType.getEntryMaxSize())
                                        .build()))
                .toList();

        final SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(caches);
        
        return cacheManager;
    }
}
```

## 3. 주의 해야할 점
### 로컬 캐시의 Evict 시점
로컬 캐시를 사용할 경우 운영되는 서버의 메모리에 올라가기 때문에 너무 오랫동안 적재되어있거나 너무 많은 양이 적재되면 메모리가 꽉차 예외가 발생할 수 있습니다. 캐시를 사용할 때는 항상 Evict에 대해 고려해보아야합니다.

저희 프로젝트에서 사용되는 Polygon 기능은 조회가 많지도 적지도 않은 딱 보통 정도로 예상할 수 있을 것 같습니다. 사용자가 위치 정보에 대한 동의가 없을 경우에만 적용되는 기능이기에 어떻게 보면 보통보다 적을 수도 있겠습니다. 그래서 저는 TTL로 7일을 설정했습니다. 첫번째 조회 후 7일 이후 조회가 없다면 자동으로 캐시에서 사라지는 것이죠.

그치만 만약에, 이 Polygon 데이터 국가의 행정구역이 변경되어 수동으로 수정해야한다면 어떤식으로 해결해야할까요? 현재 해당 기능에 대해 구현된 것은 아니지만 아래와 같이 처리해볼 수 있을 것 같습니다.

**첫번째, DB 트리거를 이용**

트리거는 DB에서 삽입, 삭제, 수정 등의 데이터 변경이 일어나면 발생하는 이벤트입니다.

트리거 이벤트 테이블을 별도로 만들고, 해당하는 테이블에 대해서 SpringBoot가 계속 Polling하는 방식으로 구현해볼 수 있을 것 같습니다. 스케줄링은 하루로 두고, 매 자정마다 변경된 데이터가 있는지 확인해서 처리해볼 수 있을 것 같습니다.

**두번째, Debezium과 Kafka를 이용**

Debezium은 각 테이블 안의 row에 변화가 발생했을 때 이를 캡쳐하여 애플리케이션이 변경된 사항에 대해 대응할 수 있도록 로그 기반의 동기화를 제공합니다.

MySQL에서 Debezium과 관련된 플러그인을 설치하고 Kafka를 통해 Connect하여 사용할 수 있습니다. 실시간으로 대응할 수 있어 좋지만, 저희 프로젝트에서 이용하기위해 Kafka와 Debezium을 이용하는 것은 오버스펙이라고 볼 수 있을 것 같습니다.

최종적으로 선택한다면 저는 DB 트리거를 이용해서 처리해볼 수 있을 것 같습니다. 아무래도 폴리곤 데이터는 변경되는 주기가 6개월이기도하고, 수동으로 빠르게 변경해서 처리해야하는 기능도 아니기 때문에 해당 기능에 대한 보완을 한다면 DB 트리거를 이용할 것 같습니다.

다만 나중에 실시간으로 DB에 대한 동기화가 필요한 것이 있는데 중요하다면 Debezium도 이용해볼 수 있을 것 같습니다.
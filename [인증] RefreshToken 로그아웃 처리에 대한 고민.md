### 고민
현재 제가 진행하고 있는 프로젝트에서는 RefreshToken을 Database에 저장하고 로그아웃시 테이블에서 레코드를 삭제하는 로직으로 구현되어있습니다. 클라이언트에서 로그아웃 버튼을 클릭했을 때 `/auth/logout` API를 요청하여 RefreshToken을 DB에서 삭제하는 방식인거죠. 
(이때 로그인 된 사용자만 로그아웃을 할 수 있기 때문에 AccessToken을 통해 API 인증을 수행합니다.)

> 근데 만약 클라이언트에서 로그아웃 버튼을 누르는 것이 아닌 RefreshToken이 만료되었다면 어떻게 처리하지?
>

클라이언트에서 RefreshToken이 만료된 경우 Access Token으로 로그아웃 API를 요청할 수 없게 됩니다. (토큰이 만료되었기 때문에)
그럼 RefreshToken은 데이터베이스에 영원히 쌓이는 문제가 생깁니다.

이 문제를 어떻게 하면 해결 할 수 있을까요? 🤔🤔 

### 해결 방법
클라이언트에서 RefreshToken이 만료된 경우 우선 로그아웃에 API를 요청할 수 없다고 생각하니 딱 2가지가 생각났습니다. `Redis의 TTL` 기능을 이용하여 로그인시에 RefreshToken의 만료시간과 동일하게 넣어주기, 그리고 `Spring Batch`를 이용하여 특정 시간마다 데이터베이스에 있는 RefreshToken을 불러와 만료된 토큰들은 삭제 처리하는 것이죠.

저는 이중에서 Redis의 TTL 기능을 이용했습니다. 우선 Spring Batch를 돌린다고 가정했을 때 분명 테이블에는 수 많은 RefreshToken이 있을텐데 이 모든 토큰들을 매번 불러와서 JWT 검증을 하고 삭제 작업을 한다는 것은 많은 I/O를 불러일으킬 수 있다고 생각했습니다.

**CacheConfig.java**
```java
@Configuration
@EnableCaching
@RequiredArgsConstructor
public class CacheConfig {

    private final RedisProperties redisProperties;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        final RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName(redisProperties.getHost());
        redisStandaloneConfiguration.setPort(redisProperties.getPort());

        final String password = redisProperties.getPassword();
        if (password != null) { // local, dev 환경에서는 배포된 Redis가 비밀번호가 걸려있고 prod 환경에서는 비밀번호가 걸려있지 않은 AWS ElasticCache를 사용
            redisStandaloneConfiguration.setPassword(password);
        }

        return new LettuceConnectionFactory(redisStandaloneConfiguration);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        // ObjectMapper에 JavaTimeModule을 담아 LocalDateTime을 직렬화, 역직렬화 하지 못하는 현상을 해결
        final PolymorphicTypeValidator typeValidator = BasicPolymorphicTypeValidator.builder()
                .allowIfBaseType(Object.class)
                .build();

        final ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule());
        objectMapper.activateDefaultTyping(typeValidator, ObjectMapper.DefaultTyping.NON_FINAL);

        final GenericJackson2JsonRedisSerializer genericJackson2JsonRedisSerializer = new GenericJackson2JsonRedisSerializer(
                objectMapper);

        final RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(genericJackson2JsonRedisSerializer);

        return redisTemplate;
    }
}
```

우선 Redis를 사용하기 위해 Config 설정을 지정합니다. RedisTemplate을 이용할 때 Key는 String, Value 값은 Object 형태로 했고 GenericJackson2를 이용할 때는 패키지 명까지 저장되어서 MSA 환경에서는 문제가 생길 수 있지만 현재 모놀리식 형태로 하고 있기 때문에 전혀 지장이 없을 것 같아서 사용했습니다. 추가로 Redis에 LocalDateTime을 저장할 때 역직렬화, 직렬화로 생기는 문제를 해결하기위해 ObjectMapper를 이용하였습니다.

**RefreshToken.java**
``` java
@Getter
@Builder
@AllArgsConstructor(access = AccessLevel.PRIVATE)
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class RefreshToken {

    private String token;
    private Long memberId;
    private LocalDateTime createdAt;
}
```
Redis에 담길 RefreshToken 객체입니다. token 그리고 해당 토큰을 가지고 있는 memberId, 생성된 시각을 담고 있습니다.

**RedisRepository.java**
``` java

@Repository
public class RedisRepository {

    private final RedisTemplate<String, Object> redisTemplate;
    private final HashOperations<String, String, Object> hashOperations;

    public RedisRepository(final RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
        this.hashOperations = redisTemplate.opsForHash();
    }

    public <T> void saveHash(final String key, final String field, final T value, final Long duration) {
        hashOperations.put(key, field, value);
        redisTemplate.expire(key, duration, TimeUnit.SECONDS);
    }

    public <T> T findHash(final String key, final String field) {
        return (T)hashOperations.get(key, field);
    }

    public Boolean existsHash(final String key, final String field) {
        return hashOperations.hasKey(key, field);
    }

    public void deleteHash(final String key, final String field) {
        hashOperations.delete(key, field);
    }
}
```

그리고 마지막은 RedisRepository를 만들어 Hash와 관련된 메소드들을 만들고 해당 메소드들을 이용하여 저장하고 삭제할 수 있도록 공통화 시켜두었습니다.

### 동작 방식
1. 로그인이 되면 RefreshToken을 Redis에 HashMap형태로 저장한다.("refresh-token", refreshToken.getToken(), refreshToken)
2. 로그아웃 API를 요청받으면 클라이언트의 RefreshToken으로 키를 조회해서 삭제시킨다.
3. 클라이언트에서 RefreshToken이 만료된 경우에는 Redis에서도 TTL 설정으로 인해 같이 만료되어 자동으로 사라진다.

### 결론
DB에서 처리하게 되면 불필요한 I/O 접근이 생기고 클라이언트에서 RefreshToken이 만료되면 DB에서 삭제할 방법이 없기 때문에 Redis의 TTL 기능을 이용하여 처리하면 좋습니다. 로컬 캐시를 이용해도 TTL 기능을 이용하여 처리할 수 있지만 현재 제가 진행하고 있는 프로젝트는 prod 서버가 2대이므로 Redis 서버를 따로 구현하여 모든 서버에서 Redis 서버에 있는 데이터를 바라볼 수 있도록 했습니다.
### 문제 상황
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Game extends BaseEntity {
    
   // 생략
   
    @NotNull
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "host_id")
    private Member host;

    @NotNull
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "address_depth1_id")
    private AddressDepth1 addressDepth1;

    @NotNull
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "address_depth2_id")
    private AddressDepth2 addressDepth2;

    @Embedded // @OneToMany
    private GamePositions gamePositions = new GamePositions();

    @Embedded // @OneToMany
    private GameMembers gameMembers = new GameMembers();

   // 생략

}
```

위 `Game` Entity에서 Lazy로 되어있는 필드로 인하여 **N+1 문제**가 발생하기 때문에, 해당 문제를 해결하기 위해 fetch 조인을 아래와 같이 설정하여 조회했습니다.

```java
    public List<Game> findGamesWithInDistance(
            final Double latitude,
            final Double longitude,
            final Double distance
    ) {
        final String pointWKT = String.format("POINT(%s %s)", latitude, longitude);

        return jpaQueryFactory
                .selectFrom(game)
                .where(isWithInDistance(pointWKT, distance))
                .orderBy(getOrderByDistance(pointWKT))
                .join(game.host).fetchJoin()  // @ManyToOne
                .join(game.addressDepth1).fetchJoin() // @ManyToOne
                .join(game.addressDepth2).fetchJoin() // @ManyToOne
                .leftJoin(game.gamePositions.gamePositions).fetchJoin() // @OneToMany
                .leftJoin(game.gameMembers.gameMembers).fetchJoin() // @OneToMany
                .fetch();
    }
```

### 문제 발생
위 메서드가 실행될 때 아래와 같이 MultipleBagFetchException이 발생하였습니다.

```
Caused by: org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags
```

### 문제 원인
**QueryDSL**
```java
    public List<Game> findGamesWithInDistance(){
                // 생략
                .leftJoin(game.gamePositions.gamePositions).fetchJoin() // @OneToMany
                .leftJoin(game.gameMembers.gameMembers).fetchJoin() // @OneToMany
                .fetch();
                // 생략
    }
```

**Game**
```java
    @Embedded // @OneToMany
    private GamePositions gamePositions = new GamePositions();

    @Embedded // @OneToMany
    private GameMembers gameMembers = new GameMembers();
```
- `@OneToMany` 즉, Hibernate PersistentBag으로 이루어진 필드를 2개 이상 Join 하려고 할 때 발생합니다.

<br>


> **Hibernate PersistenceBag**  [Java Docs]
> 
> An unordered, un-keyed collection that can contain the same element multiple times. The Java Collections Framework, curiously, has no Bag interface. It is, however, common to use Lists to represent a collection with bag semantics, so Hibernate follows this practice.
>
> **동일한 요소를 여러 번 포함할 수 있는 순서 없는 키 없는 컬렉션**입니다. 흥미롭게도 Java Collections Framework에는 Bag 인터페이스가 없습니다. 그러나 Lists를 사용하여 Bag 시맨틱스가 있는 컬렉션을 표현하는 것은 일반적이므로 Hibernate는 이 방법을 따릅니다.

우선 위 에러에 나오는 `bags`이라는 단어는 무엇을 의미하는 것일까요?

그 의미를 알기 위해서는 Hibernate의 `PersistentBag`이라는 개념을 알아야합니다. 
저희가 JPA를 사용할 때 Hibernate 구현체를 이용하게 되는데 이 Hibernate는 Entity 필드에 포함된 Java 컬렉션이 영속상태가 되면 Hibernate가 제공하는 컬렉션으로 바뀌게 됩니다. Hibernate는 컬렉션을 효율적으로 관리하기 위해 엔티티를 영속 상태로 만들 때 원본 컬렉션을 감싸고 있는 컬렉션을 만들고 사용하는 것이죠.

그래서 List로 되어있는 GamePostions(Embedded)는 영속 상태가 되면 `PersistenceBag`으로 변경됩니다.   

> 그래서 **PersistentBag**으로 되어있는데 Join하는게 왜 문제인데?
>

우선 일대다 관계에서는 Join문을 이용하면 중복된 결과 값을 가져올 수 있습니다.

예를 들어, User와 Post가 있다고 가정해봅시다. 1명의 User는 N개의 Post를 생성할 수 있습니다.
그럼 User를 가져올 때 Post를 같이 가져와볼까요?
```SQL
select u from User u join fetch u.posts
```
만약 User가 1명이고 Post가 100개라면 당연히 불러오는 User 데이터는 1개, Post 데이터는 100개가 되어야할 것입니다. 하지만 실제로 이렇게 조회하는 경우 Join을 했기 때문에 실제 1번 User의 중복된 값이 100개 Post 데이터는 100개로 나오는 것이죠.

즉, PersistentBag으로 이루어져있는 `@OneToMany`의 필드를 여러 개 불러오게 되면 불필요한 많은 양의 데이터를 불러올 수 있기 때문에 Hibernate 자체에서 2개 이상의 `@OneToMany` 관계를 갖는 Entity를 한 번에 불러오려고 시도할 때 해당 예외가 발생하게 됩니다.


### 해결 방법
```java

@Repository
@RequiredArgsConstructor
public class GameSearchRepository {

    private final JPAQueryFactory jpaQueryFactory;

    public List<Game> findGamesWithInDistance(
            final Double latitude,
            final Double longitude,
            final Double distance
    ) {
        final String pointWKT = String.format("POINT(%s %s)", latitude, longitude);

        return jpaQueryFactory
                .selectFrom(game)
                .join(game.host).fetchJoin()
                .join(game.addressDepth1).fetchJoin()
                .join(game.addressDepth2).fetchJoin()
                .leftJoin(game.gameMembers.gameMembers).fetchJoin() // 1개만 조인
                .where(isWithInDistance(pointWKT, distance))
                .orderBy(getOrderByDistance(pointWKT))
                .fetch();
    }

    ... 생략
}
```
해결 방법에는 여러가지가 있지만 저는 join을 2개의 `@OneToMany`중 하나의 `@OneToMany`를 걸어 예외를 방지하고, 나머지 필드는 BatchSize를 통하여 성능 이슈를 해결했습니다.

이때 fetch join 할 `@OneToMany`를 고르는 기준은 더 많은 데이터가 나올 것으로 예상되는 곳에 fetch join을 걸어서 해결했습니다.

```java
@Embeddable
public class GamePositions {

    @BatchSize(size = 1000)
    @OneToMany(mappedBy = "game", cascade = {CascadeType.PERSIST, CascadeType.REMOVE}, orphanRemoval = true)
    private List<GamePosition> gamePositions = new ArrayList<>();

    public List<Position> getPositions() {
        return gamePositions.stream()
                .map(GamePosition::getPosition)
                .toList();
    }

    ... 생략

}
```
그리고 나머지 `@OneToMany`의 필드인 GamePositions는 `@BatchSize` 를 해당 엔티티에 걸어 Lazy로 로딩해올 때 `in`을 이용해서 조회할 수 있도록 처리했습니다.(`@BatchSize`를 사용하지 않을 경우 만약 GamePosition의 데이터가 100개라면 쿼리가 100번 나가게 됩니다.)









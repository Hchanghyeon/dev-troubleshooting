# dev-troubleshooting
개발하며 마주쳤던 크고 작은 문제, 고민, 성능 개선 등을 적어놓은 레포입니다.

## 트러블 슈팅
> Exception
> 
[[예외] Fetch 조인시 발생하는 MultipleBagFetchException 트러블 슈팅](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5B%EC%98%88%EC%99%B8%5D%20Fetch%20%EC%A1%B0%EC%9D%B8%EC%8B%9C%20%EB%B0%9C%EC%83%9D%ED%95%98%EB%8A%94%20MultipleBagFetchException%20%ED%8A%B8%EB%9F%AC%EB%B8%94%20%EC%8A%88%ED%8C%85.md)

> Cache
>
[[캐시] 로컬 캐시를 저장하지 못하는 현상 트러블 슈팅](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5B%EC%BA%90%EC%8B%9C%5D%20%EB%A1%9C%EC%BB%AC%20%EC%BA%90%EC%8B%9C%EB%A5%BC%20%EC%A0%80%EC%9E%A5%ED%95%98%EC%A7%80%20%EB%AA%BB%ED%95%98%EB%8A%94%20%ED%98%84%EC%83%81%20%ED%8A%B8%EB%9F%AC%EB%B8%94%20%EC%8A%88%ED%8C%85.md)

> Proxy
> 
[[프록시] Self Invocation 트러블 슈팅](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5B%ED%94%84%EB%A1%9D%EC%8B%9C%5D%20Self%20Invocation%20%ED%8A%B8%EB%9F%AC%EB%B8%94%20%EC%8A%88%ED%8C%85.md)

> Test
>
[[테스트] 테스트 코드에서의 JPA 변경감지 동작](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5B%ED%85%8C%EC%8A%A4%ED%8A%B8%5D%20%ED%85%8C%EC%8A%A4%ED%8A%B8%20%EC%BD%94%EB%93%9C%EC%97%90%EC%84%9C%EC%9D%98%20JPA%20%EB%B3%80%EA%B2%BD%EA%B0%90%EC%A7%80%20%EB%8F%99%EC%9E%91.md)

> Concurrency
>
[[동시성] 공연 예매시 발생하는 좌석 동시성 문제 해결하기](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5B%EB%8F%99%EC%8B%9C%EC%84%B1%5D%20%EA%B3%B5%EC%97%B0%20%EC%98%88%EB%A7%A4%EC%8B%9C%20%EB%B0%9C%EC%83%9D%ED%95%98%EB%8A%94%20%EC%A2%8C%EC%84%9D%20%EB%8F%99%EC%8B%9C%EC%84%B1%20%EB%AC%B8%EC%A0%9C%20%ED%95%B4%EA%B2%B0.md)

> MessageConverter
>
[[MessageConverter] 다른 서버로 API 요청시 응답 값을 정상적으로 불러오지 못하는 문제](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5BMessageConverter%5D%20%EB%8B%A4%EB%A5%B8%20%EC%84%9C%EB%B2%84%EB%A1%9C%20API%20%EC%9A%94%EC%B2%AD%EC%8B%9C%20%EC%9D%91%EB%8B%B5%20%EA%B0%92%EC%9D%84%20%EC%A0%95%EC%83%81%EC%A0%81%EC%9C%BC%EB%A1%9C%20%EB%B6%88%EB%9F%AC%EC%98%A4%EC%A7%80%20%EB%AA%BB%ED%95%98%EB%8A%94%20%EB%AC%B8%EC%A0%9C(feat.%20camelcase).md)

## 고민
> Authentication
>

[[인증] RefreshToken 로그아웃 처리에 대한 고민](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5B%EC%9D%B8%EC%A6%9D%5D%20RefreshToken%20%EB%A1%9C%EA%B7%B8%EC%95%84%EC%9B%83%20%EC%B2%98%EB%A6%AC%EC%97%90%20%EB%8C%80%ED%95%9C%20%EA%B3%A0%EB%AF%BC.md)

[[OAuth] 소셜 최초 로그인시 회원가입 추가 정보를 받는 로직에 대한 고민](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5BOAuth%5D%20%EC%86%8C%EC%85%9C%20%EC%B5%9C%EC%B4%88%20%EB%A1%9C%EA%B7%B8%EC%9D%B8%EC%8B%9C%20%ED%9A%8C%EC%9B%90%EA%B0%80%EC%9E%85%20%EC%B6%94%EA%B0%80%20%EC%A0%95%EB%B3%B4%EB%A5%BC%20%EB%B0%9B%EB%8A%94%20%EB%A1%9C%EC%A7%81%EC%97%90%20%EB%8C%80%ED%95%9C%20%EA%B3%A0%EB%AF%BC.md)

> JPA
>
 [[JPA] 엔티티 간 연관 관계에 대한 고민](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5BJPA%5D%20%EC%97%94%ED%8B%B0%ED%8B%B0%20%EA%B0%84%20%EC%97%B0%EA%B4%80%20%EA%B4%80%EA%B3%84%EC%97%90%20%EB%8C%80%ED%95%9C%20%EA%B3%A0%EB%AF%BC.md)

## 성능 개선, 코드 개선 
[[Spatial Index, Fetch Join, Batch Size] 중심 좌표로 부터 특정 거리의 데이터 불러오기](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5BSpatial%20Index%2C%20Fetch%20Join%2C%20Batch%20Size%5D%20%EC%A4%91%EC%8B%AC%20%EC%A2%8C%ED%91%9C%EB%A1%9C%20%EB%B6%80%ED%84%B0%20%ED%8A%B9%EC%A0%95%20%EA%B1%B0%EB%A6%AC%EC%9D%98%20%EB%8D%B0%EC%9D%B4%ED%84%B0%20%EB%B6%88%EB%9F%AC%EC%98%A4%EA%B8%B0.md)

[[AOP] 공통된 검증 로직 AOP로 개선하기](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5BAOP%5D%20%EA%B3%B5%ED%86%B5%EB%90%9C%20%EA%B2%80%EC%A6%9D%20%EB%A1%9C%EC%A7%81%20AOP%EB%A1%9C%20%EA%B0%9C%EC%84%A0%ED%95%98%EA%B8%B0.md)

[[캐시] 로컬 캐시(Caffeine) 적용기](https://github.com/Hchanghyeon/dev-troubleshooting/blob/main/%5B%EC%BA%90%EC%8B%9C%5D%20%EB%A1%9C%EC%BB%AC%20%EC%BA%90%EC%8B%9C(Caffeine)%20%EC%A0%81%EC%9A%A9%EA%B8%B0.md)

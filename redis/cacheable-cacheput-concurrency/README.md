어느날 운영 중인 서비스에서 redis 캐싱 데이터와 DB 데이터 간의 정합성이 맞지 않은 문제가 발생했습니다.
분명 db의 상태가 변경되면 redis 캐싱도 업데이트를 해주었기 때문에 캐싱 데이터와 db 상태가 일치하지 않는 문제는 이해할 수 없었습니다. 

보통 redis는 다음과 같은 용도로 이용됩니다. 
- 클라이언트에게 전달되는 값이 동일할 때
- 빈번하게 호출될 때 
- 한 번 처리할 때 많은 서버 리소스를 요구 할 때

저 또한 redis를 db콜을 줄이기 위해 많이 사용했습니다. 예를 들면, 유저의 상태 정보를 가져오는 Check API가 있고, 다른 API에서는 유저의 상태 정보를 가져오기 위해 Check API를 통해 유저의 상태 정보를 조회합니다. 
Check API를 호출할 때마다 DB콜을 한다면 트래픽이 증가하면 Database I/O가 증가하고 DB에 큰 부하가 발생할 수 있습니다. 그래서 Check API는 한번 DB에서 조회한 데이터를 redis에 캐싱합니다.  

기능 구현을 위해 사용했던 의존성은 ```spring-boot-starter```에서 제공하는 
```@Cacheable```과 ```@CachePut``` 어노테이션을 이용했습니다.

### @Cacheable
먼저 사용 방법에 대해 말씀 드리면 ```@Cacheable```은 키값으로 캐싱되어 있지 않으면 데이터를 캐시에 저장합니다. 하지만 키 값으로 캐싱되어 있다면 메서드 로직을 실행하지 않고 캐시를 반환합니다. 
코드로 보여드리면 아래와 같습니다. 
```java
// 캐시 저장 (Key를 지정한 경우)
@Cacheable(value = "memberCacheStore", key = "#member.name")
public Member cacheableByKey(Member member) {
    System.out.println("cacheable 실행");
    ...
    return member;
}
```
1. 만약 "dave" name을 가진 Member가 파라미터로 들어오면 처음에는 캐싱 전이므로 "cacheable 실행" 이라는 문자열이 출력됩니다.
2. 그리고 "dave" name을 가진 Member가 redis에 캐싱됩니다. 
3. 이후 "dave" name을 가진 Member가 다시 메서드를 호출하면 "cacheable 실행" 이라는 문자열은 출력하지 않고 캐싱 되어있는 "dave" name을 가진 Member 객체를 반환합니다. 

### @CachePut
```@CachePut```은 ```@CachePut```과 비슷하지만 캐싱되어 있는 내용을 사용하지 않고 항상 저장만 한다는 것이 다릅니다. 
```java
@CachePut(value = "memberCacheStore", key = "#member.name")
public Member cachePut(Member member) {
    System.out.println("cachePut 실행");
    ...
    return member;
}
```

위의 코드에서 "dave"라는 Member가 파라미터로 계속 들어와도 "cachePut 실행" 이라는 문자열은 계속 출력됩니다. 
그리고 계속 캐싱을 업데이트 합니다.

## 문제 상황 

그런데 위의 ```@Cacheable```과 ```@CachePut```을 함께 이용한다면 데이터 정합성이 깨지는 문제가 발생할 수 있습니다. 


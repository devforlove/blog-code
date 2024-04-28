Redis에서는 다양한 형태의 Collection을 지원합니다. 이처럼 Redis에서 이미 Collection을 만들어 놓았기 때문에 개발자는 복잡한 기능 구현 없이 비즈니스 로직만 작성할 수 있습니다. 

그렇다면, 우리는 직접 Redis의 Collection을 구현하진 않더라도 어떻게 사용하는지는 제대로 알아야 할 것입니다. 이번 글에서는 Redis에서 지원하는 다양한 형태의 Collection을 알아보고, 이후 주의점에 대해서도 알아보려고 합니다. 
Redis에는 다음의 Collection들을 제공합니다. 
- Strings
- List
- Set
- Sorted Set
- Hash

## Strings 

### GET, SET
Redis에서 문자열에 값을 저장하기 위해 SET 명령어를 사용하고, 조회하기 위해 GET 명령어를 사용합니다.

```shell
> set bike:1 bike 
> get bike:1
```

만약 단일 명령으로 여러 키의 값을 설정하거나 검색하는 기능은 대기 시간을 줄이는 데도 유용합니다. 이러한 이유로 MSET과 MGET과 같은 명령이 있습니다. 
```shell
> mset bike:1 "Deimos" bike:2 "Ares" bike:3 "Vanth" 
> mget bike:1 bike:2 bike:3

# 결과 
1) "Deimos"
2) "Ares"
3) "Vanth"
```

### INCR
문자열이 Redis의 기본 값이더라도 이를 사용하여 원자 값을 증가할 수 있습니다. 

```shell
> set total_crashes 0
> incr total_crashes
# 결과 
1

```

INCR이 원자적이라는 것은 여러 클라이언트에서 동일한 키에 대해 INCR을 발행하더라도 결코 경쟁 조건에 빠지지 않습니다. 
예를 들어, 클라이언트 1이 "10"을 읽고, 클라이언트 2가 "10"을 동시에 읽고, 둘 다 11로 증가하고 새 값을 11로 설정하는 일은 결코 발생하지 않습니다. 
이는 Redis가 싱글 스레드로 동작하기 때문입니다. 싱글 스레드로 동작하기 때문에 싱글 스레드에 많은 작업 시간을 할당하는 O(n) 명령은 되도록 피해야 합니다.  

대부분 Redis의 문자열 연산은 O(1)로 매우 효율적입니다. **그러나 O(n)이 될 수 있는 SUBSTR, GETRANGE 명령에는 주의해야 합니다.** 이러한 무작위 엑세스 문자열 명령은 큰 문자열을 처리할 때 성능 문제를 일으킬 수 있습니다. 

## LIST

Redis의 List는 Linked List로 구현되어 있습니다. 따라서 Redis 목록은 다음과 같은 용도록 자주 사용됩니다. 
- 스택과 큐를 구현합니다. 
- 백그라운드 작업자 시스템을 위한 대기열 관리를 구축합니다. 

### LPUSH, RPUSH, LPOP, RPOP
LPUSH, RPUSH는 목록의 헤드에 새 요소를 추가합니다. LPUSH는 왼쪽 끝에 요소를 추가합니다. RPUSH는 오른쪽 끝에 요소를 추가합니다.
LPOP, RPOP는 목록의 헤드에 요소를 제거합니다. LPOP은 왼쪽 끝에 요소를 제거합니다. RPOP은 오른쪽 끝에 요소를 제거합니다. 

List 이러한 명령어를 이용해서 스택, 큐 자료구조를 손쉽게 구현할 수 있습니다.
아래는 큐를 구현한 내용입니다. 
```shell
> LPUSH bikes:repairs bike:1
> LPUSH bikes:repairs bike:2
> RPOP bikes:repairs
"bike:1"
> RPOP bikes:repairs
"bike:2"
```

Redis의 리스트는 연결 리스트를 통해 구현되어 있습니다. 따라서 왼쪽 끝, 오른쪽 끝에 원소를 저장하는 속도는 굉장히 빠릅니다. 하지만 특정 인덱스에 접근하거나 삭제하는 작업은 O(N) 시간 복잡도를 보입니다.

### LRANGE, LTRIM
LRANGE 명령은 목록에서 요소 범위를 추출합니다. 마지막 요소를 -1로 지정하게 되면 마지막 요소를 의미합니다. 즉 0번째 인덱스 부터 마지막 인덱스 까지를 의미합니다. 
```shell
> LPUSH bikes:repairs bike:1
> LPUSH bikes:repairs bike:2
> LPUSH bikes:repairs bike:important_bike
> LRANGE bikes:repairs 0 -1
1) "bike:important_bike"
2) "bike:1"
3) "bike:2"
```

LTRIM 명령은 List의 길이를 제한합니다. 
```shell
> RPUSH bikes:repairs bike:1 bike:2 bike:3 bike:4 bike:5
> LTRIM bikes:repairs 0 2
> LRANGE bikes:repairs 0 -1
1) "bike:1"
2) "bike:2"
3) "bike:3"
```

일반적으로 LTRIM과 LRANGE는 O(N) 명령이지만, 목록의 앞부분이나 뒷부분의 작은 범위에 엑세스하는 것은 일정한 시간 작업입니다. 
하지만 List의 크기가 커진다면, Redis의 작업에 병목이 발생하고 Redis 장애로 까지 번질 수 있습니다.



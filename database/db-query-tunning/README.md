제가 운영하는 프로젝트에는 현재 진행중진 이벤트를 가져오는 쿼리가 존재합니다. 하지만 테이블이 크고 일대다로 이어져 있는 데이터를 가져오면서 실행시간이 10초가 넘어가는 문제가 있었습니다. 

물론 해당 쿼리가 사용자마다 호출되는 것이 아니라 2분에 한번씩 호출되긴 했어도, 2분 마다 데이터베이스의 부하에 큰 부하를 주었습니다. 

문제가 되었던 쿼리의 대략적인 테이블 구조는 이렇습니다. 

![img.png](img.png)

```event``` 테이블은 event 정보 가지고 있습니다.
```ab_test``` 테이블은 각 이벤트의 A/B 테스트 정보를 가지고 있는 테이블입니다.
가장큰 문제는 1+N 문제였습니다. ```event``` 테이블과 ```ab_test``` 테이블은 일대다로 이어져 있기 때문입니다. 한번의 쿼리로 ```event```에서 조회되는 레코드 갯수만큼 ```ab_test``` 레코드를 조회합니다.

## 1+N 문제 해결 
만약 JPA를 사용했다면 ```default_batch_fetch_size``` 설정을 이용해 lazy loading 최적화를 이용해 쉽게 해결할 수 있었을 것입니다. 
하지만 해당  프로젝트에서 사용하고 있던 기술이 mybatis 였기에 쿼리 자체적으로 1+N 문제를 해결해야 했습니다.     

다음 쿼리는 기존의 쿼리를 간략히 한 sql입니다. 
```sql
select
*
from `event`
where
status in ('at', 'rn')
and sysdate() between start_date and end_date
and priority is not null
and activate_date is not null
order by priority desc,  activate_date asc
```

```sql
select
*
from ab_test
where
event_key = #{event_key}
order by nkey
```

위의 쿼리들은 ```event``` 조회 쿼리의 결과 만큼 ```ab_test```를 조회해야 했기에 DB에 부하를 주면서, 실행시간도 10초 이상으로 느립니다. 
이를 개선하기 위해 한 번의 쿼리로 ```event```와 ```ab_test```를 조회하도록 개선했습니다.
아래는 개선한 sql 입니다. 

```sql
SELECT 
`event`.*,
ab_test.*
FROM `event`
LEFT JOIN ab_test
ON `event`.event_key = ab_test.event_key
AND ab_test.event_key IS NOT null
WHERE `event`.status IN ('at', 'rn')
AND sysdate() >= `event`.start_date
AND sysdate() <= `event`.end_date
AND `event`.priority IS NOT null
AND `event`.activate_date IS NOT null
```

left outer join을 이용해서 ```ab_test``` 테이블의 칼럼들을 한꺼번에 가져오도록 개선하였습니다. 이젠 쿼리 한번에 N번의 쿼리가 추가적으로 나가는 문제는 개선이 되었습니다.
하지만 문제는 한 번에 필요한 레코드를 모두 가져오기 때문에, 너무나 많은 레코드를 가져오는 것이 문제가 되었습니다. 

직접 수행을 해보니 총 1000개의 레코드를 가져오는 것을 확인할 수 있었습니다. 

덕분에 CPU 사용률도 올라가는 문제가 있었습니다. 


## 커서 방식을 이용한 쿼리 분리 

한번에 많은 레코드를 가져오면 




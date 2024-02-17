
저장할 데이터가 증가함에 따라 개발자는 '데이터베이스'를 어떻게 확장할 것인가에 대해 고민하게 됩니다. 데이터베이스 확장은 결국 더 큰 장비로 확장하는 scale up 방식과, 더 많은 데이터 베이스 장비를 구축하는 scale out 방식으로 갈립니다. 

scale up 방식은 일반적으로 성능 확장이 더 쉽다는 장점이 있지만, 한계가 있습니다. 대형 장비는 대체적으로 가격이 비싸고 결국에는 더 확장할 수 없는 물리적 한계에 부딪힙니다. 
반면에 scale out은 수십대, 수백대의 장비를 관리해야 하기 때문에 관리가 더 어렵습니다. 대신 scale up에 비해 경제적이고 확장이 용이합니다. 

데이터의 물리적 양이 계속 많아지는 추세인 요즘 확장이 용이한 mongoDB의 필요성이 점점 커지고 있습니다.  
그리고 Spring은 Mongo DB를 지원합니다. 이번 글에서는 스프링에서 제공하는 ```spring-boot-starter-data-mongodb``` 사용법에 대해 알아볼 것입니다.

## Mongo DB 설정 
먼저 Mongo DB 서버에서 database를 만들어줍니다.
```shell
use myDB # db생성  
```
DB가 있으면 이동하고, 없으면 새로 생성해서 이동합니ㅏㄷ. 
```
switched to db myDB
```

그리고 Role을 생성해주어야 합니다. 
```shell
db.createRole( { role: "myRole", privileges: [ { resource: { db: "myDB", collection: "" }, actions: [ "insert", "find", "update", "remove" ] } ], roles: [ ] } );
```
insert, read, update, delete를 모두 할 수 있는 Role을 생성해주었습니다. 
다음으론 User를 생성해주었습니다. 
```shell
db.createUser( { user: "inwook.jung", pwd:  "test1234", roles: [ { "role" : "myRole", "db" : "myDB" } ] } )
```

## spring-data-mongodb 설정 

Spring에서 Mongo DB를 사용하기 위해서는 아래의 의존성을 추가해주어야 합니다.
```
implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
```

그리고 application.yml에는 다음과 같이 host, port, database, username, password를 지정해주어야 합니다. 방금전에 myDB Database에서 만들었던 사용자의 username, password를 지정해줍니다.

```yaml
spring:
  data:
    mongodb:
      host: localhost
      port: 27017
      database: myDB
      username: inwook.jung
      password: test1234
```



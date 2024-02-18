
저장할 데이터가 증가함에 따라 개발자는 '데이터베이스'를 어떻게 확장할 것인가에 대해 고민하게 됩니다. 데이터베이스 확장은 결국 더 큰 장비로 확장하는 scale up 방식과, 더 많은 데이터 베이스 장비를 구축하는 scale out 방식으로 갈립니다. 

scale up 방식은 일반적으로 성능 확장이 더 쉽다는 장점이 있지만, 한계가 있습니다. 대형 장비는 대체적으로 가격이 비싸고 결국에는 더 확장할 수 없는 물리적 한계에 부딪힙니다. 
반면에 scale out은 수십대, 수백대의 장비를 관리해야 하기 때문에 관리가 더 어렵습니다. 대신 scale up에 비해 경제적이고 확장이 용이합니다. 

데이터의 물리적 양이 계속 많아지는 추세인 요즘 확장이 용이한 mongoDB의 필요성이 점점 커지고 있습니다.  
그리고 Spring은 Mongo DB를 지원합니다. 이번 글에서는 스프링에서 제공하는 ```spring-data-mongodb``` 사용법에 대해 알아볼 것입니다.

## Mongo DB 설정 
먼저 Mongo DB 서버에서 database를 만들어줍니다.
```shell
use myDB # db생성  
```
DB가 있으면 이동하고, 없으면 새로 생성해서 이동합니다. 
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

그리고 다음과 같이 Document를 만들어줍니다. 
```java
@Getter
@Document(collection = "users")
public class User {

	@Id
	private String id;
	private String name;
	private int age;
}
```
```@Id``` 어노테이션이 추가된 필드가 _id 키를 가진 필드가 됩니다. 

다음은 리포지토리입니다. ```@EnableMongoRepositories```로 리포지토리가 있는 패키지를 basePackages로 지정해주었습니다. 리포지토리를 구현한 구현체가 빈으로 등록됩니다. 
```java
public interface UserRepository extends Repository<User, String> {

	List<User> findByNameStartingWith(String regexp);
	User save(User user);
	List<User> findAll();
	Optional<User> findById(String id);
	void deleteById(String id);
}

@Configuration
@EnableMongoRepositories(basePackages = "com.example.monogdbplayground.repositories")
public class MongoConfig {

}
```

## 도큐먼트 갱신 
Mongo DB에서 도큐먼트의 특정 부분만 갱신하는 경우가 많습니다. 부분 갱신에는 원자적 갱신 연산자를 사용합니다. 

### $inc 제한자
만약 도큐먼트의 특정 필드를 증가시키고 싶다면 $inc 제한자를 사용합니다.
필터 도큐먼트를 첫 번째 매개변수로, 변경 사항을 수정하는 수정자 도큐먼트를 두 번째 매개변수로 사용합니다. 

```json
{
  "_id": ObjectId("65d0646cc833341e64b9d3a2"),
  "name": "joe",
  "age": 16,
}
```

```shell
db.myDB.updateOne({"name" : "joe"},{"$inc" : {"age" : 1}})
```

spring-data-mongodb에서는 다음과 같이 $inc를 적용합니다.  
```java
public void inc(String name) {
    Query query = new Query(Criteria.where("name").is(name));
    
    Update update = new Update();
    update.inc("age", 1); // (1)

    mongoTemplate.updateFirst(query, update, User.class);
}
```
spring-data-mongodb에서도 필터 부분인 Query와 수정자 부분인 Update가 나뉘어져 있습니다. 그리고  (1)에서 inc 메서드를 이용해 $inc 제한자를 적용해줍니다. 

### $set 제한자 
$set 제한자는 필드 값을 설정합니다. 필드가 존재하지 않으면 새 필드가 생성됩니다.  

```json
{
  "_id": ObjectId("65d0646cc833341e64b9d3a2"),
  "name": "joe",
  "favorite book": "Ender's Game",
}
```

```shell
db.myDB.updateOne({"name" : "joe"},{"$set" : {"favorite book" : "Green Eggs and Ham"}})
```

아래는 $set을 spring-data-mongodb로 구현한 부분입니다. 
```java
public void set(String name) {
    Query query = new Query(Criteria.where("name").is(name));
    
    Update update = new Update();
    update.set("favorite book", "Green Eggs and Ham"); // (1)

    mongoTemplate.updateFirst(query, update, User.class);
}
```

### $push 제한자
$push 제한자는 배열이 이미 존재하면 배열 끝에 요소를 추가하고, 존재하지 않으면 새로운 배열을 생성합니다. 예를 들어 블로그 게시물에 배열 형태의 "comments" 키를 삽입한다고 가정하겠습니다.
Mongo DB에선 다음과 같이 특정 도큐먼트의 배열에 요소를 추가합니다.  

```json
{
  "_id": ObjectId("65d0646cc833341e64b9d3a2"),
  "name": "joe",
  "email": "joe@example.com",
  "content": [
    {
      "name": "kevein",
      "email", "kevin@example.com",
      "content": "hello"
    }
  ]
}
```

```shell
db.blog.posts.updateOne({"title" : "A blog post"}, {
  "$push": { "comments" : {
    "name": "joe", 
    "email", "joe@example.com", 
    "content": "nice post."
  }}
})
```

만약 여러개의 값을 추가해야 한다면 $each 제한자를 사용할 수 있습니다. 
```shell
db.blog.posts.updateOne({"title" : "A blog post"}, {
  "$push": { "comments" : {"$each": [
  {
    "name": "joe", 
    "email", "joe@example.com", 
    "content": "nice post."
  },
  {
    "name": "dave", 
    "email", "dave@example.com", 
    "content": "good."
  }
]}}})
```

spring-data-mongodb 에서 $push를 구현한 코드입니다. 
```java
public void updateMulti(String name) {
    Query query = new Query(Criteria.where("name").is(name));

    Update update = new Update();
    Document document = new Document()
            .append("name", "joe")
            .append("text", "good job");
    update.push("comments", document);

    mongoTemplate.updateFirst(query, update, User.class);
}
```
$push로 배열에 값을 추가할때 여러개의 값을 추가해야 한다면 $each를 사용해야 합니다. 
```java
public void push(String name) {
    Query query = new Query(Criteria.where("name").is(name));

    Update update = new Update();
    Document document1 = new Document()
            .append("name", "joe")
            .append("text", "good job");
    Document document2 = new Document()
    .append("name", "dave")
    .append("text", "hello");
    update.push("comments").each(document1, document2);

    mongoTemplate.updateFirst(query, update, User.class);
}
```

### $pull 제한자
$pull은 도큐먼트에서 조건과 일치하는 요소를 모두 제거합니다. 

```json
{
  "_id": ObjectId("65d0646cc833341e64b9d3a2"),
  "name": "joe",
  "todo": [
    "dishes",
    "dry cleaning",
    "dry cleaning",
    "laundry"
  ]
}
```

```shell
db.todo.updateOne({"name" : "joe"}, {
  "$pull": {"todo" : "dry cleaning"}})
```
결과를 보면 일치하는 요소인 dry cleaning이 모두 사라진 것을 확인할 수 있습니다. 
```json
{
  "_id": ObjectId("65d0646cc833341e64b9d3a2"),
  "name": "joe",
  "todo": [
    "dishes",
    "laundry"
  ]
}
```

아래는 spring-data-mongodb로 $pull을 구현한 코드입니다. 
```java
public void pull(String name) {
    Query query = new Query(Criteria.where("name").is(name));

    Update update = new Update();
    update.pull("comments", "dry cleaning");

    mongoTemplate.updateFirst(query, update, User.class);
}

```

### 배열의 위치 기반 변경 

도큐먼트의 배열을 변경하는 방법에는 두가지가 있습니다. 
- 배열의 위치를 이용하는 방법 
- 위치 연산자를 이용하는 방법 

만약 아래와 같은 도큐먼트가 있다고 가정하겠습니다. 
```json
{
  "_id": ObjectId("65d0646cc833341e64b9d3a2"),
  "content": "...",
  "comments": [
    {
      "commentId": 1,
      "comment": "good post",
      "author": "John",
      "votes": 0
    },
    {
      "commentId": 2,
      "comment": "i thought it was too short",
      "author": "Claire",
      "votes": 3
    },
    {
      "commentId": 3,
      "comment": "free watches",
      "author": "Alice",
      "votes": -5
    },
    {
      "commentId": 4,
      "comment": "vacation getaways",
      "author": "Lynn",
      "votes": -7
    }
  ]
}
```

comments의 첫번째 배열 요소를 변경하고 싶다면 아래와 같은 명령어를 이용하면 가능합니다. 
```shell
db.blog.updateOne({"post" : postId}, {"$inc": {"comments.0.votes" :  1}})
```
하지만 보통 도큐먼트를 쿼리해서 검사해보지 않고는 배열의 몇 번째 요소를 변경할지 알 수 없습니다. 그래서 Mongo DB에서는 일치하는 배열 요소 및 요소의 위치를 알아내서 갱신하는 위치 연산자 "$"를 제공합니다. 
```shell
db.blog.updateOne({"post" : postId, "comments.commentId" : commentId}, {"$inc" : {"comments.$.votes" : 1}})
```
도큐먼트 id(postId)와 comments의 id(commentId)를 제공하고 "$" 위치연산자를 이용해서 votes를 1 늘립니다. 
아래는 spring-data-mongodb로 구현한 코드입니다. 
```java
public void updateArray(String id, String commentId) {
    Query query = new Query(Criteria.where("id").is(id).and("comments.comment_id").is(commentId));

    Update update = new Update();
    update.set("comments.$.text", "I think so");

    mongoTemplate.updateFirst(query, update, User.class);
}
```

## 도큐먼트 조회 

Mongo DB에서 find 함수는 쿼리에 사용합니다. find의 첫 매개변수에 따라 어떤 도큐먼트를 가져올지 결정합니다. 빈 쿼리 도큐먼트는 컬렉션 내 모든 것과 일치합니다. 
```shell
db.myDB.find() #컬렉션 내 모든 도큐먼트 반환  
```

간단한 데이터형은 찾으려는 값만 지정하면 쉽게 쿼리할 수 있습니다. 예를 들어 "age"가 27인 모든 도큐먼트를 찾으려면 키/값 쌍을 다음처럼 추가하면 됩니다. 
```
db.users.find({"age" : 27})
```
쿼리 도큐먼트에 여러 개의 키/값 쌍을 추가할 수 있으며, '조건1 AND 조건2 AND 조건3 ... AND 조건N'으로 해석됩니다. 예를 들면 이름이 "Joe"이면서 나이가 27인 도큐먼트를 찾는다면 쿼리는 아래와 같습니다. 
```
db.users.find({"name" : "Joe", "age" : 27})
```

<, <=, >, >= 에 해당하는 비교 연산자는 각각 "$lt", "$lte", "$gt", "$gte" 입니다. 조합해 사용하면 특정 범위 내 값을 쿼리할 수 있습니다. 
```
db.users.find({"age" : {"$gte" : 27, "$lte" : 33}})
```

Mongo DB에서 OR 쿼리에는 두 가지 방법이 있습니다. 
"$in"은 하나의 키를 다양한 값과 비교하는 쿼리에 사용합니다. 예를 들면, 티켓 번호가 45, 392, 934인 티켓을 찾는다고 가정한다면 아래와 같은 쿼리가 필요합니다. 
```
db.ticket.find({"ticketNo" : {"$in" : [45, 392, 934]}})
```
만약 여러 키를 주어진 값과 비교하기 위해서는 다음과 같이 쿼리합니다. $or을 이용해서 ticketNo가 일치하거나 winner가 true인 도큐먼트를 반환합니다. 
```
db.ticket.find({"$or" : [{"ticketNo" : {"$in" : [45, 392, 934]}}, 
                {"winner" : true}]});
```

위의 모든 쿼리는 spring-data-mongodb 에서 ```@Query```를 이용하면 손쉽게 작성할 수 있습니다. 아래는 리포지로리에서 ```@Query```를 이용해서 쿼리를 작성한 코드입니다. 
```java
public interface UserRepository extends Repository<User, String> {
	@Query("{'age' : ?0}")
	List<User> findByAge(int age);
	
	@Query("{'name' : ?0, 'age' : ?1}")
	List<User> findByNameAndAge(String name, int age);
	
	@Query("{'age' : {'$gte' : ?0, '$lte' : ?1}}")
	List<User> findAgeGteAndLte(int gte, int lte);

	@Query("{'ticketNo' : {'$in' : ?0}}")
	List<User> findTicketNoIn(List<Integer> ticketNos);

	@Query("{'ticketNo' : {'$in' : ?0}}")
	List<User> findByTicketNoIn(List<Integer> ticketNos);

	@Query("{'$or' : [{'ticketNo' : {'$in' : ?0}}, {'winner' : ?1}]}")
	List<User> findByTicketNoInOrWinner(List<Integer> ticketNos, boolean isWinner);
}
```

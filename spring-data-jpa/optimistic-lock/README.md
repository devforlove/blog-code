## @Version
낙관적 락은 엔티티를 변경할 때, 다른 트랜잭션에서 변경하지 않을 거라 낙관적으로 가정하는 방법입니다.
따라서 락 기능을 사용하지 않고, 엔티티의 버전을 통해 동시성을 제어합니다.
Hibernate ORM은 ```@Version``` 어노테이션을 제공하여 낙관적 락을 제공합니다.

```java
@Entity
public class Board {

  @Id
  private String id;
  private String title;

  @Version
  private Integer version;
}
```
위 ```Board``` 엔티티가 변경될 때 마다 ```version``` 이 자동으로 하나씩 증가합니다. 그리고 엔티티를 수정할 때, 엔티티를 조회한 시점의 버전과 수정한 시점의 버전이 일치하지 않으면 예외가 발생합니다.

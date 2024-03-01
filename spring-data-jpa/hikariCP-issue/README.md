
어느날 아래와 같은 에러가 뜨면서 커넥션 연결이 실패하는 이슈가 발생했습니다. 
```sql
the last packet successfully received from the server was 30,035 milliseconds ago.
```

조금 더 로그를 분석해보니 이런 경고가 뜨고있었습니다. 아래와 같이 각 커넥션으로 연결을 시도하다가 모두 실패했습니다. 
```shell
2023-04-18 08:01:12.345  WARN 20 --- [nio-8080-exec-9] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@1d5d809b (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value. 
2023-04-18 08:01:12.346  WARN 20 --- [nio-8080-exec-2] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@656228eb (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value. 
2023-04-18 08:01:12.346  WARN 20 --- [nio-8080-exec-2] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@1ee6e7a8 (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value. 
2023-04-18 08:01:12.347  WARN 20 --- [nio-8080-exec-9] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@100d934e (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value. 
2023-04-18 08:01:12.347  WARN 20 --- [nio-8080-exec-2] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@316d65af (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value. 
2023-04-18 08:01:12.347  WARN 20 --- [nio-8080-exec-9] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@5a179137 (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value. 
2023-04-18 08:01:12.348  WARN 20 --- [nio-8080-exec-2] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@4cf3ac6d (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value. 
2023-04-18 08:01:12.348  WARN 20 --- [nio-8080-exec-9] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@2bbf9bb6 (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value. 
2023-04-18 08:01:12.348  WARN 20 --- [nio-8080-exec-2] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@38a2e068 (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value. 
2023-04-18 08:01:12.349  WARN 20 --- [nio-8080-exec-9] com.zaxxer.hikari.pool.PoolBase          : HikariPool-3 - Failed to validate connection com.mysql.cj.jdbc.ConnectionImpl@1ed2242a (No operations allowed after connection closed.). Possibly consider using a shorter maxLifetime value.
```

## 원인
문제는 커넥션 누수였습니다.

Application의 HikariCP의 maxTimeOut으로 인해 커넥션 연결을 끊기도 전에 MySQL에서 wait_timeout으로 인해 연결을 끊어버리면 위에서 발생했던 것처럼 커넥션 누수가 발생합니다. 
이는 HikariCP와 Tomcat-dbcp 철학이 달라서 발생한 문제입니다. 
- HikariCP는 Tomcat-dbcp와 달리 사용하지 않는 Connection을 빠르게 회수하도록 설계
- Tomcat-dbcp는 지속적으로 DB에 Validation Query를 보내서 커넥션이 끊어지지 않도록 설계


## HikariCP

아래는 ```PoolBase.isConnectionAlive()``` 메서드입니다. 

```java
boolean isConnectionDead(final Connection connection) {
  try {
     try {
        setNetworkTimeout(connection, validationTimeout);

        final var validationSeconds = (int) Math.max(1000L, validationTimeout) / 1000;

        if (isUseJdbc4Validation) {
           return !connection.isValid(validationSeconds);
        }

        try (var statement = connection.createStatement()) {
           if (isNetworkTimeoutSupported != TRUE) {
              setQueryTimeout(statement, validationSeconds);
           }

           statement.execute(config.getConnectionTestQuery());
        }
     }
     finally {
        setNetworkTimeout(connection, networkTimeout);

        if (isIsolateInternalQueries && !isAutoCommit) {
           connection.rollback();
        }
     }

     return false;
  }
  catch (Exception e) {
     lastConnectionFailure.set(e);
     logger.warn("{} - Failed to validate connection {} ({}). Possibly consider using a shorter maxLifetime value.",
                 poolName, connection, e.getMessage());
     return true;
  }
}
```

HikariConfig의 ```maxLifeTime```이 끝나면 커넥션 풀에서 **해당 커넥션을 종료 후 리소스(메모리)도 해제**합니다. 그리고 **새로운 커넥션 객체를 생성해서 커넥션 풀에 추가**합니다.

중요한 점은 ```wait_timeout```으로 인해 DBMS에서 이미 커넥션을 닫았다면 HikariCP는 어떤 행위도 할 수 없습니다. 그저 에러 로그만 찍힐 뿐입니다. 
즉, DBMS에서 ```wait_timeout```으로 커넥션을 끊기 전에 HikariCP의 ```maxLifeTime```으로 커넥션을 해제해야 합니다. 

## 해결 방법

아래는 이 문제를 해결하기 위한 잘 알려진 솔루션입니다. 

1. maxTimeOut 조정 

HikariCP에서는 maxTimeOut을 DB의 wait_timeout 보다 2~3초 낮게 설정하는 것을 권장합니다. 

문제는 현재 MySQL 운영 서버의 ```wait_timeout```이 15초이고, HikariCP의 ```maxTimeOut```의 최솟값이 30초입니다. 
```java
private void validateNumerics(){
    if(maxLifetime!=0&&maxLifetime<SECONDS.toMillis(30)){
        LOGGER.warn("{} - maxLifetime is less than 30000ms, setting to default {}ms.",poolName,MAX_LIFETIME);
        maxLifetime=MAX_LIFETIME;
    }
    // ... 생략
}
```

HikariCP는 ```maxTimeOut```을 30초보다 낮게 잡으면 Default인 30분으로 강제로 설정됩니다. 
- HikariCP에서 maxTimeOut이 너무 낮으면 성능 저하나 커넥션이 끊기는 이슈가 발생하는 것을 우려했다고 합니다. 

결국 해당 부분은 라이브러리를 교체하거나 내부를 뜯어고쳐야 합니다. 권장되지 않는 방법이라서 적용하지 않았습니다. 

2. wait_timeout 조정 

MySQL 서버의 wait_timeout을 조금 더 높게 수정하면 이를 해결할 수 있습니다. 
하지만 wait_timeout을 높이면 다른 시스템의 기존 레거시 코드에서 어떤 사이드 이펙트가 발생할지 예측할 수 없기 때문에 wait_timeout을 높일 수 없다는 답변을 받았습니다. 

결국 hikariCP의 ```maxTimeOut```과 MySQL 서버의 ```wait_timeout```을 변경하지 않고 해결할 방법을 찾아야 했습니다. 

## 적용한 방법 

그래서 결국 선택한 방법은 세션의 wait_timeout을 설정하는 방법입니다. 

wait_timeout 설정도 GLOBAL 설정이 있고, SESSION 설정이 있습니다. 
즉, SpringBoot 앱에서 MySQL 서버로 연경할 때 세션의 ```wait_timeout```을 바꿀 수 있다면 해결할 수 있지 않을까? 하는 생각이었습니다. 

```yaml
hikari:
  pool-name: HikariCP
  maximum-pool-size: 10
  connection-timeout: 10000
  validation-timeout: 10000
  max-lifetime: 580000
  # Session wait_timeout 수정
  connection-init-sql: set wait_timeout = 600
```

```spring.datasource.hikari.connection-init-sql``` 프로퍼티를 사용하면 Connection 객체가 생성될 때 세션 ```wait_timeout```을 수정할 수 있습니다.

문제가 생긴 프로젝트의 경우 다중 데이터 소스를 사용하고 있어서 아래와 같이 설정할 수 있었습니다.
```java
if(hikariConfig.getDriverClassName().equals("com.mysql.cj.jdbc.Driver")) {
    long waitTimeOut = TimeUnit.MILLISECONDS.toSeconds(hikariConfig.getMaxLifetime()) + 5;
    hikariConfig.setConnectionInitSql(String.format("SET SESSION wait_timeout = %s", waitTimeOut));
}
```
**세션의 wait_timeout을 HikariCP의 maxLifeTime보다 5초 더 높게 설정하였습니다.** 

결과적으로 더 이상 커넥션 누수가 발생하지 않았고 maxLifeTime을 늘리라는 경고가 나타나지도 않았습니다. 
성곡적으로 적용을 완료할 수 있었고 현재도 문제없이 운영중입니다.

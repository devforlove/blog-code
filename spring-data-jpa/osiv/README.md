이번 글에서는 JPA의 OSIV에 대해 알아보고자합니다. 

OSIV는 트랜잭션 외부에서 지연로딩이 가능하도록 하는 기능입니다. 보통은 트랜잭션이 종료되면 EntityManger도 종료되면서 Persistence Context도 회수됩니다. 
하지만 ```spring.jpa.open-in-view```를 true로 설정하면 트랜잭션 바깥에서도 Persistence Context가 종료되지 않습니다. 


## OpenEntityManagerInViewInterceptor

```OpenEntityManagerInViewInterceptor```는 OSIV를 실행시켜주는 인터셉터입니다. ```spring.jpa.open-in-view```를 true로 설정하면 ```JpaWebConfiguration```이 활성화 되면서 ```OpenEntityManagerInViewInterceptor```도 bean으로 등록됩니다.
아래는 ```JpaWebConfiguration```의 어노테이션입니다. 

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(WebMvcConfigurer.class)
@ConditionalOnMissingBean({ OpenEntityManagerInViewInterceptor.class, OpenEntityManagerInViewFilter.class })
@ConditionalOnMissingFilterBean(OpenEntityManagerInViewFilter.class)
@ConditionalOnProperty(prefix = "spring.jpa", name = "open-in-view", havingValue = "true", matchIfMissing = true) // spring.jpa.open-in-view가 true이면 bean등록 
protected static class JpaWebConfiguration {
```

```OpenEntityManagerInViewInterceptor```는 요청을 받으면 요청을 가로채서 ```EntityManager```를 생성합니다. 그리고 ```EntityManager```를 ```TransactionSynchronizationManager```에 저장합니다. 

```java
    @Override
	public void preHandle(WebRequest request) throws DataAccessException {
        ...
			logger.debug("Opening JPA EntityManager in OpenEntityManagerInViewInterceptor");
			try {
				EntityManager em = createEntityManager();
				EntityManagerHolder emHolder = new EntityManagerHolder(em);
				TransactionSynchronizationManager.bindResource(emf, emHolder);

				AsyncRequestInterceptor interceptor = new AsyncRequestInterceptor(emf, emHolder);
				asyncManager.registerCallableInterceptor(key, interceptor);
				asyncManager.registerDeferredResultInterceptor(key, interceptor);
			}
			catch (PersistenceException ex) {
				throw new DataAccessResourceFailureException("Could not create JPA EntityManager", ex);
			}
		}
	}
```

OSIV를 적용하면 ```@Transactional``` 이전에 이미 ```EntityManager```를 생성해놓음을 알 수 있습니다. 
```TransactionSynchronizationManager```를 통해 스레드에 ```EntityManager```가 바인딩 되면 추후에 ```EntityManager```의 라이프 사이클이 완전히 달라집니다. 

## doGetTransaction

```@Transactional```에 의해 트랜잭션이 시작되면 ```doGetTransaction``` 메서드가 호출됩니다. 
JpaTransactionManager의 ```doGetTransaction``` 메서드에서는 ```TransactionSynchronizationManager```에서 스레드에 할당된 ```EntityManager```를 찾습니다. 
OSIV가 설정되어 있다면 항상 스레드에 ```EntityManager```가 할당되어있습니다. 

여기서 중요한 부분은 setEntityManagerHolder에 newEntityManagerHolder를 false로 넘기는 부분입니다. 
OSIV가 설정되어 있다면 항상 ```EntityManager```는 이미 존재하기에 항상 newEntityManagerHolder는 false로 넘어갑니다. 
```java
EntityManagerHolder emHolder = (EntityManagerHolder)
				TransactionSynchronizationManager.getResource(obtainEntityManagerFactory());
if (emHolder != null) {
    if (logger.isDebugEnabled()) {
        logger.debug("Found thread-bound EntityManager [" + emHolder.getEntityManager() +
                "] for JPA transaction");
    }
    txObject.setEntityManagerHolder(emHolder, false); // OSIV 설정시 newEntityManagerHolder를 항상 false로 넘깁니다. 
}
```

```java
public void setEntityManagerHolder(
				@Nullable EntityManagerHolder entityManagerHolder, boolean newEntityManagerHolder) {

    this.entityManagerHolder = entityManagerHolder;
    this.newEntityManagerHolder = newEntityManagerHolder;
}
```

## doCleanupAfterCompletion


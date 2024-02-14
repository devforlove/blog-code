
이번 글은 우아한 기술블로그의 [포스팅](https://techblog.woowahan.com/2606/)을 바탕으로 합니다.

우연히 포스팅을 보다가 흥미로운 글을 발견했습니다. 보통의 상식으로 JPA에서 ```@Transactional```로 트랜잭션을 보장하는 상황에서 RuntimeException이 발생하면 롤백이 발생합니다.
그런데 만약 propagation으로 트랜잭션이 합쳐지는 상황에서 안쪽의 트랜잭션에서 발생한 RuntimeException을 바깥쪽의 트랜잭션에서 try/catch로 잡으면 정상적으로 커밋이될까요?
상식적으로는 try/catch로 RuntimeException을 잡았기 때문에 정상 커밋이 될 것 같지만, 예상과는 달리 롤백이 됩니다. 

```java
@Slf4j
@Service
@Transactional
@RequiredArgsConstructor
public class OuterService {

	private final InnerService innerService;

	public void outerAction() {
		try {
			innerService.innerAction();
		} catch (RuntimeException e) {
			log.warn("OuterService caught exception at outer. {}", e.getMessage());
		}
	}
}

@Service
@Transactional
@RequiredArgsConstructor
public class InnerService {

	private final ProductRepository productRepository;

	public void innerAction() {
		productRepository.save(
				Product.builder()
						.name("product")
						.price(1000)
						.build()
		);
		throw new RuntimeException("RuntimeException inside");
	}
}
```
위의 ```OuterService```를 호출하면 다음과 같은 콜스택이 발생하면서 롤백이 발생함을 알 수 있습니다. 
```
ransaction silently rolled back because it has been marked as rollback-only
org.springframework.transaction.UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only
	at app//org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(AbstractPlatformTransactionManager.java:803)
	at app//org.springframework.transaction.support.AbstractPlatformTransactionManager.commit(AbstractPlatformTransactionManager.java:757)
	at app//org.springframework.transaction.interceptor.TransactionAspectSupport.commitTransactionAfterReturning(TransactionAspectSupport.java:669)
	at app//org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:419)
	at app//org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
	at app//org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:184)
	at app//org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:765)
	at app//org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:717)
	at app//com.example.jpaplayground.rollback.OuterService$$SpringCGLIB$$0.outerAction(<generated>)
```

스택을 거꾸로 올라가면서 살펴보면 
1. 트랜잭션의 프록시 객체를 호출 
2. 프록시 객체 내부의 invokeWithTransaction 메서드 호출
3. commitTransactionAfterReturning 메서드 내부에서 commit 메서드 호출
4. processCommit 메서드에서 UnexpectedRollbackException 예외 발생

그리고 메세지는 ```rollback-only```로 마킹되었다고 합니다. 
추론해보면 inner 트랜잭션에서 발생한 예외로 인해 ```rollback-only```로 마킹이 되었고, outer 트랜잭션에서 ```rollback-only``` 마킹으로 인해 롤백이 되었다는 것을 알 수 있습니다.   

그렇다면 분명 outer 트랜잭션에서 예외를 try/catch로 잡았는데 롤백이 발생하는 이유는 뭘까요? 코드로 직접 확인해보겠습니다. 

## inner 트랜잭션에서 예외 발생 

스프링의 트랜잭션 로직을 살펴보면, 트랜잭션에서 예외 발생시 ```completeTransactionAfterThrowing``` 메서드가 호출됩니다.   
그리고 TransactionManager의 rollback 메서드가 호출됩니다. 
```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
						"] after exception: " + ex);
			}
			if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
        ...
```

그리고 rollback 메서드를 따라가다 보면 아래 코드에서 병합되는 트랜잭션은 ```doSetRollbackOnly``` 메서드에서 ```rollback-only```를 마킹합니다. 
```java
// Participating in larger transaction
...
if (status.hasTransaction()) {
    if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
        if (status.isDebug()) {
            logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
        }
        doSetRollbackOnly(status);
    }
	...
```
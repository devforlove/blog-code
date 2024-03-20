Spring Batch에선 Exception이 발생하면 작업이 종료됩니다. 이후 실패한 작업을 재시작 할 수 있지만, 예외가 너무 자주 발생한다면 작업 시간이 계속 늦어질 수 밖에 없습니다. 

그래서 Spring Batch에선 내결함성과 관련된 기능들을 제공합니다. 이번 글에서는 스프링 배치의 내결함성에 대해 알아보려고 합니다.

Spring Batch의 내결함성(fault tolerant)과 관련된 기능으로 skip과 retry가 있습니다.
- skip은 예외가 발생했을 때 예외가 발생한 Item을 건너뜁니다. 
- retry는 예외가 발생했을 때 예외가 발생한 Item을 재시도합니다. 

## Skip 

사용 방법은 아래와 같습니다. 
```java
@Bean(name = JOB_NAME + "step")
public Step step() {
    return new StepBuilder(JOB_NAME + "step", jobRepository)
            .<Product, ProductBackup>chunk(chunkSize, transactionManager)
            .reader(reader(null))
            .processor(processor())
            .writer(writer())
            .faultTolerant() // (1)
            .skip(NullPointerException.class) // (2)
            .skipLimit(3) // (3)
            .build();
}
```
(1) 첫번째로 ```faultTolerant``` 메서드를 사용합니다. 이 메서드를 사용하면 ```FaultTolerantStep```을 Step으로 사용하게 됩니다.   
(2) ```skip``` 메서드를 통해 어떤 예외를 건너뛸 것인지를 지정합니다.   
    - ```noSkip``` 메서드를 지정하면 건너 뛰지 않을 예외도 지정할 수 있습니다.
(3) ```skipLimit```은 예외를 몇번 건너뛸 것인지 지정합니다. 3이라고 지정했는데 예외가 4번 발생하면 배치 작업이 실패합니다. 

### ItemReader에서 예외 발생시 Skip 동작 

ItemReader에서 Item을 read 메서드에서 예외 발생시 ```Skip``` 가능하다면, 해당 Item을 건너뜁니다. 

```java
protected I read(StepContribution contribution, Chunk<I> chunk) throws Exception {
        while(true) {
            try {
                return this.doRead();
            } catch (Exception var4) {
                if (this.shouldSkip(this.skipPolicy, var4, contribution.getStepSkipCount())) {
                    contribution.incrementReadSkipCount();
                    chunk.skip(var4);
                    if (chunk.getErrors().size() >= this.maxSkipsOnRead) {
                        throw new SkipOverflowException("Too many skips on read");
                    }

                    this.logger.debug("Skipping failed input", var4);
                }
        ...
```
```shoulSkip``` 메서드에서 예외 발생 횟수가 skipLimit을 넘었는지 체크합니다. 만약 예외 발생횟수가 skipLimit을 넘지 않았다면 예외를 throw 하지 않고 다음 Item을 읽습니다. 

### ItemProcessor에서 예외 발생시 Skip 동작 

ItemProcessor에서 예외 발생시 skipLimit이 넘지 않았다면, Chunk의 처음으로 돌아가 다시 작업을 시작합니다. 
- 하지만 **이전 실행에서 발생한 예외 정보가 내부적으로 남아 있어 예외가 발생했던 item의 차례가 온다면 처리하지 않고 넘어갑니다.**
- 그리고 Chunk를 처음 부터 시작하더라도, 이미 조회된 캐싱된 데이터를 사용하기 때문에 ItemReader를 사용하진 않습니다.

예외가 발생한 Item을 건너뛰는 작업은 ```recoveryCallback``` 내부에서 발생합니다. 
```java
RecoveryCallback<O> recoveryCallback = new RecoveryCallback<O>() {
                public O recover(RetryContext context) throws Exception {
                    Throwable e = context.getLastThrowable();
                    if (FaultTolerantChunkProcessor.this.shouldSkip(FaultTolerantChunkProcessor.this.itemProcessSkipPolicy, e, contribution.getStepSkipCount())) {
                        iterator.remove(e);
                        contribution.incrementProcessSkipCount();
                        FaultTolerantChunkProcessor.this.logger.debug("Skipping after failed process", e);
                        return null;
                    } else if ((Boolean)FaultTolerantChunkProcessor.this.rollbackClassifier.classify(e)) {
                        throw new RetryException("Non-skippable exception in recoverer while processing", e);
                    } else {
                        iterator.remove(e);
                        return null;
                    }
                }
            };
```

```shouldSkip``` 메서드를 통해 Skip이 가능한 예외인지 판단한 이후에, Skip이 가능하다면 반복문에서 제거(remove) 합니다. 
따라서 예외가 발생했던 item은 제외하고 processor 작업을 진행하게 됩니다. 

### ItemWriter에서 예외 발생시 Skip 동작

ItemWriter에서 예외가 발생하면, skipLimit이 넘지 않았다면 Chunk의 처음으로 돌아가서 재동작합니다. 
- 다만 **ItemWriter는 Item을 하나씩만 받아서 처리합니다.** 

recoveryCallback을 보면 ```scan``` 메서드를 호출하는데, 따라가다 보면 Item을 개별적으로 ItemWriter에게 전달합니다. 
```java
recoveryCallback = new RecoveryCallback<Object>() {
                public Object recover(RetryContext context) throws Exception {
                    if (!FaultTolerantChunkProcessor.this.shouldSkip(FaultTolerantChunkProcessor.this.itemWriteSkipPolicy, context.getLastThrowable(), -1L)) {
                        throw new ExhaustedRetryException("Retry exhausted after last attempt in recovery path, but exception is not skippable.", context.getLastThrowable());
                    } else {
                        inputs.setBusy(true);
                        data.scanning(true);
                        FaultTolerantChunkProcessor.this.scan(contribution, inputs, outputs, FaultTolerantChunkProcessor.this.chunkMonitor, true);
                        return null;
                    }
                }
            };
```

```java
if (!outputs.isEmpty() && !inputs.isEmpty()) {
            Chunk<I>.ChunkIterator inputIterator = inputs.iterator();
            Chunk<O>.ChunkIterator outputIterator = outputs.iterator();
            if (!inputs.getSkips().isEmpty() && inputs.getItems().size() != outputs.getItems().size() && outputIterator.hasNext()) {
                outputIterator.remove();
            } else {
                Chunk<O> items = Chunk.of(new Object[]{outputIterator.next()});
                inputIterator.next();

                try {
                    this.writeItems(items);

```
아래 구문을 보면 ```writeItems``` 메서드에 하나의 item만 전달합니다. 
```java
Chunk<O> items = Chunk.of(new Object[]{outputIterator.next()});
```

**Skip을 사용하는 상황에서, ItemWriter에서 예외가 발생하면 ItemWriter가 Item을 하나씩 처리함을 알 수 있습니다.**


## Retry 

Retry는 Skip과 달리 예외가 발생해도 건너뛰지 않고, 재시도 합니다. 

### ItemProcessor에서 예외 발생시 Retry 동작

```java
if (this.canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {
    try {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Retry: count=" + context.getRetryCount());
        }

        lastException = null;
        result = retryCallback.doWithRetry(context);
        this.doOnSuccessInterceptors(retryCallback, context, result);
        Object var35 = result;
        return var35;
    } catch (Throwable var31) {
        Throwable e = var31;
        lastException = var31; 
    ...
```
위의 ```canRetry``` 메서드 내부에서 재시도 횟수를 체크합니다. 만약 예외가 발생했는데 재시도 횟수를 넘어가면 더 이상 재시도를 하지 않고 작업이 중단됩니다. 
만약 예외가 발생하면 Chunk의 처음으로 가서, reader 부터 작업을 합니다. 


### ItemWriter에서 예외 발생시 Retry 동작








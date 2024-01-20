이번 글에서는 저번 글에서 다루었던 멀티쓰레드 Step에 이어서 파티셔닝에 대해 다루어보고자 한다.
Spring Batch에서 파티셔닝은 멀티쓰레드 Step과 마찬가지로 Scaling 하는 방법 중에 하나다. 단 멀티쓰레드 Step은 하나의 Step 내에서 Chunk를 여러 쓰레드에서 수행했다면, 파티셔닝에서는 Master Step이 대량의 데이터 처리를 위해 지정된 수의 작업자 (Slave Step) 으로 일감을 분할 처리한다.

![img.png](images/img.png)

위의 그림에서 볼 수 있듯이, Master Step이 하나 있고, 내부적으로 Slave Step을 이용해서 멀티 쓰레드에서 분산처리한다. 즉, 쓰레드 갯수 만큼 Slave Step(Worker)을 생성해서 동시에 처리하는 것으로 이해할 수 있다.
멀티쓰레드 Step과 파티셔닝의 차이에 대해 좀 더 자세히 이야기 해보자면

- 멀티쓰레드 Step에서는 하나의 Step에서 Chunk를 여러 쓰레드에서 분할 처리한다. 따라서 어느 쓰레드에서 어떤 데이터를 처리하게 할지 세밀한 조정은 불가능하다. 
  - 또한, itemReader, itemWriter 등이 멀티쓰레드 환경을 지원하는지의 유무가 굉장히 중요하다.
- 반면 파티셔닝은 각 작업을 독립적인 Slave Step으로 분할한다. 그리고 각 Step은 별도의 StepExecution 파라미터 환경을 가지게 하여 처리된다.
  - 각 Step이 서로 독립적으로 동작하기 때문에, 앞서 멀티쓰레드 Step에서 수행했던 비동기 처리(synchronized)도 필요없다.
  - 파티셔닝에서는 itemReader, itemProcessor의 멀티쓰레드 환경 지원 여부도 중요하지 않다. 

파티셔닝에서는 데이터를 여러개의 Worker Step이 나누어서 처리하게 된다. 그리고 각 Worker Step은 ItemReader / ItemProcessor / ItemWriter 등을 가지고 동작하는 완전한 Step이다.
파티셔닝은 Partitioner와 PartitionHandler로 구성된다.

### Partitioner
파티셔너는 StepExecution에 매핑 할 Execution Context를 gridSize만큼 생성한다.
각 ExecutionContext에 저장된 정보를 바탕으로 Slave Step에서는 구역을 나누어 쓰레드마다 독립적으로 참조 및 활용한다. 

### PartitionHandler
PartitionHandler는 PartitionStep에 의해서 호출된다. 
내부적으로 TaskExecutor를 통해서 Worker Step을 병렬적으로 실행시킨다. 
PartitionHandler는 병렬로 실행 후 최종 결과를 담은 StepExecution을 PartitionStep에 반환해준다. 

그럼 이제 코드로 어떻게 Spring Batch에서 파티셔닝을 하는지 알아보자. 먼저 partitioner이다.
```java
@Slf4j
@RequiredArgsConstructor
public class ProductIdRangePartitioner implements Partitioner {

   private final ProductBatchRepository productBatchRepository;
   private final LocalDate startDate;
   private final LocalDate endDate;

   @Override
   public Map<String, ExecutionContext> partition(int gridSize) {
      long min = productBatchRepository.findMinId(startDate, endDate);
      long max = productBatchRepository.findMaxId(startDate, endDate);
      log.info("min={}", min);
      log.info("max={}", max);
      long targetSize = (max - min) / gridSize + 1;

      Map<String, ExecutionContext> result = new HashMap<>();
      long number = 0;
      long start = min;
      long end = start + targetSize - 1;

      while (start <= max) {
         ExecutionContext value = new ExecutionContext(); 
         result.put("partition" + number, value);// (1)

         if (end >= max) {
            end = max;
         }

         value.putLong("minId", start);
         value.putLong("maxId", end);
         start += targetSize;
         end += targetSize;
         number++;
      }

      return result;
   }
}
```
(1) 파티션당 ExecutionContext를 세팅하는 부분이다.

ExecutionContext에는 minId와 maxId가 세팅되어서 StepExecution으로 변경된다. StepExecution은 각 Step에게 전달되어 minId, maxId 정보를 전달한다. 
각 Slave Step은 독립적으로 itemReader, ItemProcessor, ItemWriter를 가지면서 minId 부터 maxId 까지의 아이템만 처리한다.

파티셔닝이 제대로 적용되는지 테스트 코드로 확인해보자.
```java
@Test
public void gridSize에_맞게_id가_분할되는지_test() {
   //given
   Mockito.lenient()
         .when(productBatchRepository.findMinId(any(LocalDate.class), any(LocalDate.class)))
         .thenReturn(1L);
   Mockito.lenient()
         .when(productBatchRepository.findMaxId(any(LocalDate.class), any(LocalDate.class)))
         .thenReturn(10L);

   partitioner = new ProductIdRangePartitioner(productBatchRepository, LocalDate.of(2021, 1, 20), LocalDate.of(2021, 1, 21));

   //when
   Map<String, ExecutionContext> executionContextMap = partitioner.partition(5); // (1)

   //then
   ExecutionContext partition1 = executionContextMap.get("partition0");
   assertThat(partition1.getLong("minId")).isEqualTo(1L);
   assertThat(partition1.getLong("maxId")).isEqualTo(2L);

   ExecutionContext partition5 = executionContextMap.get("partition4");
   assertThat(partition5.getLong("minId")).isEqualTo(9L);
   assertThat(partition5.getLong("maxId")).isEqualTo(10L);
}
```

- 위의 코드를 확인해보면 given 영역에서 minId가 1이 나오고, maxId가 10이 나오도록 설정했다. 즉 1~10의 id가 파티셔닝 되는 것이다. 그리고 (1)에서 gridSize를 5를 주었기 때문에 5개로 id가 나누어질 것이다.
- 쉽게 말하면 id는 (1~2) (3~4) (5~6) (7~8) (9~10) 이렇게 나뉜다. executionContextMap에는 "partition" + num을 키 Key로 minId, maxId는 Value로 가지고 있다.
- executionContextMap.get("partition0")으로 반환된 ExecutionContext는 minId가 1, maxId가 2로 나온다.
- executionContextMap.get("partition4")으로 반환된 ExecutionContext는 minId가 9, maxId가 10으로 나온다.

아래와 같이 정상적으로 테스트가 통과한다.

아래 코드는 파티셔닝을 적용하는 스프링 배치 코드이다.
```java
@RequiredArgsConstructor
@Configuration
@Slf4j
public class PartitionJobConfiguration {

	private final ProductBackupRepository productBackupRepository;

	private final ProductBatchRepository productBatchRepository;

	private final JobRepository jobRepository;
	private final EntityManagerFactory entityManagerFactory;
	private final PlatformTransactionManager transactionManager;
	private static int poolSize = 10;
	private static int chunkSize = 10;

	@Bean
	public TaskExecutorPartitionHandler partitionHandler() {
		TaskExecutorPartitionHandler partitionHandler = new TaskExecutorPartitionHandler();
		partitionHandler.setStep(slaveStep());
		partitionHandler.setTaskExecutor(executor());
		partitionHandler.setGridSize(poolSize);
		return partitionHandler;
	}

	@Bean
	public TaskExecutor executor() {
		ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
		executor.setCorePoolSize(poolSize);
		executor.setMaxPoolSize(poolSize);
		executor.setThreadNamePrefix("partition-thread");
		executor.setWaitForTasksToCompleteOnShutdown(Boolean.TRUE);
		executor.initialize();
		return executor;
	}

	@Bean
	public Job job_230410() {
		return new JobBuilder("job_230410", jobRepository)
				.start(stepManager())
				.build();
	}

	@Bean
	public Step stepManager() {
		return new StepBuilder("stepManager", jobRepository)
				.partitioner("slaveStep", partitioner(null, null))
				.step(slaveStep())
				.partitionHandler(partitionHandler())
				.build();
	}

	@Bean
	public Step slaveStep() {
		return new StepBuilder("slaveStep", jobRepository)
				.<Product, ProductBackup>chunk(chunkSize, transactionManager)
				.reader(reader(null, null))
				.processor(processor())
				.writer(writer(null, null))
				.build();
	}

	@Bean
	@StepScope
	public ProductIdRangePartitioner partitioner(
			@Value("#{jobParameters[startDate]}") String startDate,
			@Value("#{jobParameters[endDate]}") String endDate
	) {
		LocalDate startLocalDate = LocalDate.parse(startDate, DateTimeFormatter.ofPattern("yyyy-MM-dd"));
		LocalDate endLocalDate = LocalDate.parse(endDate, DateTimeFormatter.ofPattern("yyyy-MM-dd"));

		return new ProductIdRangePartitioner(productBatchRepository, startLocalDate, endLocalDate);
	}

	@Bean
	@StepScope
	public JpaPagingItemReader<Product> reader(
			@Value("#{stepExecutionContext[minId]}") Long minId,
			@Value("#{stepExecutionContext[maxId]}") Long maxId
	) {
		Map<String, Object> params = new HashMap<>();
		params.put("minId", minId);
		params.put("maxId", maxId);

		log.info("reader minId={}, maxId={}", maxId, maxId);

		return new JpaPagingItemReaderBuilder<Product>()
				.name("reader_230410")
				.entityManagerFactory(entityManagerFactory)
				.pageSize(chunkSize)
				.queryString(
						"SELECT p " +
						"FROM Product p " +
						"WHERE p.id BETWEEN :minId AND :maxId")
				.parameterValues(params)
				.build();
	}

	public ItemProcessor<Product, ProductBackup> processor() {
		return ProductBackup::new;
	}

	@Bean
	@StepScope
	public ItemWriter<ProductBackup> writer(
			@Value("#{stepExecutionContext[minId]}") Long minId,
			@Value("#{stepExecutionContext[maxId]}") Long maxId
	) {
		return items -> {
			productBackupRepository.saveAll(items);
		};
	}
}
```

위의 Spring Batch 파티셔닝 코드가 제대로 동작하는지 테스트코드로 검증해보자.
```java
@Test
void testJob() throws Exception {
   //given
   LocalDate txDate = LocalDate.of(2021, 1, 12);

   ArrayList<Product> products = new ArrayList<>();
   int expectedCount = 50;
   for(int i = 1; i <= expectedCount; i++) {
      products.add(new Product(i, txDate, ProductStatus.APPROVE));
   }
   productRepository.saveAll(products);

   JobParameters jobParameters = jobLauncherTestUtils.getUniqueJobParametersBuilder()
         .addString("startDate", txDate.format(FORMATTER))
         .addString("endDate", txDate.plusDays(1).format(FORMATTER))
         .toJobParameters();

   //when
   JobExecution jobExecution = jobLauncherTestUtils.launchJob(jobParameters);

   //then
   assertThat(jobExecution.getStatus()).isEqualTo(BatchStatus.COMPLETED);
   List<ProductBackup> backups = productBackupRepository.findAll();
   assertThat(backups.size()).isEqualTo(expectedCount);

   List<Map<String, Object>> metaTable = jdbcTemplate.queryForList("select step_name, status, commit_count, read_count, write_count from BATCH_STEP_EXECUTION");
   for(Map<String, Object> step : metaTable) {
      System.out.println("row=" + step);
   }
}
```

위의 테스트 코드에서 볼 수 있듯이, 50개의 아이템으로 테스트를 진행한다. Partitioner의 gridSize는 10으로 지정했기 때문에 Slave Step은 10개가 생성이 되어야 하고, 각 Step에서는 5개의 아이템에 대해 작업을 한다. 
그리고 이를 확인하기 위해 Batch_Step_Execution 테이블을 조회하니까 제대로 결과가 나온 것을 확인할 수 있었다. 

![img_1.png](images/img_1.png)

각 파티션이 5개의 아이템을 읽고, 5개의 row가 적재되어 있음을 확인할 수 있다.

## 마무리
파티셔닝을 통한 Sprign Batch의 Scaling 방법에 대해 알아보았다. 이번에 파티셔닝을 사용하면서 기존에 단일 쓰레드로 처리하던 Step이 있었다면, 코드 수정없이 파티셔닝을 적용해 볼 수 있겠다는 생각을 해보았다. Master Step, Partitioner, PartitionHandler만 추가해주면 파티셔닝이 깔끔하게 적용되기 때문이다.
파티셔닝은 아래와 같은 상황에서 사용이 적합하다.

조회 기간이 한달 정도 라면, 파티셔닝으로 2~3일로 분할해서 각각의 쓰레드가 처리하도록 할 수 있다.
전체 테이블의 데이터를 다른 테이블로 백업할 때도 id 기반으로 아이템을 분할하여 처리할 경우에 유용하다.

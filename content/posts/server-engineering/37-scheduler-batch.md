---
title: "[비동기 처리와 이벤트 드리븐] 4편 — 스케줄러와 배치 처리: 시간이 트리거인 작업들"
date: 2026-03-17T21:04:00+09:00
draft: false
tags: ["Spring Batch", "Quartz", "스케줄러", "배치", "비동기", "서버"]
series: ["비동기 처리와 이벤트 드리븐"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 5
summary: "Cron vs 분산 스케줄러와 단일 실행 보장, Spring Batch 아키텍처(Job, Step, Chunk, Tasklet), Quartz Scheduler 클러스터 모드와 미스파이어 전략, 대용량 배치 성능 최적화까지"
---

밤 12시, 데이터베이스에는 수백만 건의 정산 레코드가 쌓여 있다. 사용자 요청이 없어도, 누군가 버튼을 누르지 않아도, 시스템은 정확히 그 시각에 깨어나 작업을 시작한다. 스케줄러와 배치 처리는 실시간 요청-응답 패러다임과는 다른 세계다. 시간이 트리거가 되고, 대량의 데이터가 파이프라인을 흐르며, 실패는 반드시 복구 가능해야 한다. 단순해 보이는 `@Scheduled(cron = "0 0 0 * * *")` 한 줄 뒤에는, 분산 환경에서의 중복 실행 방지, 장애 복구 전략, 수억 건 처리의 성능 최적화라는 거대한 빙산이 숨어 있다. 이 글에서는 그 빙산 전체를 해부한다.

---

## Cron과 분산 스케줄러: 단일 실행 보장의 함정

### 전통적인 Cron의 한계

리눅스 crontab은 단순하고 강력하다. 하지만 서버가 두 대 이상이 되는 순간 근본적인 문제가 발생한다. 두 서버 모두 동일한 crontab을 가지고 있다면, 매일 자정에 정산 배치가 두 번 실행된다. 최악의 경우 중복 정산, 이중 메일 발송, 잘못된 통계가 생성된다.

```
# 서버 A의 crontab
0 0 * * * /app/scripts/daily-settlement.sh

# 서버 B의 crontab (동일)
0 0 * * * /app/scripts/daily-settlement.sh
# 매일 자정, 두 스크립트가 동시에 실행된다.
```

이 문제를 해결하려고 흔히 쓰는 안티패턴이 있다. 서버 한 대만 crontab을 가지도록 구성하는 것이다. 이른바 "배치 전용 서버"다. 이 서버가 장애를 일으키면? 배치는 영원히 실행되지 않는다. 단일 장애점(Single Point of Failure)을 만들어 놓고 고가용성을 이야기하는 모순이다.

### 분산 환경에서의 단일 실행 보장

분산 스케줄러의 핵심 과제는 **"N개의 노드 중 정확히 하나만 실행한다"**는 보장이다. 이를 구현하는 방식은 크게 세 가지다.

**1. 데이터베이스 락 기반**

가장 단순하고 범용적인 방식이다. 실행 전 DB에 락 레코드를 INSERT하고, 성공한 노드만 실행한다.

```java
@Component
public class DistributedScheduler {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Scheduled(cron = "0 0 0 * * *")
    public void dailySettlement() {
        String lockKey = "daily-settlement-" + LocalDate.now();

        try {
            // 락 획득 시도: INSERT는 UK 제약으로 하나만 성공
            int inserted = jdbcTemplate.update(
                "INSERT INTO scheduler_lock (lock_key, acquired_at, node_id) " +
                "VALUES (?, NOW(), ?) " +
                "ON DUPLICATE KEY UPDATE lock_key = lock_key", // MySQL
                lockKey, getNodeId()
            );

            if (inserted == 0) {
                log.info("Lock already acquired by another node. Skipping.");
                return;
            }

            log.info("Lock acquired. Starting daily settlement.");
            executeSettlement();

        } catch (DuplicateKeyException e) {
            log.info("Concurrent lock attempt detected. Skipping.");
        } finally {
            // 락 해제는 작업 완료 후 또는 TTL로 자동 처리
        }
    }
}
```

**2. Redis 분산 락**

DB보다 빠르고 TTL 설정이 자연스럽다. Redisson 라이브러리를 쓰면 더 견고하다.

```java
@Component
public class RedisDistributedScheduler {

    @Autowired
    private RedissonClient redissonClient;

    @Scheduled(cron = "0 0 0 * * *")
    public void dailySettlement() {
        String lockKey = "lock:daily-settlement:" + LocalDate.now();
        RLock lock = redissonClient.getLock(lockKey);

        boolean acquired = false;
        try {
            // 최대 0초 대기, 락 유지 시간 30분
            acquired = lock.tryLock(0, 30, TimeUnit.MINUTES);

            if (!acquired) {
                log.info("Another node is running the job. Skipping.");
                return;
            }

            executeSettlement();

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

**3. ZooKeeper / Curator 기반 리더 선출**

가장 정교한 방식이다. 클러스터에서 리더 노드가 선출되고, 리더만 스케줄을 실행한다. 리더가 죽으면 자동으로 새 리더가 선출된다.

```java
@Component
public class LeaderElectionScheduler {

    private final LeaderSelector leaderSelector;

    public LeaderElectionScheduler(CuratorFramework client) {
        this.leaderSelector = new LeaderSelector(
            client,
            "/scheduler/leader",
            new LeaderSelectorListenerAdapter() {
                @Override
                public void takeLeadership(CuratorFramework client) {
                    log.info("This node is now the leader.");
                    // 리더인 동안 계속 스케줄 실행
                    runSchedulerLoop();
                }
            }
        );
        leaderSelector.autoRequeue(); // 리더십을 잃으면 자동으로 재참여
        leaderSelector.start();
    }
}
```

### ShedLock: Spring 생태계의 실용적 해법

실무에서 가장 널리 쓰이는 라이브러리다. 어노테이션 하나로 분산 락을 적용할 수 있다.

```java
// build.gradle
implementation 'net.javacrumbs.shedlock:shedlock-spring:5.10.0'
implementation 'net.javacrumbs.shedlock:shedlock-provider-jdbc-template:5.10.0'

// Configuration
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "PT30M")
public class ShedLockConfig {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime() // 서버 시간 차이 문제 해결
                .build()
        );
    }
}

// Usage
@Component
public class SettlementScheduler {

    @Scheduled(cron = "0 0 0 * * *")
    @SchedulerLock(
        name = "daily-settlement",
        lockAtLeastFor = "PT5M",    // 최소 5분간 락 유지 (빠른 완료 후 재실행 방지)
        lockAtMostFor = "PT30M"     // 최대 30분 (프로세스 죽어도 락 자동 해제)
    )
    public void runDailySettlement() {
        LockAssert.assertLocked(); // 락 없이 호출되면 예외 발생 (테스트 안전장치)
        executeSettlement();
    }
}
```

ShedLock이 필요로 하는 테이블:

```sql
CREATE TABLE shedlock (
    name        VARCHAR(64)  NOT NULL,
    lock_until  TIMESTAMP(3) NOT NULL,
    locked_at   TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    locked_by   VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

---

## Spring Batch 아키텍처: 엔터프라이즈 배치의 표준

### 핵심 도메인 모델

Spring Batch의 아키텍처는 계층적이다. `Job`이 최상위이고, `Job`은 여러 `Step`으로 구성되며, 각 `Step`은 `Chunk` 기반이거나 `Tasklet` 기반이다.

```
Job
├── JobParameters (실행 구분 키)
├── JobInstance (논리적 실행 단위)
├── JobExecution (실제 실행 기록)
└── Step[]
    ├── StepExecution (각 스텝의 실행 기록)
    ├── Chunk 방식: ItemReader → ItemProcessor → ItemWriter
    └── Tasklet 방식: 단일 execute() 메서드
```

**JobRepository**는 이 모든 실행 이력을 DB에 저장한다. 덕분에 실패한 Job을 정확히 실패 지점부터 재시작할 수 있다.

### Chunk 기반 처리: 배치의 핵심

Chunk 처리는 "N건씩 읽고, 처리하고, 쓴다"는 패턴이다. 트랜잭션은 Chunk 단위로 커밋된다.

```java
@Configuration
public class OrderSettlementJobConfig {

    @Bean
    public Job orderSettlementJob(JobRepository jobRepository,
                                   Step settlementStep) {
        return new JobBuilder("orderSettlementJob", jobRepository)
            .incrementer(new RunIdIncrementer())
            .start(settlementStep)
            .build();
    }

    @Bean
    @JobScope
    public Step settlementStep(JobRepository jobRepository,
                                PlatformTransactionManager transactionManager,
                                @Value("#{jobParameters['targetDate']}") String targetDate) {
        return new StepBuilder("settlementStep", jobRepository)
            .<Order, SettlementRecord>chunk(1000, transactionManager) // 1000건씩 처리
            .reader(orderItemReader(targetDate))
            .processor(settlementProcessor())
            .writer(settlementWriter())
            .faultTolerant()
            .skipLimit(10)                          // 최대 10건 skip 허용
            .skip(InvalidOrderException.class)      // 이 예외는 skip
            .retryLimit(3)                          // 최대 3회 retry
            .retry(TransientDataAccessException.class) // 이 예외는 retry
            .build();
    }

    @Bean
    @StepScope
    public JdbcPagingItemReader<Order> orderItemReader(
            @Value("#{jobParameters['targetDate']}") String targetDate) {

        Map<String, Object> params = new HashMap<>();
        params.put("targetDate", targetDate);
        params.put("status", "COMPLETED");

        return new JdbcPagingItemReaderBuilder<Order>()
            .name("orderItemReader")
            .dataSource(dataSource)
            .selectClause("SELECT order_id, user_id, amount, created_at")
            .fromClause("FROM orders")
            .whereClause("WHERE DATE(created_at) = :targetDate AND status = :status")
            .sortKeys(Map.of("order_id", Order.ASCENDING))
            .parameterValues(params)
            .pageSize(1000)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
    }

    @Bean
    @StepScope
    public ItemProcessor<Order, SettlementRecord> settlementProcessor() {
        return order -> {
            if (order.getAmount() <= 0) {
                throw new InvalidOrderException("Invalid amount: " + order.getOrderId());
            }

            BigDecimal fee = order.getAmount().multiply(new BigDecimal("0.035"));
            BigDecimal netAmount = order.getAmount().subtract(fee);

            return SettlementRecord.builder()
                .orderId(order.getOrderId())
                .userId(order.getUserId())
                .grossAmount(order.getAmount())
                .fee(fee)
                .netAmount(netAmount)
                .settledAt(LocalDate.now())
                .build();
        };
    }

    @Bean
    @StepScope
    public JdbcBatchItemWriter<SettlementRecord> settlementWriter() {
        return new JdbcBatchItemWriterBuilder<SettlementRecord>()
            .dataSource(dataSource)
            .sql("INSERT INTO settlement_records " +
                 "(order_id, user_id, gross_amount, fee, net_amount, settled_at) " +
                 "VALUES (:orderId, :userId, :grossAmount, :fee, :netAmount, :settledAt)")
            .beanMapped()
            .build();
    }
}
```

### Tasklet: 단순 작업의 표현

Chunk가 과도한 경우, 단일 작업에는 Tasklet을 쓴다. 파일 이동, 테이블 트런케이트, 외부 API 호출 등에 적합하다.

```java
@Component
@StepScope
public class CleanupTasklet implements Tasklet {

    @Value("#{jobParameters['targetDate']}")
    private String targetDate;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) throws Exception {

        // 오래된 임시 데이터 정리
        int deleted = jdbcTemplate.update(
            "DELETE FROM temp_settlement WHERE created_at < ?",
            LocalDate.parse(targetDate).minusDays(7)
        );

        contribution.incrementWriteCount(deleted);
        log.info("Cleaned up {} temp records", deleted);

        return RepeatStatus.FINISHED; // CONTINUABLE이면 계속 실행
    }
}

// Step 설정
@Bean
public Step cleanupStep(JobRepository jobRepository,
                        PlatformTransactionManager transactionManager,
                        CleanupTasklet cleanupTasklet) {
    return new StepBuilder("cleanupStep", jobRepository)
        .tasklet(cleanupTasklet, transactionManager)
        .build();
}
```

### Job Flow: 조건부 실행과 분기

Step 간 의존관계와 조건부 흐름을 선언적으로 표현할 수 있다.

```java
@Bean
public Job complexJob(JobRepository jobRepository,
                       Step validationStep,
                       Step processStep,
                       Step notificationStep,
                       Step errorHandlingStep) {
    return new JobBuilder("complexJob", jobRepository)
        .start(validationStep)
            .on("FAILED").to(errorHandlingStep)
            .on("*").to(processStep)
        .from(processStep)
            .on("COMPLETED WITH SKIPS").to(notificationStep)
            .on("COMPLETED").to(notificationStep)
            .on("FAILED").fail()
        .from(errorHandlingStep)
            .on("*").end("STOPPED")
        .end()
        .build();
}
```

### JobParameters와 멱등성

Spring Batch의 `JobInstance`는 `Job + JobParameters` 조합으로 식별된다. 동일한 파라미터로 실행하면 이미 완료된 Job은 재실행되지 않는다. 이 특성을 활용해 배치의 멱등성을 보장한다.

```java
@Component
public class BatchJobLauncher {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job orderSettlementJob;

    public void launchSettlement(LocalDate targetDate) throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addString("targetDate", targetDate.toString())
            .addLong("timestamp", System.currentTimeMillis()) // 재실행 허용 시
            .toJobParameters();

        JobExecution execution = jobLauncher.run(orderSettlementJob, params);

        log.info("Job status: {}, ExitCode: {}",
            execution.getStatus(),
            execution.getExitStatus().getExitCode());
    }

    // 실패한 Job 재시작
    public void restartFailedJob(Long jobExecutionId) throws Exception {
        // Spring Batch는 FAILED 상태의 JobExecution을 자동으로 찾아 재시작
        // 동일한 JobParameters로 run() 호출 시 이전 실패 지점부터 재개
    }
}
```

---

## Quartz Scheduler: 클러스터 모드와 미스파이어 전략

### Quartz의 핵심 구성요소

```
Scheduler
├── Job (실행할 작업 인터페이스)
├── JobDetail (Job의 메타데이터 + JobDataMap)
├── Trigger (언제 실행할지)
│   ├── CronTrigger (Cron 표현식)
│   └── SimpleTrigger (고정 간격)
└── JobStore (스케줄 정보 저장소)
    ├── RAMJobStore (인메모리, 클러스터 불가)
    └── JDBCJobStore (DB 기반, 클러스터 가능)
```

### 클러스터 모드 설정

여러 노드가 동일한 DB를 바라보며, Quartz가 직접 분산 조율을 처리한다.

```yaml
# application.yml
spring:
  quartz:
    job-store-type: jdbc
    jdbc:
      initialize-schema: always
    properties:
      org.quartz.scheduler.instanceName: ClusteredScheduler
      org.quartz.scheduler.instanceId: AUTO  # 노드별 고유 ID 자동 생성

      org.quartz.jobStore.class: org.quartz.impl.jdbcjobstore.JobStoreTX
      org.quartz.jobStore.driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
      org.quartz.jobStore.tablePrefix: QRTZ_
      org.quartz.jobStore.isClustered: true
      org.quartz.jobStore.clusterCheckinInterval: 10000  # 10초마다 클러스터 체크

      org.quartz.threadPool.threadCount: 10
      org.quartz.threadPool.threadPriority: 5
```

Job과 Trigger 정의:

```java
@Component
public class QuartzJobConfig {

    @Bean
    public JobDetail settlementJobDetail() {
        return JobBuilder.newJob(SettlementQuartzJob.class)
            .withIdentity("settlementJob", "settlement")
            .withDescription("Daily settlement processing")
            .usingJobData("batchSize", 1000)
            .storeDurably()  // Trigger가 없어도 JobDetail 유지
            .build();
    }

    @Bean
    public CronTrigger settlementTrigger(JobDetail settlementJobDetail) {
        return TriggerBuilder.newTrigger()
            .forJob(settlementJobDetail)
            .withIdentity("settlementTrigger", "settlement")
            .withSchedule(
                CronScheduleBuilder
                    .cronSchedule("0 0 0 * * ?")  // 매일 자정
                    .withMisfireHandlingInstructionDoNothing() // 미스파이어 전략
                    .inTimeZone(TimeZone.getTimeZone("Asia/Seoul"))
            )
            .build();
    }
}

@Component
public class SettlementQuartzJob implements Job {

    @Autowired
    private BatchJobLauncher batchJobLauncher;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDataMap dataMap = context.getMergedJobDataMap();
        int batchSize = dataMap.getInt("batchSize");

        try {
            LocalDate targetDate = LocalDate.now().minusDays(1);
            batchJobLauncher.launchSettlement(targetDate);
        } catch (Exception e) {
            throw new JobExecutionException(e, false); // false: 재실행 안 함
        }
    }
}
```

### 미스파이어 전략 (Misfire Instructions)

미스파이어는 스케줄된 시각에 실행되지 못한 상황을 말한다. 서버 다운, 스레드 부족, DB 연결 실패 등이 원인이다. Quartz는 복구 후 이 상황을 어떻게 처리할지 전략을 제공한다.

**CronTrigger 미스파이어 전략:**

```java
// 1. MISFIRE_INSTRUCTION_DO_NOTHING
// 놓친 실행을 무시하고, 다음 스케줄 시각에만 실행
CronScheduleBuilder.cronSchedule("0 0 0 * * ?")
    .withMisfireHandlingInstructionDoNothing()

// 2. MISFIRE_INSTRUCTION_FIRE_NOW
// 복구 즉시 한 번 실행하고, 이후 정상 스케줄로 복귀
CronScheduleBuilder.cronSchedule("0 0 0 * * ?")
    .withMisfireHandlingInstructionFireAndProceed()

// 3. MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY (기본값)
// 놓친 모든 실행을 즉시 순차적으로 실행 (위험!)
```

**SimpleTrigger 미스파이어 전략:**

```java
// 5분 간격 실행, 미스파이어 전략 설정
SimpleScheduleBuilder.simpleSchedule()
    .withIntervalInMinutes(5)
    .repeatForever()
    .withMisfireHandlingInstructionNextWithRemainingCount()
    // 다음 실행 시각부터 재개, 놓친 횟수 차감

// 또는 즉시 재개
    .withMisfireHandlingInstructionNowWithRemainingCount()
```

**실무 전략 선택 가이드:**

| 상황 | 권장 전략 |
|------|-----------|
| 정산/집계 (하루 한 번) | `DO_NOTHING` - 이미 다음날 재실행 |
| 알림 발송 (시간 민감) | `FIRE_NOW` - 지연이지만 한 번은 실행 |
| 데이터 동기화 (누적 가능) | `IGNORE_MISFIRE` - 모든 미스파이어 처리 |
| 실시간 모니터링 (최신만 의미) | `DO_NOTHING` - 과거 데이터는 의미 없음 |

### Quartz의 Job 상태 관리

```java
// @DisallowConcurrentExecution: 동일 Job의 동시 실행 방지
@DisallowConcurrentExecution
@PersistJobDataAfterExecution  // JobDataMap 변경사항 DB에 저장
@Component
public class StatefulBatchJob implements Job {

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDataMap dataMap = context.getJobDetail().getJobDataMap();

        // 이전 실행에서 저장한 체크포인트 읽기
        Long lastProcessedId = dataMap.getLong("lastProcessedId");

        long newLastId = processFrom(lastProcessedId);

        // 다음 실행을 위해 체크포인트 저장
        dataMap.put("lastProcessedId", newLastId);
    }
}
```

---

## 대용량 배치 성능 최적화

### 파티셔닝: 데이터를 나눠 병렬 처리

수천만 건의 데이터를 단일 스레드로 처리하면 몇 시간이 걸린다. 파티셔닝은 데이터를 논리적 단위로 나누어 병렬 처리하는 전략이다.

```java
@Configuration
public class PartitionedSettlementJobConfig {

    @Bean
    public Step masterStep(JobRepository jobRepository,
                           Step workerStep,
                           Partitioner partitioner) {
        return new StepBuilder("masterStep", jobRepository)
            .partitioner("workerStep", partitioner)
            .step(workerStep)
            .gridSize(10)  // 10개 파티션
            .taskExecutor(new SimpleAsyncTaskExecutor())
            .build();
    }

    @Bean
    public Partitioner orderPartitioner() {
        return gridSize -> {
            // 전체 ID 범위를 gridSize등분
            long minId = jdbcTemplate.queryForObject(
                "SELECT MIN(order_id) FROM orders WHERE status = 'COMPLETED'", Long.class);
            long maxId = jdbcTemplate.queryForObject(
                "SELECT MAX(order_id) FROM orders WHERE status = 'COMPLETED'", Long.class);

            long rangeSize = (maxId - minId) / gridSize + 1;

            Map<String, ExecutionContext> partitions = new HashMap<>();
            for (int i = 0; i < gridSize; i++) {
                ExecutionContext context = new ExecutionContext();
                context.putLong("minId", minId + (long) i * rangeSize);
                context.putLong("maxId", minId + (long) (i + 1) * rangeSize - 1);
                context.putString("partitionName", "partition" + i);
                partitions.put("partition" + i, context);
            }
            return partitions;
        };
    }

    @Bean
    @StepScope
    public JdbcPagingItemReader<Order> partitionedOrderReader(
            @Value("#{stepExecutionContext['minId']}") Long minId,
            @Value("#{stepExecutionContext['maxId']}") Long maxId) {

        return new JdbcPagingItemReaderBuilder<Order>()
            .name("partitionedOrderReader")
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM orders")
            .whereClause("WHERE order_id BETWEEN :minId AND :maxId")
            .sortKeys(Map.of("order_id", Order.ASCENDING))
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .pageSize(500)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
    }
}
```

### 원격 파티셔닝: 여러 서버에 분산

단일 서버 병렬처리의 한계를 넘어, 여러 서버가 각 파티션을 처리한다. Spring Batch Integration과 메시지 큐를 활용한다.

```java
// Master 서버 설정
@Bean
public Step remotePartitioningMasterStep(JobRepository jobRepository,
                                          DirectChannel requests,
                                          PollableChannel replies) {
    return new RemotePartitioningMasterStepBuilder("masterStep", jobRepository)
        .partitioner("workerStep", orderPartitioner())
        .gridSize(10)
        .outputChannel(requests)   // 파티션 정보를 Worker에게 전송
        .inputChannel(replies)     // Worker의 완료 신호 수신
        .build();
}

// Worker 서버 설정
@Bean
public Step remotePartitioningWorkerStep(JobRepository jobRepository,
                                          DirectChannel requests,
                                          DirectChannel replies) {
    return new RemotePartitioningWorkerStepBuilder("workerStep", jobRepository)
        .inputChannel(requests)
        .outputChannel(replies)
        .<Order, SettlementRecord>chunk(500, transactionManager)
        .reader(partitionedOrderReader(null, null))
        .processor(settlementProcessor())
        .writer(settlementWriter())
        .build();
}
```

### 병렬 Step: 독립적인 Step의 동시 실행

Step 간 의존관계가 없다면 동시에 실행할 수 있다.

```java
@Bean
public Job parallelStepsJob(JobRepository jobRepository,
                             Step step1, Step step2, Step step3,
                             Step aggregationStep) {

    // step1, step2, step3를 동시에 실행하고, 모두 완료 후 aggregationStep 실행
    Flow parallelFlow = new FlowBuilder<SimpleFlow>("parallelFlow")
        .split(new SimpleAsyncTaskExecutor())
        .add(
            new FlowBuilder<SimpleFlow>("flow1").start(step1).build(),
            new FlowBuilder<SimpleFlow>("flow2").start(step2).build(),
            new FlowBuilder<SimpleFlow>("flow3").start(step3).build()
        )
        .build();

    return new JobBuilder("parallelStepsJob", jobRepository)
        .start(parallelFlow)
        .next(aggregationStep)
        .end()
        .build();
}
```

### 커밋 인터벌 최적화

커밋 인터벌(chunk size)은 성능에 가장 직접적인 영향을 미친다. 너무 작으면 트랜잭션 오버헤드가 크고, 너무 크면 메모리 부족과 실패 시 재처리 범위가 넓어진다.

```java
// 성능 테스트를 통한 최적 커밋 인터벌 도출
@Bean
@Profile("performance-test")
public Step benchmarkStep(JobRepository jobRepository,
                           PlatformTransactionManager transactionManager) {

    int[] intervals = {100, 500, 1000, 5000, 10000};
    // 각 인터벌로 10만 건 처리 시간 측정
    // 일반적으로 1000~5000이 DB 네트워크 왕복과 메모리의 균형점

    return new StepBuilder("benchmarkStep", jobRepository)
        .<Order, SettlementRecord>chunk(1000, transactionManager)
        .reader(orderItemReader("2026-01-01"))
        .processor(settlementProcessor())
        .writer(settlementWriter())
        .build();
}
```

**실측 기반 커밋 인터벌 가이드:**

| 데이터 특성 | 권장 인터벌 | 이유 |
|-------------|-------------|------|
| 단순 INSERT/UPDATE | 1,000~5,000 | 네트워크 왕복 최소화 |
| 복잡한 비즈니스 로직 포함 | 100~500 | 메모리와 실패 범위 제한 |
| 대용량 LOB 포함 | 50~200 | 메모리 OOM 방지 |
| 외부 API 호출 포함 | 1~10 | API 레이트 리밋 준수 |

### ItemReader 성능 최적화

```java
// 안티패턴: HibernateCursorItemReader의 무분별한 사용
// Hibernate 세션이 배치 전체에 걸쳐 열려있음 -> 메모리 폭발
@Bean
public HibernateCursorItemReader<Order> badReader() {
    return new HibernateCursorItemReaderBuilder<Order>()
        .queryString("FROM Order WHERE status = 'COMPLETED'")
        // 수백만 건이 세션 캐시에 쌓임
        .build();
}

// 권장: JdbcPagingItemReader + fetch size 튜닝
@Bean
public JdbcPagingItemReader<Order> goodReader() {
    return new JdbcPagingItemReaderBuilder<Order>()
        .name("orderReader")
        .dataSource(dataSource)
        .fetchSize(1000)  // JDBC fetch size: 네트워크 왕복 최소화
        .pageSize(1000)   // 페이지 크기: chunk size와 일치시키는 것이 효율적
        .selectClause("SELECT order_id, user_id, amount")
        .fromClause("FROM orders")
        .whereClause("WHERE status = 'COMPLETED'")
        .sortKeys(Map.of("order_id", Order.ASCENDING)) // 페이징에 필수
        .rowMapper(new BeanPropertyRowMapper<>(Order.class))
        .build();
}
```

### ItemWriter 성능 최적화: 배치 INSERT

```java
// 안티패턴: 건별 INSERT
@Bean
public ItemWriter<SettlementRecord> slowWriter() {
    return items -> {
        for (SettlementRecord record : items) {
            jdbcTemplate.update(
                "INSERT INTO settlement_records VALUES (?, ?, ?)",
                record.getOrderId(), record.getAmount(), record.getSettledAt()
            );
            // N번의 DB 왕복 발생
        }
    };
}

// 권장: JdbcBatchItemWriter의 배치 INSERT
@Bean
public JdbcBatchItemWriter<SettlementRecord> fastWriter() {
    return new JdbcBatchItemWriterBuilder<SettlementRecord>()
        .dataSource(dataSource)
        .sql("INSERT INTO settlement_records " +
             "(order_id, user_id, amount, settled_at) " +
             "VALUES (:orderId, :userId, :amount, :settledAt)")
        .beanMapped()
        // JDBC batch update 사용: 단 1번의 DB 왕복
        .build();
}

// MyBatis 사용 시 배치 실행기 활용
@Bean
@StepScope
public MyBatisBatchItemWriter<SettlementRecord> mybatisBatchWriter() {
    return new MyBatisBatchItemWriterBuilder<SettlementRecord>()
        .sqlSessionFactory(sqlSessionFactory)
        .statementId("com.example.mapper.SettlementMapper.insert")
        .build();
}
```

### 배치 모니터링과 메트릭

```java
@Component
public class BatchMetricsListener implements JobExecutionListener, StepExecutionListener {

    @Autowired
    private MeterRegistry meterRegistry;

    @Override
    public void afterJob(JobExecution jobExecution) {
        String jobName = jobExecution.getJobInstance().getJobName();
        String status = jobExecution.getStatus().name();

        long durationMs = jobExecution.getEndTime().toInstant().toEpochMilli()
            - jobExecution.getStartTime().toInstant().toEpochMilli();

        meterRegistry.counter("batch.job.completed",
            "job", jobName,
            "status", status
        ).increment();

        meterRegistry.timer("batch.job.duration",
            "job", jobName
        ).record(durationMs, TimeUnit.MILLISECONDS);

        if (jobExecution.getStatus() == BatchStatus.FAILED) {
            meterRegistry.counter("batch.job.failed", "job", jobName).increment();
            // 알림 발송
            alertService.sendAlert("Batch job failed: " + jobName);
        }
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        log.info("Step [{}] - Read: {}, Write: {}, Skip: {}, Duration: {}ms",
            stepExecution.getStepName(),
            stepExecution.getReadCount(),
            stepExecution.getWriteCount(),
            stepExecution.getSkipCount(),
            stepExecution.getEndTime().toInstant().toEpochMilli()
                - stepExecution.getStartTime().toInstant().toEpochMilli()
        );
        return null;
    }
}
```

### 흔한 안티패턴들

**1. @Transactional과 Chunk의 충돌:**

```java
// 안티패턴: ItemProcessor에 @Transactional 적용
@Component
public class BadProcessor implements ItemProcessor<Order, SettlementRecord> {

    @Transactional  // 이미 Chunk가 트랜잭션을 관리하는데 중첩 트랜잭션 발생
    public SettlementRecord process(Order order) {
        // Spring Batch의 Chunk 트랜잭션과 중첩되어 롤백 동작이 예측 불가능
        return convert(order);
    }
}
```

**2. JPA를 Batch에 직접 사용:**

```java
// 안티패턴: JpaItemWriter로 수백만 건 처리
@Bean
public JpaItemWriter<SettlementRecord> jpaWriter() {
    JpaItemWriter<SettlementRecord> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(entityManagerFactory);
    return writer;
    // 1차 캐시에 모든 엔티티가 쌓여 GC 압박 + 성능 저하
}

// 대안: JDBC 직접 사용 또는 벌크 쿼리
@Bean
public ItemWriter<SettlementRecord> bulkJpaWriter() {
    return items -> {
        entityManager.createQuery(
            "INSERT INTO SettlementRecord (orderId, amount) " +
            "SELECT o.id, o.amount FROM Order o WHERE o.id IN :ids"
        ).setParameter("ids", extractIds(items))
        .executeUpdate();
        entityManager.clear(); // 1차 캐시 강제 비우기
    };
}
```

**3. 무한 재시도:**

```java
// 안티패턴: retry 횟수 제한 없음
.faultTolerant()
.retryLimit(Integer.MAX_VALUE)  // 트랜잭션 예외에 무한 재시도 -> 배치 영원히 종료 안 됨
.retry(Exception.class)

// 올바른 설정: 재시도 가능한 예외를 명확히 구분
.faultTolerant()
.retryLimit(3)
.retry(TransientDataAccessException.class)  // 일시적 DB 오류만 재시도
.retry(ConnectTimeoutException.class)       // 네트워크 타임아웃만 재시도
.noRetry(InvalidDataException.class)        // 데이터 오류는 재시도 안 함
.skipLimit(100)
.skip(InvalidDataException.class)           // 대신 skip
```

---

## 실전 설계: 대규모 월정산 시스템

다음은 월 1억 건 이상의 거래를 정산하는 시스템의 실제 설계 패턴이다.

```java
@Configuration
public class MonthlySettlementJobConfig {

    // Phase 1: 데이터 준비 (병렬)
    @Bean
    public Flow dataPreparationFlow(Step loadOrdersStep,
                                     Step loadRefundsStep,
                                     Step loadPromoStep) {
        return new FlowBuilder<SimpleFlow>("dataPreparationFlow")
            .split(asyncTaskExecutor(20))
            .add(
                flow(loadOrdersStep),    // 주문 데이터 로드
                flow(loadRefundsStep),   // 환불 데이터 로드
                flow(loadPromoStep)      // 프로모션 할인 로드
            )
            .build();
    }

    // Phase 2: 정산 계산 (파티셔닝)
    @Bean
    public Step calculationMasterStep(JobRepository jobRepository,
                                       Step calculationWorkerStep) {
        return new StepBuilder("calculationMaster", jobRepository)
            .partitioner("calculationWorker", merchantPartitioner())
            .step(calculationWorkerStep)
            .gridSize(50)  // 50개 파티션 (가맹점 ID 범위 기준)
            .taskExecutor(asyncTaskExecutor(50))
            .build();
    }

    // Phase 3: 정산 확정 및 알림
    @Bean
    public Job monthlySettlementJob(JobRepository jobRepository,
                                     Flow dataPreparationFlow,
                                     Step calculationMasterStep,
                                     Step validationStep,
                                     Step notificationStep) {
        return new JobBuilder("monthlySettlementJob", jobRepository)
            .start(dataPreparationFlow).on("*")
            .to(calculationMasterStep)
            .next(validationStep)
            .next(notificationStep)
            .end()
            .build();
    }

    @Bean
    public TaskExecutor asyncTaskExecutor(int corePoolSize) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(corePoolSize * 2);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("batch-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

---

## 참고 자료

- [Spring Batch Reference Documentation](https://docs.spring.io/spring-batch/reference/) — Spring Batch 공식 레퍼런스, Job/Step/Chunk 아키텍처의 모든 것
- [Quartz Scheduler Documentation](https://www.quartz-scheduler.org/documentation/) — Quartz 클러스터 설정과 미스파이어 전략 상세 설명
- [ShedLock GitHub](https://github.com/lukas-krecan/ShedLock) — 분산 스케줄러 락 라이브러리, Spring 통합 가이드
- [Baeldung: Spring Batch Tutorial](https://www.baeldung.com/spring-batch-tutorial) — 실용적인 Spring Batch 예제 모음
- [Jens Schauder: Remote Partitioning with Spring Batch](https://spring.io/blog/2023/01/09/spring-batch-5-0-goes-ga) — Spring Batch 5의 원격 파티셔닝 패턴
- [박재성, 『스프링 부트와 AWS로 혼자 구현하는 웹 서비스』](https://jojoldu.tistory.com/category/Spring%20Batch) — 실무 Spring Batch 사례와 성능 최적화 노하우

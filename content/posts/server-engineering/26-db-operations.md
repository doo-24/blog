---
title: "[데이터베이스] 9편 — DB 운영: 프로덕션 DB 관리"
date: 2026-03-17T23:01:00+09:00
draft: false
tags: ["DB 운영", "백업", "리플리케이션", "커넥션 풀", "마이그레이션", "데이터베이스", "서버"]
series: ["데이터베이스"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 3
summary: "백업 전략(논리/물리, 풀/증분)과 복구 시나리오, 리플리케이션 원리(binlog)와 복제 지연 대응, HikariCP 커넥션 풀 설정, 무중단 스키마 마이그레이션(gh-ost), 모니터링 지표까지"
---

프로덕션 데이터베이스를 운영한다는 것은 단순히 서버를 켜두는 것이 아니다.

새벽 2시에 디스크가 가득 찼다는 알림을 받거나, 마이그레이션이 테이블을 통째로 잠근 채 15분째 완료되지 않고 있거나, 레플리카가 수십 초씩 뒤처지면서 캐시 미스가 폭발적으로 늘어나는 상황을 실시간으로 처리해야 할 때, 그 순간 당신의 손이 떨리지 않으려면 평소에 충분히 연습해 두어야 한다.

이 글은 그 연습을 위한 것이다. 백업과 복구, 리플리케이션, 커넥션 풀, 무중단 마이그레이션, 모니터링까지 — 프로덕션 DBA가 매일 다루는 주제들을 가능한 한 구체적으로 다룬다.

---

## 1. 백업 전략

데이터는 한 번 잃으면 돌이킬 수 없다. 하지만 백업이 있다고 안심해서는 안 된다.

백업이 존재한다는 것과 복구가 가능하다는 것은 전혀 다른 이야기다. 많은 팀이 "백업은 하고 있어요"라고 답하지만, 실제로 복구 훈련을 해본 적이 없어서 막상 사고가 났을 때 절차가 작동하지 않는 경우가 허다하다.

### 1.1 논리 백업 vs 물리 백업

**논리 백업(Logical Backup)**은 데이터를 SQL 문이나 CSV 형태로 추출하는 방식이다. MySQL의 `mysqldump`, PostgreSQL의 `pg_dump`가 대표적이다.

```bash
# MySQL 논리 백업 — 단일 데이터베이스
mysqldump \
  --single-transaction \
  --routines \
  --triggers \
  --hex-blob \
  -u root -p myapp_db > /backup/myapp_db_$(date +%Y%m%d_%H%M%S).sql

# 압축까지 한 번에
mysqldump --single-transaction -u root -p myapp_db \
  | gzip > /backup/myapp_db_$(date +%Y%m%d).sql.gz
```

`--single-transaction` 옵션이 중요하다. InnoDB 테이블에서 이 옵션 없이 덤프하면 테이블 락이 걸려서 운영 서비스에 영향을 준다.

이 옵션은 REPEATABLE READ 트랜잭션을 열어서 일관된 스냅샷을 보장한다. MyISAM 테이블이 섞여 있다면 효과가 없으니 주의해야 한다.

논리 백업의 장점은 이식성이다. 메이저 버전이 다른 MySQL 인스턴스에도 복원할 수 있고, 특정 테이블만 선택적으로 복원하는 것도 쉽다. 단점은 속도다. 100GB 데이터베이스를 덤프하면 수십 분이 걸리고, 복원은 그보다 더 오래 걸린다.

**물리 백업(Physical Backup)**은 데이터 파일 자체를 복사하는 방식이다. Percona XtraBackup(MySQL), pg_basebackup(PostgreSQL)이 여기에 해당한다.

```bash
# XtraBackup — 풀 백업
xtrabackup \
  --backup \
  --target-dir=/backup/full_$(date +%Y%m%d) \
  --user=root \
  --password=yourpassword

# 백업 후 prepare (InnoDB 로그 적용)
xtrabackup --prepare --target-dir=/backup/full_20240318

# 복원
systemctl stop mysql
xtrabackup --copy-back --target-dir=/backup/full_20240318
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```

물리 백업은 빠르다. 100GB 데이터베이스도 IO 속도에 의존하므로 수 분 내에 끝날 수 있다. XtraBackup은 hot backup을 지원하여 서비스 중단 없이 InnoDB 테이블을 백업한다.

### 1.2 풀 백업과 증분 백업

매일 풀 백업을 하는 것은 스토리지와 시간을 낭비한다. 실용적인 전략은 풀 백업과 증분 백업을 조합하는 것이다.

```
주간 풀 백업 전략 예시:
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│  월(풀)  │  화(증분) │  수(증분) │  목(증분) │  금(증분) │  토(증분) │  일(증분) │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

XtraBackup 증분 백업:

```bash
# 월요일: 풀 백업
xtrabackup --backup \
  --target-dir=/backup/full_monday \
  --user=root --password=pass

# 화요일: 풀 백업을 기준으로 증분
xtrabackup --backup \
  --target-dir=/backup/inc_tuesday \
  --incremental-basedir=/backup/full_monday \
  --user=root --password=pass

# 수요일: 화요일 증분을 기준으로 증분
xtrabackup --backup \
  --target-dir=/backup/inc_wednesday \
  --incremental-basedir=/backup/inc_tuesday \
  --user=root --password=pass
```

복원할 때는 순서대로 prepare해야 한다:

```bash
# 1단계: 풀 백업 prepare (--apply-log-only로 롤백 안 함)
xtrabackup --prepare --apply-log-only \
  --target-dir=/backup/full_monday

# 2단계: 화요일 증분 적용
xtrabackup --prepare --apply-log-only \
  --target-dir=/backup/full_monday \
  --incremental-dir=/backup/inc_tuesday

# 3단계: 수요일 증분 적용 (마지막이면 --apply-log-only 제거)
xtrabackup --prepare \
  --target-dir=/backup/full_monday \
  --incremental-dir=/backup/inc_wednesday

# 4단계: copy-back
xtrabackup --copy-back --target-dir=/backup/full_monday
```

### 1.3 바이너리 로그를 활용한 Point-in-Time Recovery

증분 백업만으로는 마지막 백업 시점 이후의 데이터를 복구할 수 없다. 바이너리 로그(binlog)를 함께 보관하면 특정 시점(Point-in-Time)으로 복구가 가능하다.

MySQL 설정:

```ini
# /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
server-id       = 1
log_bin         = /var/log/mysql/mysql-bin.log
binlog_format   = ROW
expire_logs_days = 14
max_binlog_size  = 256M
```

특정 시점 복구 시나리오: 오늘 오전 10시에 `DROP TABLE orders;`가 실행되었다고 가정하자.

```bash
# 1. 어젯밤 풀 백업 복원
xtrabackup --copy-back --target-dir=/backup/full_20240318

# 2. 드롭 명령 직전까지의 binlog 찾기
mysqlbinlog --start-datetime="2024-03-18 00:00:00" \
            --stop-datetime="2024-03-18 09:59:59" \
            /var/log/mysql/mysql-bin.000123 > recovery.sql

# 3. 복구 SQL 적용
mysql -u root -p < recovery.sql
```

특정 포지션 기준으로도 가능하다:

```bash
# binlog 내용 확인
mysqlbinlog --no-defaults /var/log/mysql/mysql-bin.000123 | grep -A 5 "DROP TABLE"

# 해당 포지션 바로 전까지만 적용
mysqlbinlog --start-position=4 --stop-position=98765 \
  /var/log/mysql/mysql-bin.000123 | mysql -u root -p
```

### 1.4 복구 훈련 (Restore Drill)

**가장 중요한 원칙: 복구 훈련을 정기적으로 하라.** 백업 파일이 존재해도 실제로 복원이 가능한지 확인하지 않으면 사고 당일에야 절차가 작동하지 않는다는 사실을 알게 된다.

한 달에 한 번, 별도의 스테이징 환경에서 다음을 검증하라:

1. 백업 파일이 손상 없이 존재하는가
2. 복원 절차를 처음 보는 사람도 따라할 수 있는가
3. 복원 완료까지 얼마나 걸리는가 (RTO 검증)
4. 복원된 데이터가 예상 시점과 일치하는가 (RPO 검증)

```bash
# 백업 무결성 검증 스크립트 예시
#!/bin/bash
BACKUP_FILE="/backup/myapp_db_20240318.sql.gz"
TEST_DB="restore_test_$(date +%H%M%S)"

echo "백업 파일 체크섬 검증..."
sha256sum -c /backup/checksums.txt

echo "테스트 DB 복원 시작..."
mysql -u root -p -e "CREATE DATABASE $TEST_DB;"
zcat $BACKUP_FILE | mysql -u root -p $TEST_DB

echo "데이터 카운트 검증..."
PROD_COUNT=$(mysql -u root -p -e "SELECT COUNT(*) FROM myapp_db.orders;" -s)
TEST_COUNT=$(mysql -u root -p -e "SELECT COUNT(*) FROM $TEST_DB.orders;" -s)

if [ "$PROD_COUNT" != "$TEST_COUNT" ]; then
  echo "경고: 레코드 수 불일치 ($PROD_COUNT vs $TEST_COUNT)"
fi

# 정리
mysql -u root -p -e "DROP DATABASE $TEST_DB;"
```

---

## 2. 리플리케이션 원리와 복제 지연 대응

### 2.1 MySQL 리플리케이션의 작동 원리

MySQL 리플리케이션의 핵심은 바이너리 로그(binlog)다. 마스터(소스)에서 발생하는 모든 데이터 변경이 binlog에 기록되고, 레플리카가 이를 읽어서 동일한 변경을 재실행한다.

```
마스터 노드                    레플리카 노드
┌─────────────────┐           ┌─────────────────────────────┐
│  애플리케이션     │           │  I/O Thread                 │
│       ↓         │           │  (마스터 binlog 읽어 relay   │
│  트랜잭션 실행   │◄──────────│   log에 기록)               │
│       ↓         │  binlog   │         ↓                   │
│  binlog 기록    │  stream   │  SQL Thread                  │
│                 │           │  (relay log의 이벤트를       │
│                 │           │   순차적으로 실행)            │
└─────────────────┘           └─────────────────────────────┘
```

리플리케이션 설정 (MySQL 8.0 GTID 방식):

```ini
# 마스터 my.cnf
[mysqld]
server-id       = 1
log_bin         = mysql-bin
gtid_mode       = ON
enforce_gtid_consistency = ON
binlog_format   = ROW

# 레플리카 my.cnf
[mysqld]
server-id       = 2
gtid_mode       = ON
enforce_gtid_consistency = ON
read_only       = ON
relay_log       = relay-bin
log_replica_updates = ON
```

마스터에서 복제 계정 생성:

```sql
CREATE USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'StrongPass!';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
```

레플리카에서 복제 시작 (GTID 방식):

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='192.168.1.10',
  SOURCE_PORT=3306,
  SOURCE_USER='replicator',
  SOURCE_PASSWORD='StrongPass!',
  SOURCE_AUTO_POSITION=1;

START REPLICA;

-- 상태 확인
SHOW REPLICA STATUS\G
```

### 2.2 binlog 포맷: STATEMENT vs ROW vs MIXED

binlog 포맷 선택은 중요하다.

| 포맷 | 장점 | 단점 |
|------|------|------|
| STATEMENT | binlog 크기 작음 | 비결정적 함수(NOW, UUID) 재실행 시 불일치 위험 |
| ROW | 정확한 재현 보장 | binlog 크기 큼, 대량 업데이트 시 매 행 기록 |
| MIXED | 자동 선택 | 예측하기 어려운 동작 |

프로덕션 환경에서는 **ROW** 포맷을 강하게 권장한다. 복제 안전성이 최우선이기 때문이다.

```ini
binlog_format   = ROW
binlog_row_image = FULL  # 변경 전후 모든 컬럼 기록 (기본값)
# MINIMAL로 설정하면 변경된 컬럼만 기록 — 공간 절약되지만 일부 도구에서 문제
```

### 2.3 복제 지연(Replication Lag) 원인과 대응

복제 지연은 `SHOW REPLICA STATUS`의 `Seconds_Behind_Source` 값으로 확인한다. 이 값이 커지면 레플리카에서 읽는 데이터가 마스터보다 오래된 것이다.

**원인 1: 단일 SQL 스레드 병목**

MySQL 5.6 이전에는 SQL 스레드가 하나뿐이라 마스터에서 병렬로 커밋된 트랜잭션도 레플리카에서는 순차 실행된다. 마스터가 초당 100개의 트랜잭션을 처리해도 레플리카는 단일 스레드로 하나씩 적용하므로 구조적으로 따라잡지 못한다.

해결책: 병렬 복제(Parallel Replication) 활성화

```ini
# MySQL 8.0 레플리카 설정
[mysqld]
replica_parallel_workers = 4          # CPU 코어 수 고려
replica_parallel_type    = LOGICAL_CLOCK  # 동시에 커밋된 트랜잭션 병렬 실행
replica_preserve_commit_order = ON    # 커밋 순서 보장
```

**원인 2: 대용량 트랜잭션**

마스터에서 `UPDATE orders SET status = 'archived' WHERE created_at < '2020-01-01';` 같이 수백만 행을 한 트랜잭션으로 처리하면 레플리카에서 동일한 작업을 수행하는 동안 다른 이벤트가 밀린다.

해결책: 배치 처리

```sql
-- 한 번에 1000건씩 처리
SET @done = 0;
REPEAT
  UPDATE orders SET status = 'archived'
  WHERE created_at < '2020-01-01'
    AND status != 'archived'
  LIMIT 1000;
  SET @done = ROW_COUNT() = 0;
  DO SLEEP(0.1);  -- 레플리카 따라잡을 시간 부여
UNTIL @done END REPEAT;
```

**원인 3: 레플리카 하드웨어 열세**

마스터보다 느린 디스크를 쓰는 레플리카는 구조적으로 뒤처진다.

```ini
# 레플리카 I/O 성능 향상
[mysqld]
innodb_flush_log_at_trx_commit = 2  # 마스터는 1이어야 하지만 레플리카는 2 허용
sync_binlog                    = 0   # 레플리카에서 binlog 동기화 완화
```

**복제 지연 모니터링**

```sql
-- 복제 지연 확인
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source: 현재 지연 초

-- Performance Schema로 세부 확인
SELECT
  CHANNEL_NAME,
  SERVICE_STATE,
  LAST_ERROR_MESSAGE,
  LAST_HEARTBEAT_TIMESTAMP
FROM performance_schema.replication_connection_status;

SELECT
  CHANNEL_NAME,
  SERVICE_STATE,
  LAST_APPLIED_TRANSACTION,
  APPLYING_TRANSACTION
FROM performance_schema.replication_applier_status_by_worker;
```

**복제 지연이 발생했을 때 애플리케이션 대응**

읽기 쿼리를 레플리카로 분산하는 구조에서 복제 지연이 생기면, 방금 쓴 데이터가 레플리카에서 조회되지 않는 문제가 발생한다.

```java
// Spring DataSource 라우팅 예시
// 쓰기 후 즉시 읽어야 하는 경우 마스터 강제 사용
@Transactional
public Order createOrder(OrderRequest req) {
    Order order = orderRepository.save(new Order(req));
    // 이 트랜잭션 내의 읽기도 마스터로 간다
    return orderRepository.findById(order.getId()).orElseThrow();
}

// 중요하지 않은 조회는 레플리카 허용
@Transactional(readOnly = true)
public Page<Order> listOrders(Pageable pageable) {
    // readOnly=true → DataSourceRouter가 레플리카 선택
    return orderRepository.findAll(pageable);
}
```

### 2.4 복제 에러 처리

복제 에러가 발생하면 SQL 스레드가 중단된다. `SHOW REPLICA STATUS\G`에서 `Last_SQL_Error`를 확인하라.

```sql
-- 에러 확인
SHOW REPLICA STATUS\G
-- Last_SQL_Error: Error 'Duplicate entry ...' on query ...

-- 특정 에러 코드 무시 (임시 해결책, 주의해서 사용)
STOP REPLICA SQL_THREAD;
SET GLOBAL replica_skip_errors = '1062';  -- 1062: duplicate key
START REPLICA SQL_THREAD;

-- 또는 단일 트랜잭션 건너뛰기 (GTID 방식)
STOP REPLICA;
SET GTID_NEXT='master_uuid:transaction_id';
BEGIN; COMMIT;  -- 빈 트랜잭션으로 해당 GTID 소비
SET GTID_NEXT='AUTOMATIC';
START REPLICA;
```

---

## 3. 커넥션 풀 설정과 풀 고갈 진단

### 3.1 커넥션 풀이 필요한 이유

데이터베이스 커넥션 생성은 비싼 작업이다. TCP 핸드쉐이크, 인증, 세션 초기화 — 이 과정이 매 요청마다 반복되면 응답 지연이 눈에 띄게 늘어난다.

커넥션 풀은 미리 만들어둔 커넥션을 재사용함으로써 이 오버헤드를 제거한다.

### 3.2 HikariCP 핵심 설정

HikariCP는 Java 생태계에서 사실상 표준인 커넥션 풀 라이브러리다.

```yaml
# Spring Boot application.yml
spring:
  datasource:
    url: jdbc:mysql://db.example.com:3306/myapp?useSSL=true&serverTimezone=UTC
    username: appuser
    password: secret
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      # 풀 사이즈
      minimum-idle: 5           # 유휴 상태에서 유지할 최소 커넥션 수
      maximum-pool-size: 20     # 최대 커넥션 수

      # 타임아웃
      connection-timeout: 3000  # 커넥션 획득 대기 최대 시간 (ms) — 기본 30s는 너무 길다
      idle-timeout: 600000      # 유휴 커넥션 제거 전 대기 시간 (10분)
      max-lifetime: 1800000     # 커넥션 최대 수명 (30분) — DB의 wait_timeout보다 짧게

      # 검증
      keepalive-time: 60000     # 유휴 커넥션 keepalive 주기 (1분)
      connection-test-query: SELECT 1  # 커넥션 유효성 검증 쿼리

      # 이름 (모니터링에서 구분 용이)
      pool-name: MyApp-HikariCP
```

### 3.3 풀 사이즈 계산

흔한 실수: "커넥션이 많으면 많을수록 좋다"는 생각으로 `maximum-pool-size`를 100이나 200으로 설정하는 것이다. 이는 오히려 DB 서버를 압도하여 성능을 떨어뜨린다.

Percona의 연구와 실무 경험에 따른 공식:

```
적정 풀 사이즈 = (CPU 코어 수 × 2) + 유효 스핀들 수

예: 8코어 DB 서버, SSD (스핀들 없음)
= (8 × 2) + 1 = 17 → 20으로 라운딩
```

단, 이는 단일 애플리케이션 인스턴스 기준이 아니라 DB 서버 전체 기준이다. 앱 서버 10대가 각각 20개씩 연결하면 DB는 200개 커넥션을 받는다.

```
앱 인스턴스당 풀 사이즈 = DB 최적 전체 커넥션 수 / 앱 서버 수

예: DB 최적 커넥션 100, 앱 서버 5대
→ 인스턴스당 최대 풀 사이즈 = 20
```

MySQL에서 현재 커넥션 수 확인:

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';  -- 실제 쿼리 실행 중인 스레드

-- 커넥션별 상태 상세
SELECT
  USER,
  HOST,
  DB,
  COMMAND,
  TIME,
  STATE,
  INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
ORDER BY TIME DESC;
```

### 3.4 커넥션 풀 고갈(Pool Exhaustion) 진단

증상: 애플리케이션 로그에 `HikariPool-1 - Connection is not available, request timed out after 3000ms` 에러가 쏟아지기 시작한다.

**진단 Step 1: 현재 풀 상태 확인**

HikariCP JMX 또는 Micrometer 메트릭 활성화:

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics
```

```bash
# Actuator로 풀 상태 확인
curl http://localhost:8080/actuator/metrics/hikaricp.connections
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending
```

**진단 Step 2: 슬로우 쿼리 또는 트랜잭션 특정**

```sql
-- 오래 실행 중인 쿼리 확인
SELECT * FROM information_schema.PROCESSLIST
WHERE COMMAND = 'Query' AND TIME > 5
ORDER BY TIME DESC;

-- InnoDB 트랜잭션 확인 (락 대기 포함)
SELECT
  trx_id,
  trx_started,
  trx_mysql_thread_id,
  trx_query,
  trx_lock_structs,
  trx_rows_locked
FROM information_schema.INNODB_TRX
ORDER BY trx_started ASC;
```

**진단 Step 3: 커넥션 누수 확인**

HikariCP에서 누수 감지 설정:

```yaml
hikari:
  leak-detection-threshold: 5000  # 5초 이상 반납 안 된 커넥션 경고
```

이 설정을 켜면 커넥션이 오래 유지될 때 스택 트레이스와 함께 경고 로그가 출력된다. 누수가 발생하는 코드를 정확히 찾을 수 있다.

**흔한 누수 패턴:**

```java
// 잘못된 코드 — Connection을 finally에서 닫지 않음
public void badExample() throws SQLException {
    Connection conn = dataSource.getConnection();
    PreparedStatement ps = conn.prepareStatement("SELECT ...");
    ResultSet rs = ps.executeQuery();
    // 예외 발생 시 conn이 반납되지 않는다!
    while (rs.next()) { ... }
    conn.close();  // 예외 발생하면 이 줄에 못 옴
}

// 올바른 코드 — try-with-resources 사용
public void goodExample() throws SQLException {
    try (Connection conn = dataSource.getConnection();
         PreparedStatement ps = conn.prepareStatement("SELECT ...");
         ResultSet rs = ps.executeQuery()) {
        while (rs.next()) { ... }
    }  // 자동으로 닫힘
}
```

### 3.5 max_lifetime과 DB wait_timeout 관계

MySQL의 `wait_timeout`은 유휴 커넥션을 서버 측에서 끊는 시간이다. HikariCP의 `max-lifetime`이 이 값보다 길면 DB가 이미 끊은 커넥션을 HikariCP가 살아있다고 착각해서 애플리케이션에 돌려줄 수 있다.

```sql
-- MySQL 기본 wait_timeout 확인
SHOW VARIABLES LIKE 'wait_timeout';  -- 보통 28800 (8시간)
```

안전한 설정:

```
max-lifetime < wait_timeout - 30초 (여유분)

예: wait_timeout=28800s(8h) → max-lifetime=1800000ms(30분)
```

---

## 4. 무중단 스키마 마이그레이션

### 4.1 왜 일반 ALTER TABLE이 위험한가

```sql
-- 이 쿼리가 프로덕션에서 얼마나 위험한지 알고 있는가?
ALTER TABLE orders ADD COLUMN discount_amount DECIMAL(10,2) DEFAULT 0;
```

MySQL은 기본적으로 DDL을 실행할 때 테이블에 MDL(Metadata Lock)을 건다. 수천만 행이 있는 테이블이라면:

1. 테이블 전체를 임시 테이블로 복사
2. 새 컬럼 추가
3. 원본과 교체

이 과정에서 테이블 락이 걸려 INSERT/UPDATE가 줄줄이 대기한다. 5분짜리 마이그레이션이 서비스 장애가 되는 순간이다.

MySQL 5.6+에서 `ALGORITHM=INPLACE`를 지원하는 일부 DDL은 락 없이 동작하지만, 모든 DDL이 그런 것은 아니다.

```sql
-- 어떤 알고리즘 지원하는지 확인 (실제 실행 안 함)
ALTER TABLE orders ADD INDEX idx_status (status), ALGORITHM=INPLACE, LOCK=NONE;
-- 에러가 나면 INPLACE 불가
```

### 4.2 gh-ost: GitHub의 온라인 스키마 변경 도구

gh-ost는 트리거 없이 binlog를 직접 파싱해서 온라인으로 스키마를 변경한다. 기존 pt-online-schema-change가 트리거 방식을 사용해 발생하던 오버헤드와 엣지 케이스를 해결한다.

**작동 원리:**

```
1. 고스트 테이블(ghost table) 생성: _orders_gho
2. 원본 테이블에서 배치 복사 (기존 데이터)
3. binlog에서 실시간 변경 이벤트 수신 및 고스트 테이블에 적용
4. 원본과 고스트 테이블이 동기화되면 테이블 이름 교체 (atomic rename)

orders → _orders_del (삭제 예정)
_orders_gho → orders (새 테이블)
```

**실제 실행:**

```bash
gh-ost \
  --user="root" \
  --password="pass" \
  --host="db.example.com" \
  --database="myapp" \
  --table="orders" \
  --alter="ADD COLUMN discount_amount DECIMAL(10,2) NOT NULL DEFAULT 0" \
  --allow-on-master \
  --execute \
  --max-load="Threads_running=25" \
  --critical-load="Threads_running=50" \
  --chunk-size=500 \
  --throttle-control-replicas="replica1.example.com" \
  --max-lag-millis=1500 \
  --initially-drop-ghost-table \
  --initially-drop-old-table \
  --timestamp-old-table \
  --postpone-cut-over-flag-file=/tmp/ghost.postpone.flag \
  2>&1 | tee /var/log/gh-ost-migration.log
```

주요 파라미터 설명:

- `--max-load`: Threads_running이 25를 넘으면 자동 스로틀링
- `--critical-load`: 50을 넘으면 즉시 중단
- `--max-lag-millis`: 복제 지연이 1.5초를 넘으면 스로틀링
- `--postpone-cut-over-flag-file`: 해당 파일이 존재하면 컷오버(이름 교체) 연기 — 수동으로 파일 삭제해서 타이밍 제어

**마이그레이션 중 상태 모니터링:**

```bash
# gh-ost 소켓을 통한 실시간 상태 확인
echo "status" | nc -U /tmp/gh-ost.myapp.orders.sock

# 스로틀 직접 제어
echo "throttle" | nc -U /tmp/gh-ost.myapp.orders.sock
echo "no-throttle" | nc -U /tmp/gh-ost.myapp.orders.sock

# 컷오버 준비 완료 후 플래그 파일 삭제로 진행
rm /tmp/ghost.postpone.flag
```

### 4.3 pt-online-schema-change

Percona Toolkit의 pt-osc는 오래되고 안정적인 도구다. 트리거 방식이라 gh-ost와는 다른 트레이드오프가 있다.

```bash
pt-online-schema-change \
  --user=root \
  --password=pass \
  --host=db.example.com \
  D=myapp,t=orders \
  --alter="ADD COLUMN discount_amount DECIMAL(10,2) NOT NULL DEFAULT 0" \
  --max-load="Threads_running:25" \
  --critical-load="Threads_running:50" \
  --chunk-size=1000 \
  --sleep=0.1 \
  --check-replication-filters \
  --check-slave-lag \
  --max-lag=2 \
  --print \
  --execute
```

pt-osc는 트리거를 생성해서 원본 테이블의 변경을 복사 테이블에 반영한다. 이미 트리거가 있는 테이블에는 사용할 수 없고, 트리거 자체의 오버헤드가 있다. gh-ost를 사용할 수 없는 환경(RDS 등 binlog 접근 제한)에서 대안이 된다.

### 4.4 단계적 마이그레이션 패턴

컬럼을 NOT NULL로 추가하거나 컬럼 타입을 변경하는 것은 한 번에 하면 위험하다. 여러 배포 단계로 나눠야 한다.

**예시: `user_id INT` → `user_uuid VARCHAR(36)` 전환**

```
Phase 1 (배포 A): 새 컬럼 추가 (nullable)
  ALTER TABLE orders ADD COLUMN user_uuid VARCHAR(36);

Phase 2 (배포 B): 코드에서 두 컬럼 모두 쓰기
  INSERT INTO orders (user_id, user_uuid, ...) VALUES (123, 'abc-123', ...)

Phase 3 (배치 작업): 기존 데이터 마이그레이션
  UPDATE orders SET user_uuid = (SELECT uuid FROM users WHERE id = user_id)
  WHERE user_uuid IS NULL LIMIT 1000;

Phase 4 (배포 C): 코드에서 user_uuid만 읽기
  SELECT * FROM orders WHERE user_uuid = 'abc-123'

Phase 5 (배포 D): user_id 읽기 제거, user_uuid에 NOT NULL 추가
  ALTER TABLE orders MODIFY COLUMN user_uuid VARCHAR(36) NOT NULL;
  ALTER TABLE orders DROP COLUMN user_id;
```

각 단계 사이에 검증과 롤백 계획이 있어야 한다. 성급하게 합치면 한 단계 실패 시 전체를 롤백해야 한다.

---

## 5. 모니터링 지표

### 5.1 핵심 지표 세 가지

DB 모니터링에서 항상 눈여겨봐야 할 지표는 크게 세 가지다: **QPS(Queries Per Second)**, **슬로우 쿼리 비율**, **버퍼풀 히트율**.

### 5.2 QPS 모니터링

```sql
-- 현재 누적 쿼리 수 조회
SHOW GLOBAL STATUS LIKE 'Queries';
SHOW GLOBAL STATUS LIKE 'Questions';

-- 1초 간격으로 QPS 계산
SHOW GLOBAL STATUS LIKE 'Com_select';
SHOW GLOBAL STATUS LIKE 'Com_insert';
SHOW GLOBAL STATUS LIKE 'Com_update';
SHOW GLOBAL STATUS LIKE 'Com_delete';
```

실시간 QPS는 Prometheus + mysqld_exporter로 수집하면 편하다:

```yaml
# prometheus.yml 스크레이프 설정
scrape_configs:
  - job_name: 'mysql'
    static_configs:
      - targets: ['db-exporter:9104']
    scrape_interval: 15s
```

Grafana 대시보드에서 볼 PromQL:

```
# QPS (5분 평균)
rate(mysql_global_status_questions[5m])

# 읽기 vs 쓰기 비율
rate(mysql_global_status_commands_total{command="select"}[5m])
  / rate(mysql_global_status_questions[5m])
```

QPS 급증은 여러 원인을 가진다:

- 정상적인 트래픽 증가
- N+1 쿼리 문제 (ORM에서 자주 발생)
- 캐시 장애로 인한 DB 직접 요청 폭증
- 배치 잡 실행

각 원인에 따라 대응이 다르므로 QPS만 보지 말고 동시에 슬로우 쿼리와 커넥션 수도 함께 봐야 한다.

### 5.3 슬로우 쿼리 모니터링

```ini
# 슬로우 쿼리 로그 활성화
[mysqld]
slow_query_log       = ON
slow_query_log_file  = /var/log/mysql/slow.log
long_query_time      = 1      # 1초 이상인 쿼리 기록
log_queries_not_using_indexes = ON  # 인덱스 미사용 쿼리도 기록
min_examined_row_limit = 100  # 100행 이상 검사한 쿼리만 (노이즈 줄이기)
```

`pt-query-digest`로 슬로우 쿼리 분석:

```bash
# 슬로우 쿼리 로그 분석 — 상위 10개 문제 쿼리
pt-query-digest /var/log/mysql/slow.log \
  --limit=10 \
  --report-format=query_report \
  2>/dev/null

# 시간 범위 필터링
pt-query-digest /var/log/mysql/slow.log \
  --since="2024-03-18 09:00:00" \
  --until="2024-03-18 10:00:00"
```

출력 예시 해석:

```
# Query 1: 45 QPS, 23x exec, 120.23s total
# avg: 5.22s, max: 45.11s, 95th: 12.34s
# Rows examined: 500k (avg), Rows sent: 1 (avg)
SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC;
```

`Rows examined: 500k, Rows sent: 1`은 인덱스 없이 50만 행을 다 읽어서 1개를 반환하고 있다는 뜻이다. `status`와 `created_at`에 복합 인덱스가 필요하다.

```sql
-- 실행 계획 확인
EXPLAIN SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC;

-- 인덱스 추가 (gh-ost 사용)
gh-ost ... --alter="ADD INDEX idx_status_created (status, created_at)"
```

Performance Schema를 활용한 실시간 슬로우 쿼리 확인:

```sql
-- 평균 실행 시간 기준 상위 쿼리
SELECT
  DIGEST_TEXT,
  COUNT_STAR,
  AVG_TIMER_WAIT / 1e9 AS avg_ms,
  MAX_TIMER_WAIT / 1e9 AS max_ms,
  SUM_ROWS_EXAMINED / COUNT_STAR AS avg_rows_examined
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'myapp'
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 10;
```

### 5.4 InnoDB 버퍼풀 히트율

InnoDB 버퍼풀은 디스크 I/O를 줄이기 위한 메모리 캐시다. 히트율이 낮으면 쿼리마다 디스크에서 데이터를 읽어야 해서 성능이 급락한다.

```sql
-- 버퍼풀 히트율 계산
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';      -- 디스크에서 읽은 횟수
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';  -- 전체 읽기 요청

-- 히트율 = 1 - (reads / read_requests)
SELECT
  (1 - (
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads')
    /
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
  )) * 100 AS buffer_pool_hit_rate;
```

일반적으로 99% 이상이어야 정상이다. 95% 미만이면 심각하다.

버퍼풀 히트율이 낮을 때 대응:

```ini
# 버퍼풀 크기 증가 — 일반적으로 서버 메모리의 70-80%
[mysqld]
innodb_buffer_pool_size = 8G  # 서버 RAM 12GB일 때

# 동적 변경 (MySQL 5.7+)
SET GLOBAL innodb_buffer_pool_size = 8589934592;  # 8GB in bytes
```

버퍼풀 크기를 늘려도 히트율이 낮다면, 데이터 전체가 메모리에 올라오지 않을 만큼 데이터셋이 크다는 뜻이다. 이 경우:

1. 쿼리 최적화로 읽는 데이터 양 줄이기
2. 파티셔닝으로 자주 사용하는 데이터만 핫 파티션으로
3. 캐시 레이어(Redis) 도입으로 DB 히트 자체 줄이기

### 5.5 종합 모니터링 대시보드 구성

```
MySQL 운영 대시보드 레이아웃:

┌─────────────────────────────────────────────────────┐
│  QPS (5m avg)  │  Slow Query %  │  Connections      │
│    1,234/s     │    0.12%       │  85/200 (42%)     │
├─────────────────────────────────────────────────────┤
│  Buffer Pool Hit Rate  │  Replication Lag            │
│        99.8%           │  replica-1: 0.3s            │
│                        │  replica-2: 0.8s            │
├─────────────────────────────────────────────────────┤
│  InnoDB Row Operations (5m)                         │
│  Read: 45k/s  Insert: 1.2k/s  Update: 0.8k/s       │
├─────────────────────────────────────────────────────┤
│  Table Lock Waits  │  Deadlocks (1h)  │  Disk I/O   │
│      0             │      0           │  45 MB/s    │
└─────────────────────────────────────────────────────┘
```

알림 임계값 설정 가이드:

| 지표 | 경고 | 위험 |
|------|------|------|
| 슬로우 쿼리 비율 | > 1% | > 5% |
| 버퍼풀 히트율 | < 99% | < 95% |
| 복제 지연 | > 5초 | > 30초 |
| 커넥션 사용률 | > 70% | > 90% |
| QPS 급증 | 평소 대비 200% | 평소 대비 500% |

### 5.6 장애 상황별 진단 체크리스트

**증상: 응답 속도 갑자기 느려짐**

```bash
# 1. 실행 중인 쿼리 확인
mysql -e "SHOW PROCESSLIST\G" | grep -v Sleep

# 2. 락 대기 확인
mysql -e "SELECT * FROM sys.innodb_lock_waits\G"

# 3. 슬로우 쿼리 실시간 확인
tail -f /var/log/mysql/slow.log

# 4. 시스템 리소스 확인
iostat -x 1
vmstat 1
```

**증상: 커넥션 풀 고갈**

```bash
# 1. MySQL 커넥션 현황
mysql -e "SHOW STATUS LIKE 'Threads_%';"

# 2. 장시간 쿼리 킬
mysql -e "SELECT id, user, host, time, info FROM information_schema.PROCESSLIST WHERE time > 30 AND command != 'Sleep';"
# 필요시: KILL QUERY [id];

# 3. HikariCP 메트릭 확인
curl http://app-server:8080/actuator/metrics/hikaricp.connections.active
```

---

## 마무리: 프로덕션 DB 운영의 철칙

프로덕션 DB를 운영하면서 얻은 교훈을 몇 가지로 압축한다.

**첫째, 복구 훈련 없는 백업은 없다.** 백업 파일이 있다고 복구가 된다는 보장은 없다. 분기에 한 번이라도 실제 복구 시뮬레이션을 하라.

**둘째, DDL은 반드시 gh-ost나 pt-osc를 통해.** 테이블 크기가 100만 행을 넘으면 절대로 직접 ALTER TABLE을 날리지 마라. 서비스 시간에는 더욱.

**셋째, 커넥션 풀 사이즈는 작게 시작해서 늘려라.** 처음부터 크게 잡으면 문제가 생겼을 때 원인을 찾기 어렵다.

**넷째, 모니터링 알림은 졸음을 깨울 수 있을 때만 보내라.** 알림이 너무 많으면 무감각해진다. 위험 임계값 알림만 PagerDuty로, 경고는 Slack으로 분리하라.

**다섯째, 복제 지연에 대비한 애플리케이션 설계를 하라.** "방금 쓴 데이터를 바로 읽어야 한다"는 요구사항은 마스터로 라우팅하거나, 레디스 캐시를 통해 해결하라. 레플리카 읽기는 지연이 있다는 것을 항상 전제하라.

---

## 참고 자료

1. **"High Performance MySQL, 4th Edition"** — Silvia Botros, Jeremy Tinley (O'Reilly, 2022) — MySQL 내부 구조와 운영 전략의 바이블
2. **"Database Reliability Engineering"** — Laine Campbell, Charity Majors (O'Reilly, 2017) — SRE 관점에서 바라보는 데이터베이스 운영
3. **Percona Blog** — https://www.percona.com/blog — XtraBackup, pt-toolkit, MySQL 운영 심화 자료
4. **gh-ost 공식 문서** — https://github.com/github/gh-ost — GitHub 엔지니어링팀의 온라인 스키마 변경 도구 상세 가이드
5. **MySQL 8.0 Reference Manual: Replication** — https://dev.mysql.com/doc/refman/8.0/en/replication.html — binlog, GTID, 병렬 복제 공식 레퍼런스
6. **"Designing Data-Intensive Applications"** — Martin Kleppmann (O'Reilly, 2017) — 복제, 파티셔닝, 분산 시스템 이론의 필독서

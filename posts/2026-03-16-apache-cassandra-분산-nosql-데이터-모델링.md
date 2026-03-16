# Apache Cassandra 분산 NoSQL 데이터 모델링

## 개요

Apache Cassandra는 Facebook이 개발하고 현재 Apache Software Foundation에서 관리하는 분산 NoSQL 데이터베이스입니다. 선형 확장성(Linear Scalability), 고가용성(High Availability), 그리고 지역 분산(Geo-distribution) 측면에서 뛰어난 성능을 자랑하며, Netflix, Apple, Instagram 등 대규모 트래픽을 처리하는 기업들이 실제 프로덕션에서 사용하고 있습니다.

하지만 Cassandra를 도입하면서 가장 많이 실패하는 지점은 기술 자체가 아닌 **데이터 모델링**입니다. RDBMS의 정규화 중심 사고를 그대로 가져오면 심각한 성능 저하나 운영 장애로 이어질 수 있습니다. 이 글에서는 Cassandra의 핵심 데이터 모델링 원칙과 실전에서 바로 적용 가능한 패턴을 소개합니다.

---

## 핵심 개념

### 1. Cassandra의 데이터 구조

Cassandra의 데이터 모델은 다음 계층으로 구성됩니다.

```
Keyspace → Table → Partition → Row → Column
```

- **Keyspace**: RDBMS의 데이터베이스에 해당. 복제 전략(Replication Strategy)이 정의됨
- **Partition Key**: 데이터가 어느 노드에 저장될지 결정하는 핵심 요소
- **Clustering Column**: 파티션 내 데이터의 정렬 순서를 결정
- **Primary Key**: `(Partition Key, Clustering Column)` 조합

### 2. 쿼리 주도 설계 (Query-Driven Design)

Cassandra 모델링의 가장 중요한 원칙입니다. RDBMS처럼 엔티티 중심으로 테이블을 설계하고 JOIN으로 데이터를 합치는 방식이 아니라, **애플리케이션의 쿼리 패턴을 먼저 정의하고, 그에 맞는 테이블을 설계**합니다.

> "One query, one table" 원칙을 기억하세요.

### 3. 파티셔닝과 토큰 링

Cassandra는 Consistent Hashing을 사용합니다. Partition Key를 해시하여 토큰 링의 특정 위치에 데이터를 배치합니다. 이 때문에 Partition Key 선택이 **데이터 분산의 균등성**에 직접적인 영향을 미칩니다.

```
[Node A: 0~25%] → [Node B: 25~50%] → [Node C: 50~75%] → [Node D: 75~100%]
```

### 4. CAP 이론에서의 Cassandra

Cassandra는 **AP(Availability + Partition Tolerance)** 시스템입니다. Consistency Level을 조절하여 일관성을 높일 수 있지만, 기본적으로 Eventually Consistent 특성을 가집니다.

```
Consistency Level:
- ONE: 가장 빠름, 낮은 일관성
- QUORUM: (RF/2 + 1) 노드 응답, 균형점
- ALL: 가장 강한 일관성, 가용성 희생
```

---

## 실전 예제

### 시나리오: 타임라인 기반 이벤트 로그 시스템

사용자 활동 로그를 저장하고, 특정 사용자의 최근 활동을 시간 순으로 조회하는 시스템을 설계합니다.

#### CQL 스키마 설계

```sql
-- Keyspace 생성
CREATE KEYSPACE activity_log
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3
}
AND durable_writes = true;

USE activity_log;

-- 사용자 활동 로그 테이블
-- 쿼리 패턴: user_id로 조회, 최신순 정렬
CREATE TABLE user_activity_by_user (
    user_id     UUID,
    event_time  TIMESTAMP,
    event_id    UUID,
    event_type  TEXT,
    metadata    MAP<TEXT, TEXT>,
    PRIMARY KEY ((user_id), event_time, event_id)
) WITH CLUSTERING ORDER BY (event_time DESC, event_id DESC)
  AND default_time_to_live = 2592000  -- 30일 TTL
  AND compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'DAYS',
    'compaction_window_size': 1
  };

-- 이벤트 타입별 조회 테이블 (별도 쿼리 패턴 지원)
CREATE TABLE user_activity_by_type (
    user_id     UUID,
    event_type  TEXT,
    event_time  TIMESTAMP,
    event_id    UUID,
    metadata    MAP<TEXT, TEXT>,
    PRIMARY KEY ((user_id, event_type), event_time, event_id)
) WITH CLUSTERING ORDER BY (event_time DESC, event_id ASC)
  AND default_time_to_live = 2592000;
```

#### Java + Spring Data Cassandra 연동

```java
// build.gradle
// implementation 'org.springframework.boot:spring-boot-starter-data-cassandra'

@Table("user_activity_by_user")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserActivity {

    @PrimaryKeyColumn(name = "user_id", ordinal = 0, type = PrimaryKeyType.PARTITIONED)
    private UUID userId;

    @PrimaryKeyColumn(name = "event_time", ordinal = 1, type = PrimaryKeyType.CLUSTERED,
                      ordering = Ordering.DESCENDING)
    private Instant eventTime;

    @PrimaryKeyColumn(name = "event_id", ordinal = 2, type = PrimaryKeyType.CLUSTERED,
                      ordering = Ordering.DESCENDING)
    private UUID eventId;

    @Column("event_type")
    private String eventType;

    @Column("metadata")
    private Map<String, String> metadata;
}
```

```java
@Repository
public interface UserActivityRepository extends CassandraRepository<UserActivity, UserActivityKey> {

    // 특정 사용자의 최근 N개 활동 조회
    @Query("SELECT * FROM user_activity_by_user WHERE user_id = ?0 LIMIT ?1")
    List<UserActivity> findRecentActivities(UUID userId, int limit);

    // 특정 시간 범위 조회
    @Query("SELECT * FROM user_activity_by_user " +
           "WHERE user_id = ?0 AND event_time >= ?1 AND event_time <= ?2")
    List<UserActivity> findActivitiesByTimeRange(UUID userId, Instant from, Instant to);
}
```

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ActivityLogService {

    private final UserActivityRepository activityRepository;
    private final CassandraOperations cassandraOperations;

    public void recordActivity(UUID userId, String eventType, Map<String, String> metadata) {
        UserActivity activity = new UserActivity(
            userId,
            Instant.now(),
            UUID.randomUUID(),
            eventType,
            metadata
        );

        // 두 테이블에 동시 기록 (Batch 사용 시 주의사항은 아래 참조)
        activityRepository.save(activity);

        log.info("Activity recorded: userId={}, eventType={}", userId, eventType);
    }

    public List<UserActivity> getRecentActivities(UUID userId, int limit) {
        // Paging을 통한 대용량 처리
        return activityRepository.findRecentActivities(userId, limit);
    }

    // Lightweight Transaction (LWT) 예시
    public boolean createUniqueUserProfile(UUID userId, String username) {
        String cql = "INSERT INTO user_profiles (user_id, username, created_at) " +
                     "VALUES (?, ?, ?) IF NOT EXISTS";

        ResultSet result = cassandraOperations.getCqlOperations()
            .queryForResultSet(cql, userId, username, Instant.now());

        return result.one().getBool("[applied]");
    }
}
```

### 시나리오: Hot Partition 방지 - Bucket 패턴

글로벌 랭킹처럼 단일 Partition Key에 트래픽이 몰리는 경우, **Bucket 전략**으로 분산시킵니다.

```sql
-- 문제 있는 설계: game_id가 핫 파티션이 됨
CREATE TABLE leaderboard_bad (
    game_id     TEXT,
    score       BIGINT,
    user_id     UUID,
    PRIMARY KEY (game_id, score, user_id)
) WITH CLUSTERING ORDER BY (score DESC);

-- 개선된 설계: bucket으로 파티션 분산
CREATE TABLE leaderboard_bucketed (
    game_id     TEXT,
    bucket      INT,        -- 0~9 랜덤 버킷
    score       BIGINT,
    user_id     UUID,
    username    TEXT,
    PRIMARY KEY ((game_id, bucket), score, user_id)
) WITH CLUSTERING ORDER BY (score DESC);
```

```java
@Service
public class LeaderboardService {

    private static final int BUCKET_COUNT = 10;
    private final CassandraOperations cassandraOperations;

    public void updateScore(String gameId, UUID userId, String username, long score) {
        int bucket = ThreadLocalRandom.current().nextInt(BUCKET_COUNT);

        String cql = "INSERT INTO leaderboard_bucketed " +
                     "(game_id, bucket, score, user_id, username) VALUES (?, ?, ?, ?, ?)";
        cassandraOperations.getCqlOperations().execute(cql,
            gameId, bucket, score, userId, username);
    }

    // 전체 랭킹 조회 시 모든 버킷에서 가져와 머지
    public List<LeaderboardEntry> getTopRankings(String gameId, int topN) {
        List<LeaderboardEntry> allEntries = new ArrayList<>();

        for (int bucket = 0; bucket < BUCKET_COUNT; bucket++) {
            String cql = "SELECT * FROM leaderboard_bucketed " +
                         "WHERE game_id = ? AND bucket = ? LIMIT ?";
            // 각 버킷에서 조회
            List<LeaderboardEntry> bucketEntries = cassandraOperations.getCqlOperations()
                .query(cql, new LeaderboardEntryMapper(), gameId, bucket, topN);
            allEntries.addAll(bucketEntries);
        }

        // 메모리에서 정렬 후 상위 N개 반환
        return allEntries.stream()
            .sorted(Comparator.comparingLong(LeaderboardEntry::getScore).reversed())
            .limit(topN)
            .collect(Collectors.toList());
    }
}
```

---

## 주의사항 및 트레이드오프

### ⚠️ 1. Partition 크기 제한

단일 파티션이 너무 커지면 GC 압력, 읽기 지연, Compaction 문제가 발생합니다. **파티션 크기는 100MB 이하, Row 수는 수만 건 이하**를 권장합니다.

```sql
-- 날짜 기반 버킷팅으로 파티션 크기 제어
CREATE TABLE events_by_day (
    user_id     UUID,
    day         DATE,         -- 날짜를 파티션 키의 일부로 사용
    event_time  TIMESTAMP,
    event_id    UUID,
    payload     TEXT,
    PRIMARY KEY ((user_id, day), event_time, event_id)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

### ⚠️ 2. Logged Batch는 성능 저하 유발

```java
// ❌ 잘못된 사용: Logged Batch로 다중 파티션 업데이트
// 코디네이터 노드에 부하 집중, 성능 오히려 저하
BatchStatement batch = BatchStatement.newInstance(BatchType.LOGGED);

// ✅ 올바른 사용: 같은 파티션 내 원자성 보장 시에만 사용
// 또는 비동기 병렬 쓰기 사용
List<CompletableFuture<Void>> futures = events.stream()
    .map(event -> CompletableFuture.runAsync(() -> activityRepository.save(event)))
    .collect(Collectors.toList());

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
```

### ⚠️ 3. ALLOW FILTERING 절대 금지

```sql
-- ❌ 절대 사용 금지: Full Scan 발생
SELECT * FROM user_activity_by_user
WHERE event_type = 'PURCHASE' ALLOW FILTERING;

-- ✅ 해당 쿼리 패턴을 위한 별도 테이블 설계
-- user_activity_by_type 테이블을 활용
SELECT * FROM user_activity_by_type
WHERE user_id = ? AND event_type = 'PURCHASE'
AND event_time >= ?;
```

### ⚠️ 4. 카운터 테이블의 특수성

```sql
-- 카운터는 별도 테이블에서만 사용 가능
CREATE TABLE page_view_counts (
    page_id     TEXT,
    view_count  COUNTER,
    PRIMARY KEY (page_id)
);

-- 일반 컬럼과 혼합 불가
UPDATE page_view_counts SET view_count = view_count + 1 WHERE page_id = ?;
```

### ⚠️ 5. Secondary Index의 함정

Cassandra의 Secondary Index는 각 노드에 로컬 인덱스로 저장됩니다. 선택도(Selectivity)가 낮은 컬럼에 사용하면 **모든 노드에 브로드캐스트 쿼리**가 발생하여 심각한 성능 저하로 이어집니다. **Materialized View나 별도 테이블로 대체**하는 것이 좋습니다.

---

## 정리

Cassandra 데이터 모델링의 핵심을 한 줄로 요약하면 다음과 
# Spring Boot Actuator와 Micrometer 모니터링 설정

## 개요

프로덕션 환경에서 애플리케이션의 건강 상태와 성능 지표를 실시간으로 파악하는 것은 안정적인 서비스 운영의 핵심입니다. Spring Boot 생태계에서는 **Actuator**와 **Micrometer**를 조합하여 강력한 관측 가능성(Observability) 인프라를 구축할 수 있습니다.

많은 팀이 단순히 `/health` 엔드포인트만 열어두고 "모니터링 설정 완료"라고 여기는 경우가 있습니다. 하지만 실무에서는 JVM 메모리 누수 감지, DB 커넥션 풀 고갈, HTTP 레이턴시 분포 추적 등 세밀한 메트릭이 장애 대응 시간을 결정짓습니다. 이 글에서는 Spring Boot Actuator의 핵심 엔드포인트 설정부터 Micrometer를 통한 Prometheus 연동, 커스텀 메트릭 등록까지 실무에서 바로 적용 가능한 수준으로 다룹니다.

---

## 핵심 개념

### Spring Boot Actuator

Actuator는 Spring Boot 애플리케이션의 내부 상태를 HTTP 또는 JMX를 통해 노출하는 모듈입니다. 주요 빌트인 엔드포인트는 다음과 같습니다.

| 엔드포인트 | 설명 |
|---|---|
| `/actuator/health` | 애플리케이션 및 의존 시스템 헬스 체크 |
| `/actuator/metrics` | Micrometer 기반 메트릭 목록 |
| `/actuator/info` | 빌드 정보, Git 커밋 등 |
| `/actuator/env` | 환경 변수 및 설정값 |
| `/actuator/loggers` | 런타임 로그 레벨 변경 |
| `/actuator/threaddump` | JVM 스레드 덤프 |
| `/actuator/prometheus` | Prometheus 스크레이핑용 메트릭 |

### Micrometer

Micrometer는 JVM 애플리케이션을 위한 **메트릭 파사드(Facade)** 입니다. SLF4J가 로깅 구현체를 추상화하듯, Micrometer는 Prometheus, Datadog, InfluxDB, CloudWatch 등 다양한 모니터링 시스템을 단일 API로 추상화합니다.

핵심 메트릭 타입:
- **Counter**: 단조 증가 카운터 (요청 수, 에러 수)
- **Gauge**: 현재 값 (활성 세션 수, 큐 크기)
- **Timer**: 이벤트 지속 시간과 빈도 (HTTP 레이턴시)
- **DistributionSummary**: 크기 분포 (요청 페이로드 크기)

---

## 실전 예제

### 1. 의존성 추가

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- Prometheus 연동 -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
</dependencies>
```

### 2. Actuator 기본 설정

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, loggers, threaddump
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized  # 인증된 사용자에게만 상세 정보 노출
      probes:
        enabled: true  # Kubernetes liveness/readiness 프로브 활성화
  metrics:
    tags:
      application: ${spring.application.name}  # 모든 메트릭에 앱 이름 태그 추가
      environment: ${spring.profiles.active:local}
    distribution:
      percentiles-histogram:
        http.server.requests: true  # HTTP 요청 히스토그램 활성화
      percentiles:
        http.server.requests: 0.5, 0.75, 0.95, 0.99
      slo:
        http.server.requests: 100ms, 500ms, 1s  # SLO 버킷 설정
  server:
    port: 8081  # 별도 포트로 분리 (운영 권장)
```

> **실무 팁**: Actuator를 별도 포트(`management.server.port`)로 분리하면 API 게이트웨이나 로드밸런서에서 외부 트래픽과 모니터링 트래픽을 격리할 수 있습니다.

### 3. Security 설정 (Actuator 보호)

```java
@Configuration
@EnableWebSecurity
public class ActuatorSecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(EndpointRequest.to(HealthEndpoint.class)).permitAll()
                .requestMatchers(EndpointRequest.to(PrometheusEndpoint.class))
                    .hasRole("MONITORING")  // Prometheus 서버 전용 계정
                .anyRequest().hasRole("ADMIN")
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

### 4. 커스텀 헬스 인디케이터

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;
    private final String externalApiUrl;

    public ExternalApiHealthIndicator(
            RestTemplate restTemplate,
            @Value("${external.api.health-check-url}") String externalApiUrl) {
        this.restTemplate = restTemplate;
        this.externalApiUrl = externalApiUrl;
    }

    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                externalApiUrl, String.class
            );
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("url", externalApiUrl)
                    .withDetail("status", response.getStatusCode().value())
                    .build();
            }
            return Health.down()
                .withDetail("url", externalApiUrl)
                .withDetail("status", response.getStatusCode().value())
                .build();
        } catch (Exception e) {
            return Health.down(e)
                .withDetail("url", externalApiUrl)
                .build();
        }
    }
}
```

### 5. 커스텀 메트릭 등록

비즈니스 레벨의 메트릭을 추가하는 것이 실무에서 가장 가치 있는 작업 중 하나입니다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MeterRegistry meterRegistry;
    private final OrderRepository orderRepository;

    // Counter: 주문 생성 횟수 추적
    private Counter orderCreatedCounter;
    // Counter: 주문 실패 횟수 추적
    private Counter orderFailedCounter;
    // Timer: 주문 처리 시간 측정
    private Timer orderProcessingTimer;

    @PostConstruct
    public void initMetrics() {
        orderCreatedCounter = Counter.builder("order.created.total")
            .description("Total number of orders created")
            .tag("service", "order")
            .register(meterRegistry);

        orderFailedCounter = Counter.builder("order.failed.total")
            .description("Total number of failed orders")
            .tag("service", "order")
            .register(meterRegistry);

        orderProcessingTimer = Timer.builder("order.processing.duration")
            .description("Order processing duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(meterRegistry);
    }

    public Order createOrder(OrderRequest request) {
        return orderProcessingTimer.record(() -> {
            try {
                Order order = processOrder(request);
                orderCreatedCounter.increment();

                // 결제 수단별 태그를 사용한 세분화 메트릭
                meterRegistry.counter("order.payment.method",
                    "method", request.getPaymentMethod().name()
                ).increment();

                return order;
            } catch (Exception e) {
                orderFailedCounter.increment();
                throw e;
            }
        });
    }

    // Gauge 예시: 현재 처리 대기 중인 주문 수
    @PostConstruct
    public void registerPendingOrdersGauge() {
        Gauge.builder("order.pending.count", orderRepository, repo ->
            repo.countByStatus(OrderStatus.PENDING)
        )
        .description("Number of pending orders")
        .register(meterRegistry);
    }
}
```

### 6. AOP 기반 메트릭 자동화

반복적인 메트릭 코드를 AOP로 추상화할 수 있습니다.

```java
@Aspect
@Component
@RequiredArgsConstructor
public class MetricsAspect {

    private final MeterRegistry meterRegistry;

    @Around("@annotation(timed)")
    public Object measureExecutionTime(ProceedingJoinPoint pjp,
                                        io.micrometer.core.annotation.Timed timed) throws Throwable {
        String metricName = timed.value().isEmpty()
            ? pjp.getSignature().getDeclaringTypeName() + "." + pjp.getSignature().getName()
            : timed.value();

        Timer.Sample sample = Timer.start(meterRegistry);
        String exceptionClass = "none";

        try {
            return pjp.proceed();
        } catch (Exception e) {
            exceptionClass = e.getClass().getSimpleName();
            throw e;
        } finally {
            sample.stop(Timer.builder(metricName)
                .tag("exception", exceptionClass)
                .tag("class", pjp.getSignature().getDeclaringType().getSimpleName())
                .register(meterRegistry));
        }
    }
}
```

### 7. Prometheus + Grafana 연동 구성

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8081']
    basic_auth:
      username: 'monitoring'
      password: 'secret'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
```

유용한 Prometheus 쿼리(PromQL) 예시:

```promql
# HTTP 요청 에러율 (5분 평균)
rate(http_server_requests_seconds_count{status=~"5.."}[5m])
/ rate(http_server_requests_seconds_count[5m])

# 99th percentile 레이턴시
histogram_quantile(0.99,
  rate(http_server_requests_seconds_bucket[5m])
)

# JVM 힙 사용률
jvm_memory_used_bytes{area="heap"}
/ jvm_memory_max_bytes{area="heap"}
```

---

## 주의사항 및 트레이드오프

### 카디널리티 폭발 문제

메트릭 태그 설계 시 **절대로** 고유값(User ID, 요청 UUID, IP 주소 등)을 태그로 사용하면 안 됩니다. 이는 시계열 데이터베이스에서 **카디널리티 폭발(Cardinality Explosion)** 을 일으켜 메모리 고갈과 시스템 장애를 초래합니다.

```java
// ❌ 절대 하지 말 것
meterRegistry.counter("api.request",
    "userId", userId,        // 수백만 개의 시계열 생성!
    "requestId", requestId   // 더욱 위험!
).increment();

// ✅ 올바른 방법: 유한한 값만 태그로 사용
meterRegistry.counter("api.request",
    "userTier", user.getTier().name(),  // FREE, PREMIUM, ENTERPRISE
    "endpoint", "/api/orders"
).increment();
```

### `/actuator/env` 노출 위험

`env` 엔드포인트는 데이터베이스 패스워드, API 키 등 민감 정보를 노출할 수 있습니다. 기본적으로 비활성화하고, 필요 시 마스킹 처리를 확인하세요.

```yaml
management:
  endpoint:
    env:
      enabled: false  # 프로덕션에서 비활성화 권장
      show-values: never  # Spring Boot 3.x 이상: 값 마스킹
```

### 메트릭 수집 오버헤드

`publishPercentileHistogram: true` 설정은 메모리와 CPU를 추가로 소비합니다. 모든 메트릭에 적용하기보다 SLA가 있는 핵심 엔드포인트에 한정하세요. `percentiles`(클라이언트 사이드 계산)와 `percentiles-histogram`(서버 사이드 집계 가능)의 차이를 이해하고 선택해야 합니다.

### Kubernetes 환경에서의 헬스 프로브

```yaml
# Kubernetes Deployment
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8081
  initialDelaySeconds: 60
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8081
  initialDelaySeconds: 30
  periodSeconds: 5
```

Liveness와 Readiness를 분리하는 것이 중요합니다. Readiness가 실패하면 트래픽 수신을 중단하고, Liveness가 실패
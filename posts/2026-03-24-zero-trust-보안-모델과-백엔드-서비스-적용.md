# Zero Trust 보안 모델과 백엔드 서비스 적용

## 개요

"절대 신뢰하지 말고, 항상 검증하라(Never Trust, Always Verify)."

Zero Trust는 단순한 보안 제품이나 솔루션이 아니라 **보안 철학이자 아키텍처 패러다임**입니다. 기존의 경계 기반 보안(Perimeter Security) 모델은 방화벽 안쪽은 안전하다는 가정 하에 설계되었습니다. 하지만 클라우드 네이티브 환경, 원격 근무 확산, 마이크로서비스 아키텍처가 일반화되면서 이 전통적인 모델은 근본적인 한계를 드러냈습니다.

2020년 NIST SP 800-207을 기점으로 Zero Trust Architecture(ZTA)가 표준으로 정의되었고, 구글의 BeyondCorp, 넷플릭스의 SPIFFE/SPIRE 적용 사례가 알려지면서 엔터프라이즈 전반으로 빠르게 확산되고 있습니다.

이 글에서는 Zero Trust의 핵심 원칙을 이해하고, Spring Boot 기반의 백엔드 서비스에 실질적으로 적용하는 방법을 코드와 함께 살펴봅니다.

---

## 핵심 개념

### 1. Zero Trust의 7가지 원칙 (NIST 기반)

| 원칙 | 설명 |
|------|------|
| **모든 리소스를 명시적으로 검증** | 위치(IP)가 아닌 ID와 컨텍스트 기반 인증 |
| **최소 권한 원칙(Least Privilege)** | 필요한 권한만, 필요한 시간만 부여 |
| **침해 가정(Assume Breach)** | 내부 네트워크도 이미 침해되었다고 가정 |
| **마이크로 세그멘테이션** | 서비스 간 통신도 세밀하게 제어 |
| **지속적인 모니터링** | 실시간 이상 감지 및 감사 로그 |
| **동적 정책 적용** | 요청마다 컨텍스트(시간, 위치, 디바이스)를 반영 |
| **데이터 중심 보안** | 네트워크가 아닌 데이터 자체를 보호 |

### 2. 백엔드 서비스에서 Zero Trust 구현 포인트

백엔드 개발자 관점에서 Zero Trust는 주로 다음 세 가지 영역에서 구현됩니다.

- **서비스 간 인증(mTLS, SPIFFE)**: 서비스 A가 서비스 B를 호출할 때 양방향 인증
- **토큰 기반 인가(JWT, OAuth 2.0)**: 사용자 및 서비스 요청마다 세분화된 권한 검증
- **컨텍스트 인식 접근 제어(ABAC)**: 역할뿐 아니라 시간, 디바이스, 위치 등을 고려

---

## 실전 예제

### 예제 1: Spring Security + JWT 기반 Stateless 인증

기존 세션 기반 인증은 서버 상태에 의존하므로 Zero Trust 원칙에 맞지 않습니다. 모든 요청마다 토큰을 검증하는 방식으로 전환합니다.

```java
// JwtAuthenticationFilter.java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final RequestContextEvaluator contextEvaluator;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null && jwtTokenProvider.validateToken(token)) {
            // 토큰 유효성만 아니라 요청 컨텍스트도 함께 검증
            AccessContext context = contextEvaluator.evaluate(request, token);

            if (!context.isAllowed()) {
                response.sendError(HttpServletResponse.SC_FORBIDDEN,
                    "Context validation failed: " + context.getDenyReason());
                return;
            }

            Authentication auth = jwtTokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearer = request.getHeader("Authorization");
        if (StringUtils.hasText(bearer) && bearer.startsWith("Bearer ")) {
            return bearer.substring(7);
        }
        return null;
    }
}
```

```java
// RequestContextEvaluator.java - 컨텍스트 인식 접근 제어 핵심 로직
@Component
public class RequestContextEvaluator {

    private static final List<String> ALLOWED_COUNTRIES = List.of("KR", "US", "JP");
    private static final LocalTime BUSINESS_START = LocalTime.of(0, 0);
    private static final LocalTime BUSINESS_END = LocalTime.of(23, 59);

    public AccessContext evaluate(HttpServletRequest request, String token) {
        String clientIp = getClientIp(request);
        String userAgent = request.getHeader("User-Agent");
        String country = resolveCountry(clientIp); // GeoIP 라이브러리 활용

        // 1. 지역 기반 제어
        if (!ALLOWED_COUNTRIES.contains(country)) {
            return AccessContext.deny("Unauthorized country: " + country);
        }

        // 2. 비정상 User-Agent 탐지
        if (isSuspiciousAgent(userAgent)) {
            return AccessContext.deny("Suspicious user-agent detected");
        }

        // 3. 토큰 발급 IP와 요청 IP 비교 (선택적 강화)
        String tokenIp = JwtUtils.extractClaim(token, "client_ip");
        if (tokenIp != null && !tokenIp.equals(clientIp)) {
            return AccessContext.deny("IP mismatch detected - possible token theft");
        }

        return AccessContext.allow();
    }

    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (StringUtils.hasText(xForwardedFor)) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }

    private boolean isSuspiciousAgent(String userAgent) {
        if (!StringUtils.hasText(userAgent)) return true;
        List<String> suspiciousPatterns = List.of("sqlmap", "nikto", "scanner");
        return suspiciousPatterns.stream()
            .anyMatch(pattern -> userAgent.toLowerCase().contains(pattern));
    }
}
```

### 예제 2: 서비스 간 mTLS 통신 (RestTemplate 설정)

마이크로서비스 환경에서 서비스 간 호출 시 mTLS를 적용해 양방향 인증을 강제합니다.

```java
// MtlsRestTemplateConfig.java
@Configuration
public class MtlsRestTemplateConfig {

    @Value("${mtls.keystore.path}")
    private String keystorePath;

    @Value("${mtls.keystore.password}")
    private String keystorePassword;

    @Value("${mtls.truststore.path}")
    private String truststorePath;

    @Value("${mtls.truststore.password}")
    private String truststorePassword;

    @Bean
    public RestTemplate mtlsRestTemplate() throws Exception {
        SSLContext sslContext = buildSslContext();

        SSLConnectionSocketFactory socketFactory =
            new SSLConnectionSocketFactory(sslContext,
                new String[]{"TLSv1.3"},  // TLS 1.3만 허용
                null,
                SSLConnectionSocketFactory.getDefaultHostnameVerifier());

        CloseableHttpClient httpClient = HttpClients.custom()
            .setSSLSocketFactory(socketFactory)
            .build();

        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(httpClient);
        factory.setConnectTimeout(3000);
        factory.setReadTimeout(5000);

        return new RestTemplate(factory);
    }

    private SSLContext buildSslContext() throws Exception {
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (InputStream ks = new FileInputStream(keystorePath)) {
            keyStore.load(ks, keystorePassword.toCharArray());
        }

        KeyStore trustStore = KeyStore.getInstance("PKCS12");
        try (InputStream ts = new FileInputStream(truststorePath)) {
            trustStore.load(ts, truststorePassword.toCharArray());
        }

        return SSLContextBuilder.create()
            .loadKeyMaterial(keyStore, keystorePassword.toCharArray())
            .loadTrustMaterial(trustStore, null)
            .build();
    }
}
```

### 예제 3: ABAC(Attribute-Based Access Control) 정책 엔진

RBAC(역할 기반)보다 세밀한 제어가 필요할 때 ABAC를 구현합니다.

```java
// PolicyEngine.java
@Component
public class PolicyEngine {

    public boolean evaluate(PolicyRequest policyRequest) {
        List<Policy> matchedPolicies = findMatchingPolicies(policyRequest);

        if (matchedPolicies.isEmpty()) {
            // Deny by default - Zero Trust의 핵심
            return false;
        }

        return matchedPolicies.stream()
            .allMatch(policy -> policy.evaluate(policyRequest));
    }

    private List<Policy> findMatchingPolicies(PolicyRequest request) {
        return PolicyRegistry.getPolicies().stream()
            .filter(p -> p.matches(request.getResource(), request.getAction()))
            .collect(Collectors.toList());
    }
}

// Policy.java - 정책 정의 예시
@Component
public class DataExportPolicy implements Policy {

    @Override
    public boolean matches(String resource, String action) {
        return resource.startsWith("/api/export") && "WRITE".equals(action);
    }

    @Override
    public boolean evaluate(PolicyRequest request) {
        Map<String, Object> attributes = request.getSubjectAttributes();

        // 1. 관리자 역할 필수
        boolean isAdmin = "ROLE_ADMIN".equals(attributes.get("role"));
        // 2. MFA 인증 완료 여부
        boolean mfaVerified = Boolean.TRUE.equals(attributes.get("mfa_verified"));
        // 3. 업무 시간대 체크 (KST 09:00 ~ 18:00)
        boolean businessHours = isBusinessHours();
        // 4. 승인된 디바이스 여부
        boolean trustedDevice = Boolean.TRUE.equals(attributes.get("device_trusted"));

        return isAdmin && mfaVerified && businessHours && trustedDevice;
    }

    private boolean isBusinessHours() {
        ZonedDateTime now = ZonedDateTime.now(ZoneId.of("Asia/Seoul"));
        int hour = now.getHour();
        return hour >= 9 && hour < 18;
    }
}
```

### 예제 4: 감사 로그(Audit Log) AOP 구현

"지속적인 모니터링"을 위해 모든 민감한 API 접근을 자동으로 기록합니다.

```java
// ZeroTrustAuditAspect.java
@Aspect
@Component
@Slf4j
@RequiredArgsConstructor
public class ZeroTrustAuditAspect {

    private final AuditLogRepository auditLogRepository;
    private final ObjectMapper objectMapper;

    @Around("@annotation(ZeroTrustAudited)")
    public Object auditAccess(ProceedingJoinPoint pjp) throws Throwable {
        HttpServletRequest request = getCurrentRequest();
        String userId = getCurrentUserId();
        String resource = request.getRequestURI();
        long startTime = System.currentTimeMillis();

        AuditLog auditLog = AuditLog.builder()
            .userId(userId)
            .resource(resource)
            .method(request.getMethod())
            .clientIp(getClientIp(request))
            .timestamp(Instant.now())
            .build();

        try {
            Object result = pjp.proceed();
            auditLog.setStatus("SUCCESS");
            auditLog.setDuration(System.currentTimeMillis() - startTime);
            return result;
        } catch (Exception e) {
            auditLog.setStatus("FAILURE");
            auditLog.setErrorMessage(e.getMessage());
            throw e;
        } finally {
            // 비동기로 저장하여 성능 영향 최소화
            auditLogRepository.saveAsync(auditLog);
        }
    }
}

// 사용 예시
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}/sensitive-data")
    @ZeroTrustAudited  // 감사 로그 자동 기록
    @PreAuthorize("hasRole('ADMIN') and #oauth2.hasScope('read:sensitive')")
    public ResponseEntity<SensitiveData> getSensitiveData(@PathVariable Long id) {
        // 비즈니스 로직
        return ResponseEntity.ok(sensitiveDataService.findById(id));
    }
}
```

---

## 주의사항 및 트레이드오프

### ⚠️ 성능 오버헤드

모든 요청마다 토큰 검증, 컨텍스트 평가, 정책 엔진 실행이 발생합니다. 특히 고트래픽 서비스에서는 응답 지연이 누적될 수 있습니다.

**완화 방법:**
- JWT 검증 결과를 단기 캐시(Redis TTL 30초)에 저장
- 정책 엔진 결과를 경량 캐시로 메모이제이션
- GeoIP 조회는 로컬 MaxMind DB 사용 (네트워크 호출 제거)

### ⚠️ mTLS 인증서 관리

서비스가 늘어날수록 인증서 발급, 갱신, 폐기 관리가 복잡해집니다. Cert-Manager(쿠버네티스)나 HashiCorp Vault를 통한 자동화를 반드시 고려해야 합니다.

### ⚠️ 개발자 경험(DX) 저하

로컬 개발 환경에서 mTLS와 ABAC 정책을 모두 활성화하면 개발 속도가 크게 저하됩니다. 환경별 프로파일 분리(`dev` 프로파일에서는 mTLS 우회)가 필요하지만, 이 자체가 보안 허점이 될 수 있습니다.

### ⚠️ 점진적 도입의 중요성

Zero Trust를 한 번에 전환하려 하면 반드시 실패합니다. 구글 BeyondCorp 사례에서도 **7년에 걸쳐 점진적으로 마이그레이션**했습니다. 다음 순서로 도입을 권장합니다.

```
1단계: 강력한 인증 (MFA, 단기 토큰)
2단계: 서비스 간 mTLS 도입
3단계: ABAC 정책 엔진 도입
4단계: 지속적 모니터링 및 이상 탐지
5단계: 동적 정책 자동화
```

---

## 정리

Zero Trust는 "내부 네트워크는 안전하다"는 낡은 가정을 버리고, **모든 요청을 신원 불명의 외부 요청처럼 취급**하는 보안 철학입니다.

백엔드 개발자 입장에서 핵심은 다음 세 가지입니다.

1. **신원 중심 보안**: IP가 아닌 JWT/mTLS 기반으로 요청 주체를 명확히 검증한다.
2. **최소 권한 + Deny by Default**: 명시적으로 허용되지 않은 모든 접근은 거부한다.
3. **모든 것을 기록하고 분석한다**: 감사 로그는 선택이 아닌 필수이며, 이상 탐지의 기반이 된다.

Zero Trust는 단순히 보안 팀의 과제가 아닙니다. 백엔드 서비스를 설계하고 구현하는 개발자가 코드 레벨에서 함께 구현해야 하는 아키텍처입니다. 오늘 소개한 예제들을 시작점으로, 여러분의 서비스에 Zero Trust 원칙을 하나씩 녹여나가길 바랍니다.

---

**참고 자료**
- [NIST SP 800-207 Zero Trust Architecture](https://csrc.nist.gov/publications/detail/sp/800/207/final)
- [Google BeyondCorp: A New Approach to Enterprise Security](https://research.google/pubs/pub43231/)
- [SPIFFE/SPIRE 공식 문서](https://spiffe.io/)
- Spring Security OAuth2 공식 문서
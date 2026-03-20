# HTTP 캐싱 전략 Cache-Control 완전 정복

## 개요

웹 서비스의 성능을 논할 때 빠질 수 없는 주제가 바로 **HTTP 캐싱**이다. 아무리 서버 인프라를 증설하고 DB 쿼리를 최적화해도, 브라우저와 CDN이 불필요한 요청을 반복한다면 결국 병목은 네트워크 레이어에서 발생한다. 반대로 캐싱 전략을 잘못 설정하면 오래된 데이터가 사용자에게 노출되는 치명적인 문제가 생긴다.

`Cache-Control` 헤더는 HTTP/1.1에서 도입된 이후 현재까지 캐싱의 핵심 제어 수단으로 자리잡고 있다. 하지만 `max-age`, `no-cache`, `no-store`의 차이를 정확히 설명할 수 있는 개발자는 생각보다 많지 않다. 이 글에서는 `Cache-Control`의 각 디렉티브를 실무 관점에서 깊이 파헤치고, Spring Boot와 Nginx 기반의 실전 예제를 통해 즉시 적용 가능한 전략을 정리한다.

---

## 핵심 개념

### Cache-Control 디렉티브 분류

`Cache-Control` 헤더는 **요청(Request)**과 **응답(Response)** 양방향에서 사용할 수 있으며, 각 방향에서 유효한 디렉티브가 다르다.

#### 응답 헤더에서 자주 쓰는 디렉티브

| 디렉티브 | 설명 |
|---|---|
| `max-age=<seconds>` | 캐시 유효 시간(초 단위). 이 시간 내에는 서버에 재검증 없이 캐시 사용 |
| `s-maxage=<seconds>` | 공유 캐시(CDN, 프록시)에만 적용되는 max-age. max-age보다 우선함 |
| `no-cache` | 캐시를 저장하되, 사용 전 반드시 서버에 재검증 요청 |
| `no-store` | 캐시 자체를 저장하지 않음. 민감한 데이터에 사용 |
| `private` | 브라우저 같은 개인 캐시에만 저장. CDN/프록시에는 저장 불가 |
| `public` | 공유 캐시 포함 모든 캐시에 저장 허용 |
| `must-revalidate` | 캐시 만료 후 반드시 서버 재검증. 만료된 캐시를 절대 사용하지 않음 |
| `stale-while-revalidate=<seconds>` | 만료된 캐시를 반환하면서 백그라운드에서 갱신 |
| `immutable` | 리소스가 절대 변하지 않음을 선언. 유효 기간 내 재검증 요청 생략 |

### no-cache vs no-store: 가장 흔한 오해

많은 개발자가 `no-cache`를 "캐시하지 말라"는 의미로 오해한다. 하지만 실제로는 **"캐시는 저장하되, 사용 전에 서버에 유효성을 확인하라"**는 의미다.

```
# no-cache 동작 흐름
브라우저 → 서버: "이 리소스 아직 유효한가요?" (If-None-Match 또는 If-Modified-Since)
서버 → 브라우저: "유효함" (304 Not Modified) → 캐시된 응답 사용
            또는
서버 → 브라우저: "변경됨" (200 OK + 새 응답)
```

반면 `no-store`는 **캐시 자체를 금지**한다. 매 요청마다 서버에서 전체 응답을 받아야 하므로 진짜로 캐시를 원하지 않을 때 사용한다.

> 💡 **실무 팁**: 로그인 후 개인 정보 페이지는 `no-store`를, API 응답 중 자주 변하지만 변경 여부를 서버가 빠르게 판단할 수 있는 리소스는 `no-cache`를 쓰는 것이 적절하다.

### ETag와 Last-Modified: 재검증의 두 축

`no-cache`나 `must-revalidate`가 동작하려면 서버가 재검증 수단을 제공해야 한다.

- **ETag**: 리소스의 해시값 또는 버전 식별자. 클라이언트는 `If-None-Match` 헤더로 재검증
- **Last-Modified**: 리소스 최종 수정 시각. 클라이언트는 `If-Modified-Since` 헤더로 재검증

ETag가 더 정밀하지만 서버 연산 비용이 있고, 분산 환경에서는 서버마다 다른 ETag를 생성할 수 있어 주의가 필요하다.

---

## 실전 예제

### Spring Boot에서 Cache-Control 설정

#### 정적 리소스 캐싱 (WebMvcConfigurer)

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // JS, CSS 같은 정적 파일: 1년 캐시 + immutable
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/")
                .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS)
                        .cachePublic()
                        .immutable());

        // 자주 변경되는 이미지: 1시간 캐시
        registry.addResourceHandler("/images/**")
                .addResourceLocations("classpath:/images/")
                .setCacheControl(CacheControl.maxAge(1, TimeUnit.HOURS)
                        .cachePublic());
    }
}
```

#### REST API 응답 캐싱

```java
@RestController
@RequestMapping("/api")
public class ProductController {

    @GetMapping("/products/{id}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable Long id) {
        ProductDto product = productService.findById(id);

        // ETag 기반 조건부 응답
        String etag = "\"" + product.getVersion() + "\"";

        return ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(10, TimeUnit.MINUTES)
                        .cachePublic())
                .eTag(etag)
                .body(product);
    }

    @GetMapping("/users/me")
    public ResponseEntity<UserDto> getMyProfile(Principal principal) {
        UserDto user = userService.findByUsername(principal.getName());

        // 개인 정보: private + no-store
        return ResponseEntity.ok()
                .cacheControl(CacheControl.noStore())
                .body(user);
    }

    @GetMapping("/config")
    public ResponseEntity<ConfigDto> getPublicConfig() {
        ConfigDto config = configService.getPublicConfig();

        // 자주 변하지 않는 공개 설정: s-maxage로 CDN 캐싱
        return ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES)
                        .sMaxAge(30, TimeUnit.MINUTES)
                        .cachePublic())
                .body(config);
    }
}
```

#### ShallowEtagHeaderFilter 활용

Spring에서 제공하는 `ShallowEtagHeaderFilter`를 사용하면 컨트롤러 수정 없이 자동으로 ETag를 생성할 수 있다.

```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<ShallowEtagHeaderFilter> shallowEtagHeaderFilter() {
        FilterRegistrationBean<ShallowEtagHeaderFilter> filterRegistration
                = new FilterRegistrationBean<>();
        filterRegistration.setFilter(new ShallowEtagHeaderFilter());
        filterRegistration.addUrlPatterns("/api/products/*");
        return filterRegistration;
    }
}
```

> ⚠️ `ShallowEtagHeaderFilter`는 응답 본문 전체를 메모리에 버퍼링해서 MD5 해시를 계산한다. 대용량 응답에는 오히려 성능 저하를 유발할 수 있으므로 주의하자.

### Nginx에서 Cache-Control 설정

```nginx
server {
    listen 80;

    # 정적 파일: 1년 캐시 + immutable
    location ~* \.(js|css)$ {
        root /var/www/html;
        add_header Cache-Control "public, max-age=31536000, immutable";
        expires 365d;
    }

    # 이미지: 30일 캐시
    location ~* \.(png|jpg|gif|svg|webp)$ {
        root /var/www/html;
        add_header Cache-Control "public, max-age=2592000";
        expires 30d;
    }

    # API 프록시: 캐싱 금지
    location /api/ {
        proxy_pass http://backend:8080;
        add_header Cache-Control "no-store, no-cache";
        proxy_no_cache 1;
        proxy_cache_bypass 1;
    }

    # HTML: 재검증 필요
    location / {
        root /var/www/html;
        add_header Cache-Control "no-cache";
        try_files $uri $uri/ /index.html;
    }
}
```

### 리소스 버저닝(Cache Busting) 전략

`immutable`과 긴 `max-age`를 사용하면 리소스가 변경되어도 캐시가 유지되는 문제가 있다. 이를 해결하는 표준 방법이 **URL 버저닝**이다.

```
# 파일명에 콘텐츠 해시 포함
/static/app.a3f9c2d1.js      ← 내용이 바뀌면 해시도 바뀜
/static/style.b7e4a1c8.css
```

Webpack, Vite 같은 번들러는 기본적으로 이 방식을 지원한다. Spring Boot에서는 `ResourceUrlEncodingFilter`와 `ContentVersionStrategy`를 조합할 수 있다.

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    VersionResourceResolver resolver = new VersionResourceResolver()
            .addContentVersionStrategy("/**");

    registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS).immutable())
            .resourceChain(true)
            .addResolver(resolver);
}
```

---

## 주의사항 및 트레이드오프

### 1. CDN과 s-maxage 활용 시 캐시 무효화 비용

`s-maxage`를 길게 설정하면 CDN 히트율은 올라가지만, 내용이 변경되었을 때 **CDN 캐시를 즉시 무효화(Purge)**해야 한다. CloudFront, Cloudflare, Fastly 같은 CDN은 API를 통해 캐시 퍼지를 지원하지만, 대규모 트래픽에서 이 작업이 지연될 수 있다.

**전략**: 중요도 높은 콘텐츠 변경 시 배포 파이프라인에 CDN 퍼지 단계를 포함시키고, `surrogate-key` 또는 `Cache-Tag` 기반으로 선택적 퍼지가 가능한지 CDN 벤더에 확인하라.

### 2. stale-while-revalidate의 양면성

`stale-while-revalidate`는 만료된 캐시를 즉시 반환해 응답 지연을 줄이면서 백그라운드에서 갱신하는 강력한 전략이다. 하지만 갱신이 실패하면 계속 오래된 데이터를 반환할 수 있고, 실시간성이 중요한 데이터(주가, 재고 수량 등)에는 적합하지 않다.

### 3. 브라우저 강제 새로고침과 Cache-Control

사용자가 `Ctrl+Shift+R`(하드 리프레시)을 누르면 브라우저는 `Cache-Control: no-cache` 헤더를 요청에 포함시켜 서버에 재검증을 강제한다. `F5`(소프트 리프레시)는 조건부 요청만 보낸다. 개발/테스트 환경에서 이 차이를 항상 염두에 두어야 한다.

### 4. 분산 서버 환경에서의 ETag 일관성

여러 인스턴스가 동일 리소스를 서비스할 때 서버별로 ETag가 다르게 생성되면 불필요한 캐시 미스가 발생한다. ETag를 콘텐츠 해시 기반으로 생성하거나, 공유 캐시 저장소(Redis 등)에서 ETag를 관리하는 방식을 고려하라.

### 5. HTTPS와 캐시 동작

HTTP에서 `public` 캐시가 잘 동작하더라도, HTTPS에서는 일부 구형 프록시가 응답을 캐시하지 않을 수 있다. 현대 CDN은 문제가 없지만, 레거시 환경을 지원해야 한다면 검증이 필요하다.

---

## 캐싱 전략 결정 플로우

```
리소스 종류 판단
│
├─ 민감한 개인 데이터?
│   └─ Cache-Control: no-store
│
├─ 절대 변하지 않는 정적 파일 (해시 URL)?
│   └─ Cache-Control: public, max-age=31536000, immutable
│
├─ 자주 변하지 않는 공개 API?
│   ├─ CDN 캐싱 허용 → s-maxage 설정
│   └─ Cache-Control: public, max-age=300, s-maxage=1800
│
├─ 변경은 드물지만 즉시 반영이 필요?
│   └─ Cache-Control: no-cache (+ ETag or Last-Modified)
│
└─ HTML 진입점 (SPA index.html)?
    └─ Cache-Control: no-cache
```

---

## 정리

`Cache-Control`은 단순한 헤더가 아니라 **브라우저, CDN, 프록시 서버가 유기적으로 협력하는 캐싱 생태계의 설계도**다. 핵심 포인트를 다시 정리하면:

1. **`no-cache` ≠ 캐시 없음**. 재검증을 강제하는 것이지 캐시를 금지하는 게 아니다.
2. **정적 자산은 URL 버저닝 + `immutable`** 조합으로 최대 캐시 효율을 뽑는다.
3. **API 응답은 목적에 따라 세분화**한다. 공개 데이터는 `public + s-maxage`, 개인 데이터는 `no-store`.
4. **ETag를 활용한 재검증**으로 304 응답을 적극 유도하면 대역폭과 서버 부하를 동시에 줄일 수 있다.
5. **CDN 퍼지 전략**을 배포 프로세스에 포함시켜 캐시 일관성을 유지하라.

캐싱은 "한 번 설정하면 끝"이 아니다. 서비스 특성과 트래픽 패턴이 변함에 따라 지속적으로 측정하고 조정해야 한다. `Cache-Control`을 정확히 이해하는 것이 그 첫걸음이다.
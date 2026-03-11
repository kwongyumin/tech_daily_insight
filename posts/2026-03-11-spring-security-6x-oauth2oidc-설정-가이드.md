# Spring Security 6.x OAuth2/OIDC 설정 가이드

## 개요

Spring Security 6.x는 Spring Boot 3.x와 함께 Jakarta EE 기반으로 전환되면서 OAuth2/OIDC 설정 방식에도 상당한 변화가 생겼다. 기존 5.x에서 사용하던 `WebSecurityConfigurerAdapter`가 완전히 제거되었고, `HttpSecurity`를 람다 DSL로만 구성해야 하는 방식으로 바뀌었다. 또한 Authorization Server가 별도 프로젝트(`spring-authorization-server`)로 분리되면서 Resource Server와 Client 설정의 경계도 더욱 명확해졌다.

이 글에서는 실무에서 자주 마주치는 세 가지 시나리오를 중심으로 Spring Security 6.x의 OAuth2/OIDC 설정을 다룬다.

1. **OAuth2 Login (소셜 로그인)** — Google, Kakao 등 외부 IdP와의 연동
2. **Resource Server** — JWT Bearer Token 검증
3. **Authorization Server** — Spring Authorization Server 기반 자체 인증 서버 구축

---

## 핵심 개념

### OAuth2 역할 분리

Spring Security 6.x에서 OAuth2 관련 의존성은 역할에 따라 명확히 분리된다.

| 역할 | 의존성 |
|---|---|
| OAuth2 Client (소셜 로그인) | `spring-boot-starter-oauth2-client` |
| Resource Server (JWT 검증) | `spring-boot-starter-oauth2-resource-server` |
| Authorization Server | `spring-security-oauth2-authorization-server` |

### SecurityFilterChain 빈 방식

6.x에서는 반드시 `SecurityFilterChain`을 빈으로 등록해야 한다. `WebSecurityConfigurerAdapter` 상속 방식은 완전히 사라졌다.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

### OIDC와 OAuth2의 차이

OAuth2는 **인가(Authorization)** 프레임워크이고, OIDC(OpenID Connect)는 OAuth2 위에서 **인증(Authentication)**을 처리하는 계층이다. OIDC는 `id_token`(JWT)을 통해 사용자 정보를 전달하며, `/userinfo` 엔드포인트와 `/.well-known/openid-configuration` 디스커버리 문서를 표준으로 제공한다.

---

## 실전 예제

### 1. OAuth2 Login — Google & Kakao 소셜 로그인

**의존성 (build.gradle)**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

**application.yml**

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
          kakao:
            client-id: ${KAKAO_CLIENT_ID}
            client-secret: ${KAKAO_CLIENT_SECRET}
            client-authentication-method: client_secret_post
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/kakao"
            scope: profile_nickname, account_email
        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id
```

**SecurityConfig**

```java
@Configuration
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

    private final CustomOAuth2UserService oAuth2UserService;

    public OAuth2LoginSecurityConfig(CustomOAuth2UserService oAuth2UserService) {
        this.oAuth2UserService = oAuth2UserService;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login**", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(oAuth2UserService)
                )
                .successHandler(oAuth2AuthenticationSuccessHandler())
                .failureHandler(oAuth2AuthenticationFailureHandler())
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .clearAuthentication(true)
                .deleteCookies("JSESSIONID")
            );

        return http.build();
    }

    @Bean
    public AuthenticationSuccessHandler oAuth2AuthenticationSuccessHandler() {
        return (request, response, authentication) -> {
            OAuth2AuthenticationToken token = (OAuth2AuthenticationToken) authentication;
            String provider = token.getAuthorizedClientRegistrationId();
            // JWT 발급 또는 세션 처리 로직
            response.sendRedirect("/dashboard");
        };
    }
}
```

**CustomOAuth2UserService** — 사용자 정보 커스터마이징

```java
@Service
@RequiredArgsConstructor
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2User oAuth2User = super.loadUser(userRequest);
        String registrationId = userRequest.getClientRegistration().getRegistrationId();

        OAuth2UserInfo userInfo = OAuth2UserInfoFactory.getOAuth2UserInfo(
            registrationId, oAuth2User.getAttributes()
        );

        User user = userRepository.findByEmail(userInfo.getEmail())
            .map(existingUser -> existingUser.updateOAuth2Info(userInfo))
            .orElseGet(() -> userRepository.save(User.ofOAuth2(userInfo, registrationId)));

        return UserPrincipal.of(user, oAuth2User.getAttributes());
    }
}
```

---

### 2. Resource Server — JWT Bearer Token 검증

MSA 환경에서 각 서비스가 Authorization Server로부터 발급된 JWT를 검증하는 패턴이다.

**application.yml**

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # JWK Set URI 방식 (권장) - 공개키를 자동으로 로테이션
          jwk-set-uri: https://auth.example.com/.well-known/jwks.json
          # issuer-uri 방식 - OIDC 디스커버리 문서 활용
          # issuer-uri: https://auth.example.com
```

**SecurityConfig**

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // @PreAuthorize 활성화
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain resourceServerFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
                .authenticationEntryPoint(customAuthenticationEntryPoint())
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        authoritiesConverter.setAuthoritiesClaimName("roles");  // 커스텀 클레임명

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }

    @Bean
    public AuthenticationEntryPoint customAuthenticationEntryPoint() {
        return (request, response, authException) -> {
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("""
                {"error": "unauthorized", "message": "%s"}
                """.formatted(authException.getMessage()));
        };
    }
}
```

**컨트롤러에서 JWT 클레임 활용**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/me")
    public ResponseEntity<UserResponse> getMyInfo(
            @AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getSubject();
        String email = jwt.getClaimAsString("email");
        List<String> roles = jwt.getClaimAsStringList("roles");

        return ResponseEntity.ok(UserResponse.of(userId, email, roles));
    }

    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")
    @GetMapping("/{userId}")
    public ResponseEntity<UserResponse> getUser(@PathVariable String userId) {
        // ...
    }
}
```

---

### 3. Authorization Server — Spring Authorization Server 설정

**의존성**

```groovy
implementation 'org.springframework.boot:spring-boot-starter-oauth2-authorization-server'
```

**AuthorizationServerConfig**

```java
@Configuration
public class AuthorizationServerConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults());  // OIDC 활성화

        http
            .exceptionHandling(ex -> ex
                .defaultAuthenticationEntryPointFor(
                    new LoginUrlAuthenticationEntryPoint("/login"),
                    new MediaTypeRequestMatcher(MediaType.TEXT_HTML)
                )
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }

    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient webClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("web-client")
            .clientSecret("{bcrypt}" + new BCryptPasswordEncoder().encode("secret"))
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("http://localhost:3000/callback")
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope("read")
            .clientSettings(ClientSettings.builder()
                .requireAuthorizationConsent(true)
                .requireProofKey(true)  // PKCE 강제
                .build())
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(30))
                .refreshTokenTimeToLive(Duration.ofDays(7))
                .reuseRefreshTokens(false)
                .build())
            .build();

        return new InMemoryRegisteredClientRepository(webClient);
    }

    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        RSAKey rsaKey = generateRSAKey();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return (selector, context) -> selector.select(jwkSet);
    }

    private RSAKey generateRSAKey() {
        try {
            KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
            keyPairGenerator.initialize(2048);
            KeyPair keyPair = keyPairGenerator.generateKeyPair();
            return new RSAKey.Builder((RSAPublicKey) keyPair.getPublic())
                .privateKey(keyPair.getPrivate())
                .keyID(UUID.randomUUID().toString())
                .build();
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }

    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
            .issuer("https://auth.example.com")
            .build();
    }
}
```

---

## 주의사항 및 트레이드오프

### 1. 키 관리 — 인메모리 vs 외부 저장소

위 예제의 `generateRSAKey()`는 **서버 재시작 시 키가 변경**된다. 운영 환경에서는 반드시 키를 외부에 영구 저장해야 한다.

```java
// 운영 환경: Vault, AWS KMS, 또는 DB에서 키 로드
@Bean
public JWKSource<SecurityContext> jwkSource(KeyRepository keyRepository) {
    RSAKey rsaKey = keyRepository.loadCurrentRSAKey();
    return new ImmutableJWKSet<>(new JWKSet(rsaKey));
}
```

### 2. CSRF 설정

REST API + Stateless JWT 조합에서는 `csrf().disable()`이 적절하지만, 세션 기반 OAuth2 Login을 사용하는 경우 CSRF를 반드시 활성화해야 한다. SPA와 연동 시 `CookieCsrfTokenRepository.withHttpOnlyFalse()`를 사용하는 패턴을 고려하라.

### 3. 멀티 SecurityFilterChain 우선순위

Authorization Server와 Resource Server를 같은 애플리케이션에서 운영할 때 `@Order` 어노테이션으로 필터 체인 우선순위를 명확히 지정해야 한다. 일반적으로 Authorization Server(`@Order(1)`) > Resource Server(`@Order(2)`) > Default(`@Order(3)`) 순이다.

### 4. Refresh Token 보안

`reuseRefreshTokens(false)` 설정으로 Refresh Token
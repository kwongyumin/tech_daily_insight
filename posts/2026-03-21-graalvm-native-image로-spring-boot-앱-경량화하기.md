# GraalVM Native Image로 Spring Boot 앱 경량화하기

## 개요

마이크로서비스 아키텍처가 보편화되면서 애플리케이션의 **시작 시간(Startup Time)**과 **메모리 사용량**은 점점 더 중요한 운영 지표가 되었다. 특히 Kubernetes 환경에서 Pod가 빠르게 스케일 아웃되어야 하는 상황이나, AWS Lambda 같은 서버리스 환경에서 콜드 스타트(Cold Start) 문제를 해결해야 하는 경우라면 더욱 그렇다.

전통적인 Spring Boot 애플리케이션은 JVM 위에서 동작하기 때문에, 클래스 로딩, JIT 컴파일, 리플렉션 기반의 의존성 주입 등으로 인해 시작 시간이 수 초에서 길게는 수십 초까지 걸리기도 한다. 이 문제를 근본적으로 해결하는 방법이 바로 **GraalVM Native Image**다.

Spring Boot 3.x와 Spring Native의 GA(General Availability) 릴리즈 이후, GraalVM Native Image를 실무에 적용하는 것이 이전보다 훨씬 현실적인 선택지가 되었다. 이 글에서는 GraalVM Native Image의 핵심 원리부터 실제 Spring Boot 앱을 네이티브 이미지로 빌드하는 과정, 그리고 실무에서 마주하게 되는 트레이드오프까지 꼼꼼하게 살펴보겠다.

---

## 핵심 개념

### GraalVM Native Image란?

GraalVM은 Oracle이 개발한 고성능 JDK로, 기존 HotSpot JVM을 대체하거나 함께 사용할 수 있다. 그 중에서도 **Native Image**는 Java 애플리케이션을 AOT(Ahead-Of-Time) 컴파일하여 특정 플랫폼에 최적화된 **단일 실행 바이너리**로 만들어주는 기술이다.

핵심 동작 방식은 **Closed World Assumption**이다. 빌드 시점에 애플리케이션에서 도달 가능한(Reachable) 모든 코드, 클래스, 리소스를 정적으로 분석하고, 실행에 필요한 것만 포함시켜 바이너리를 생성한다. 결과적으로 다음과 같은 이점을 얻는다.

- **빠른 시작 시간**: JVM 초기화 없이 바로 실행되므로 수십 ms 수준의 시작 시간
- **낮은 메모리 사용량**: 불필요한 클래스 메타데이터가 제거되어 RSS(Resident Set Size)가 크게 줄어듦
- **컴팩트한 배포 단위**: JRE 없이 단일 바이너리로 배포 가능

### Spring AOT와의 연동

Spring Boot 3.x부터 공식적으로 **Spring AOT Engine**이 내장되었다. 빌드 타임에 Spring의 ApplicationContext를 분석하여, 리플렉션 없이 동작할 수 있도록 소스 코드와 힌트(Hints)를 자동 생성한다. 이로써 GraalVM이 Closed World Assumption을 적용할 때 Spring의 동적 특성으로 인해 발생하는 문제들을 사전에 해결해준다.

---

## 실전 예제

### 1. 프로젝트 초기 설정

Spring Boot 3.x와 GraalVM 17 이상 버전이 필요하다. [SDKMAN](https://sdkman.io/)을 활용하면 GraalVM 설치가 간편하다.

```bash
# GraalVM JDK 설치
sdk install java 21.0.2-graalce
sdk use java 21.0.2-graalce

# 버전 확인
java -version
# openjdk version "21.0.2" 2024-01-16
# OpenJDK Runtime Environment GraalVM CE 21.0.2+13.1 (build 21.0.2+13-jvmci-23.1-b30)
```

`pom.xml`에 필요한 의존성과 플러그인을 추가한다.

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

### 2. 간단한 REST API 구현

```java
// Product.java
@Entity
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private int price;

    // 기본 생성자 필수 (JPA 요구사항)
    protected Product() {}

    public Product(String name, int price) {
        this.name = name;
        this.price = price;
    }

    // Getter
    public Long getId() { return id; }
    public String getName() { return name; }
    public int getPrice() { return price; }
}

// ProductRepository.java
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByNameContaining(String keyword);
}

// ProductController.java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductRepository productRepository;

    public ProductController(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @GetMapping
    public List<Product> findAll() {
        return productRepository.findAll();
    }

    @PostMapping
    public Product create(@RequestBody Product product) {
        return productRepository.save(product);
    }

    @GetMapping("/search")
    public List<Product> search(@RequestParam String keyword) {
        return productRepository.findByNameContaining(keyword);
    }
}
```

### 3. 리플렉션 힌트 등록 (RuntimeHints)

Spring이 자동으로 처리하지 못하는 리플렉션 사용 케이스는 `RuntimeHintsRegistrar`를 직접 구현해야 한다.

```java
@Configuration
@ImportRuntimeHints(NativeConfig.AppRuntimeHints.class)
public class NativeConfig {

    static class AppRuntimeHints implements RuntimeHintsRegistrar {

        @Override
        public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
            // 리플렉션을 통해 접근하는 클래스 등록
            hints.reflection()
                .registerType(Product.class,
                    MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
                    MemberCategory.INVOKE_PUBLIC_METHODS,
                    MemberCategory.DECLARED_FIELDS);

            // 클래스패스 리소스 등록
            hints.resources()
                .registerPattern("data.sql")
                .registerPattern("schema.sql");

            // 동적 프록시 등록 (필요한 경우)
            hints.proxies()
                .registerJdkProxy(ProductRepository.class);
        }
    }
}
```

### 4. 네이티브 이미지 빌드

```bash
# AOT 처리 및 네이티브 이미지 빌드 (시간이 수 분 소요됨)
./mvnw -Pnative native:compile

# 빌드 결과물 확인
ls -lh target/
# -rwxr-xr-x  1 user  staff    52M  demo

# 네이티브 바이너리 실행
./target/demo
# Started DemoApplication in 0.089 seconds (process running for 0.113)
```

### 5. Docker 멀티스테이지 빌드로 컨테이너화

실무에서는 Buildpack을 활용하거나 직접 멀티스테이지 Dockerfile을 작성한다.

```dockerfile
# Dockerfile
# Stage 1: 빌드 스테이지 (GraalVM 포함)
FROM ghcr.io/graalvm/native-image-community:21 AS builder

WORKDIR /app
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -B

COPY src ./src
RUN ./mvnw -Pnative native:compile -DskipTests

# Stage 2: 런타임 스테이지 (최소한의 베이스 이미지)
FROM debian:bookworm-slim

WORKDIR /app
COPY --from=builder /app/target/demo ./demo

EXPOSE 8080
ENTRYPOINT ["./demo"]
```

```bash
# 이미지 빌드
docker build -t spring-native-demo .

# 이미지 크기 비교
docker images | grep demo
# spring-native-demo    latest    a1b2c3d4   120MB
# spring-jvm-demo       latest    e5f6g7h8   280MB

# 컨테이너 실행 및 시작 시간 확인
docker run -p 8080:8080 spring-native-demo
# Started DemoApplication in 0.102 seconds
```

### 6. Spring Boot Buildpack 활용 (권장 방법)

Maven 플러그인을 통해 Buildpack 기반으로 빌드하면 Dockerfile 없이 간편하게 OCI 이미지를 생성할 수 있다.

```bash
# Buildpack으로 네이티브 이미지 컨테이너 빌드
./mvnw spring-boot:build-image -Pnative

# 생성된 이미지로 실행
docker run -p 8080:8080 docker.io/library/demo:0.0.1-SNAPSHOT
```

---

## 주의사항 및 트레이드오프

### 빌드 시간 증가

네이티브 이미지 빌드는 일반 JAR 빌드 대비 **수십 배 이상 오래** 걸린다. 소규모 프로젝트도 3~5분, 대형 프로젝트는 10분 이상 소요될 수 있다. CI/CD 파이프라인 구성 시 이 점을 반드시 고려해야 한다.

```yaml
# GitHub Actions 예시: 빌드 캐싱 전략
- name: Cache Maven packages
  uses: actions/cache@v3
  with:
    path: ~/.m2
    key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
    restore-keys: ${{ runner.os }}-m2
```

### 동적 기능의 제한

리플렉션, 동적 클래스 로딩, JNI, 직렬화 등 런타임 동적 기능은 **빌드 타임에 명시적으로 힌트를 등록**해야 한다. 특히 다음 케이스를 주의하자.

- **Jackson, Gson 등 JSON 라이브러리**: 리플렉션 기반의 직렬화/역직렬화
- **로깅 프레임워크**: `Class.forName()` 사용
- **JPA/Hibernate**: 엔티티 클래스 프록시 생성
- **외부 라이브러리**: 아직 Native 미지원 라이브러리 존재

```java
// 잘못된 예: 런타임 동적 클래스 로딩
Class<?> clazz = Class.forName("com.example." + className); // 빌드 시점에 분석 불가

// 올바른 예: 정적으로 참조하여 힌트 등록
hints.reflection().registerType(SpecificClass.class, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS);
```

### 피크 성능(Peak Performance) 저하

JVM의 JIT 컴파일러는 런타임에 핫스팟(Hot Spot)을 최적화하지만, AOT 컴파일된 네이티브 이미지는 이러한 런타임 최적화가 없다. **처리량(Throughput) 중심의 장시간 실행 서비스**라면 오히려 일반 JVM 방식이 유리할 수 있다.

| 항목 | JVM (HotSpot) | Native Image |
|---|---|---|
| 시작 시간 | 수 초 | 수십 ms |
| 메모리 사용 | 높음 | 낮음 |
| 피크 처리량 | 높음 (JIT 최적화) | 보통 |
| 빌드 시간 | 빠름 | 느림 |
| 동적 기능 | 완전 지원 | 제한적 |

### 디버깅의 어려움

네이티브 바이너리는 일반 JVM 앱처럼 JVM 툴(JVisualVM, JFR, jstack 등)로 디버깅하기 어렵다. GraalVM은 `--enable-monitoring` 옵션을 제공하지만, 생태계가 아직 성숙하지 않아 운영 중 문제 추적이 까다롭다.

```bash
# 디버그 정보 포함 빌드
./mvnw -Pnative native:compile \
  -Dnative.debug=true \
  -DnativeImageArgs="--enable-monitoring=heapdump,jvmstat"
```

---

## 정리

GraalVM Native Image는 **시작 시간과 메모리 효율이 핵심인 워크로드**에서 강력한 옵션이다. Spring Boot 3.x의 공식 지원 덕분에 진입 장벽이 많이 낮아졌지만, 여전히 동적 기능 제약, 빌드 시간 증가, 피크 성능 한계 등 명확한 트레이드오프가 존재한다.

실무 적용 시 체크리스트를 정리하면 다음과 같다.

- [ ] **Spring Boot 3.x + GraalVM 21 LTS** 이상 버전 사용
- [ ] 사용 중인 **외부 라이브러리의 Native 지원 여부** 확인 ([GraalVM Reachability Metadata Repository](https://github.com/oracle/graalvm-reachability-metadata) 참고)
- [ ] `RuntimeHintsRegistrar`로 **리플렉션 힌트 명시적 등록**
- [ ] CI/CD 파이프라인에 **빌드 캐싱 전략** 적용
- [ ] 프로파일 기반 빌드로 **JVM 빌드와 Native 빌드 병행 지원**
- [ ] 부하 테스트를 통해 **처리량 저하 여부 검증**

서버리스, 컨테이너 오케스트레이션 환경이 지배적인 지금, GraalVM Native Image는 선택이 아닌 필수 역량이 되어가고 있다. 당장의 완벽한 마이그레이션보다는, 신규 서비스나 가벼운 마이크로서비스부터 점진적으로 도입해보길 권장한다.
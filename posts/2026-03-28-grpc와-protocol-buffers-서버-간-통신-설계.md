# gRPC와 Protocol Buffers 서버 간 통신 설계

## 개요

마이크로서비스 아키텍처가 보편화되면서 서버 간 통신 방식의 선택은 시스템 전체의 성능과 유지보수성에 직결되는 중요한 설계 결정이 되었다. REST/HTTP 기반 통신이 오랫동안 표준처럼 사용되어 왔지만, 고성능 서버 간 통신이 필요한 상황에서 **gRPC(Google Remote Procedure Call)**는 강력한 대안으로 자리잡고 있다.

gRPC는 Google이 개발한 오픈소스 RPC 프레임워크로, HTTP/2를 전송 계층으로 사용하고, **Protocol Buffers(protobuf)**를 기본 직렬화 포맷으로 채택한다. 이 조합은 낮은 지연 시간, 높은 처리량, 강타입 계약 기반 API 설계라는 세 가지 강점을 동시에 제공한다.

이 글에서는 실무에서 gRPC와 Protocol Buffers를 활용한 서버 간 통신을 설계할 때 알아야 할 핵심 개념과 Spring Boot 기반의 실전 예제를 다룬다.

---

## 핵심 개념

### Protocol Buffers란?

Protocol Buffers는 Google이 개발한 언어 중립적, 플랫폼 중립적 직렬화 메커니즘이다. `.proto` 파일에 데이터 구조를 정의하면 `protoc` 컴파일러가 각 언어에 맞는 코드를 자동 생성한다.

**JSON 대비 Protocol Buffers의 장점:**
- **크기**: 동일 데이터 기준 JSON 대비 3~10배 작은 바이너리 포맷
- **속도**: 직렬화/역직렬화 속도가 JSON 대비 현저히 빠름
- **타입 안정성**: 스키마를 강제하므로 런타임 오류를 컴파일 타임에 탐지 가능
- **하위 호환성**: 필드 번호 기반 설계로 버전 관리에 유리

### gRPC의 통신 패턴

gRPC는 네 가지 통신 패턴을 지원한다.

| 패턴 | 설명 | 사용 사례 |
|------|------|-----------|
| Unary RPC | 단일 요청 → 단일 응답 | 일반적인 API 호출 |
| Server Streaming | 단일 요청 → 스트림 응답 | 실시간 데이터 피드 |
| Client Streaming | 스트림 요청 → 단일 응답 | 파일 업로드, 배치 처리 |
| Bidirectional Streaming | 스트림 요청 → 스트림 응답 | 채팅, 실시간 게임 |

### HTTP/2가 주는 이점

gRPC는 HTTP/2 위에서 동작하기 때문에 다음 이점을 기본으로 누린다.

- **멀티플렉싱**: 하나의 TCP 연결로 여러 요청을 동시 처리
- **헤더 압축**: HPACK 알고리즘으로 헤더 오버헤드 감소
- **서버 푸시**: 클라이언트 요청 없이 서버가 데이터 전송 가능
- **이진 프레이밍**: 텍스트 기반 HTTP/1.1 대비 파싱 효율 향상

---

## 실전 예제

### 프로젝트 구조

```
grpc-demo/
├── grpc-proto/          # 공유 proto 정의
├── grpc-server/         # gRPC 서버 (주문 서비스)
└── grpc-client/         # gRPC 클라이언트 (API 게이트웨이)
```

### 1단계: Proto 파일 정의

```protobuf
// order.proto
syntax = "proto3";

package com.example.order;

option java_multiple_files = true;
option java_package = "com.example.grpc.order";

service OrderService {
  // Unary RPC
  rpc GetOrder (GetOrderRequest) returns (OrderResponse);
  
  // Server Streaming RPC
  rpc WatchOrderStatus (WatchOrderRequest) returns (stream OrderStatusUpdate);
  
  // Bidirectional Streaming RPC
  rpc ProcessOrders (stream OrderRequest) returns (stream OrderResponse);
}

message GetOrderRequest {
  string order_id = 1;
  string customer_id = 2;
}

message OrderRequest {
  string product_id = 1;
  int32 quantity = 2;
  double price = 3;
  repeated string tags = 4;
}

message OrderResponse {
  string order_id = 1;
  OrderStatus status = 2;
  string message = 3;
  int64 created_at = 4;
}

message WatchOrderRequest {
  string order_id = 1;
}

message OrderStatusUpdate {
  string order_id = 1;
  OrderStatus status = 2;
  string description = 3;
}

enum OrderStatus {
  UNKNOWN = 0;
  PENDING = 1;
  CONFIRMED = 2;
  SHIPPED = 3;
  DELIVERED = 4;
  CANCELLED = 5;
}
```

### 2단계: Gradle 의존성 설정

```groovy
// build.gradle (grpc-server)
plugins {
    id 'com.google.protobuf' version '0.9.4'
    id 'org.springframework.boot' version '3.2.0'
}

dependencies {
    implementation 'net.devh:grpc-server-spring-boot-starter:2.15.0.RELEASE'
    implementation 'io.grpc:grpc-protobuf:1.60.0'
    implementation 'io.grpc:grpc-stub:1.60.0'
    implementation 'com.google.protobuf:protobuf-java:3.25.1'
}

protobuf {
    protoc {
        artifact = 'com.google.protobuf:protoc:3.25.1'
    }
    plugins {
        grpc {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.60.0'
        }
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}
```

### 3단계: gRPC 서버 구현

```java
// OrderGrpcService.java
@GrpcService
@Slf4j
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    private final OrderUseCase orderUseCase;

    public OrderGrpcService(OrderUseCase orderUseCase) {
        this.orderUseCase = orderUseCase;
    }

    // Unary RPC 구현
    @Override
    public void getOrder(GetOrderRequest request,
                         StreamObserver<OrderResponse> responseObserver) {
        try {
            log.info("GetOrder 요청 수신: orderId={}", request.getOrderId());
            
            Order order = orderUseCase.findById(request.getOrderId());
            
            OrderResponse response = OrderResponse.newBuilder()
                    .setOrderId(order.getId())
                    .setStatus(mapStatus(order.getStatus()))
                    .setMessage("주문 조회 성공")
                    .setCreatedAt(order.getCreatedAt().toEpochMilli())
                    .build();
            
            responseObserver.onNext(response);
            responseObserver.onCompleted();
            
        } catch (OrderNotFoundException e) {
            responseObserver.onError(
                Status.NOT_FOUND
                    .withDescription("주문을 찾을 수 없습니다: " + request.getOrderId())
                    .withCause(e)
                    .asRuntimeException()
            );
        }
    }

    // Server Streaming RPC 구현
    @Override
    public void watchOrderStatus(WatchOrderRequest request,
                                  StreamObserver<OrderStatusUpdate> responseObserver) {
        log.info("WatchOrderStatus 스트리밍 시작: orderId={}", request.getOrderId());
        
        // 실제 구현에서는 이벤트 기반으로 처리
        orderUseCase.subscribeToStatusChanges(request.getOrderId(), status -> {
            if (!Thread.currentThread().isInterrupted()) {
                OrderStatusUpdate update = OrderStatusUpdate.newBuilder()
                        .setOrderId(request.getOrderId())
                        .setStatus(mapStatus(status.getStatus()))
                        .setDescription(status.getDescription())
                        .build();
                responseObserver.onNext(update);
            }
        });
        
        responseObserver.onCompleted();
    }

    // Bidirectional Streaming RPC 구현
    @Override
    public StreamObserver<OrderRequest> processOrders(
            StreamObserver<OrderResponse> responseObserver) {
        
        return new StreamObserver<>() {
            @Override
            public void onNext(OrderRequest request) {
                log.info("주문 처리 중: productId={}, qty={}", 
                    request.getProductId(), request.getQuantity());
                
                OrderResponse response = orderUseCase.processOrder(request)
                        .toGrpcResponse();
                responseObserver.onNext(response);
            }

            @Override
            public void onError(Throwable t) {
                log.error("스트리밍 오류 발생", t);
                responseObserver.onError(t);
            }

            @Override
            public void onCompleted() {
                log.info("주문 스트리밍 완료");
                responseObserver.onCompleted();
            }
        };
    }
}
```

### 4단계: gRPC 클라이언트 구현

```java
// OrderGrpcClient.java
@Component
@Slf4j
public class OrderGrpcClient {

    @GrpcClient("order-service")
    private OrderServiceGrpc.OrderServiceBlockingStub blockingStub;

    @GrpcClient("order-service")
    private OrderServiceGrpc.OrderServiceStub asyncStub;

    // Unary 호출
    public OrderResponse getOrder(String orderId, String customerId) {
        GetOrderRequest request = GetOrderRequest.newBuilder()
                .setOrderId(orderId)
                .setCustomerId(customerId)
                .build();
        
        try {
            return blockingStub
                    .withDeadlineAfter(5, TimeUnit.SECONDS) // 타임아웃 설정
                    .getOrder(request);
        } catch (StatusRuntimeException e) {
            if (e.getStatus().getCode() == Status.Code.NOT_FOUND) {
                throw new OrderNotFoundException(orderId);
            }
            throw new GrpcCommunicationException("주문 서비스 호출 실패", e);
        }
    }

    // Server Streaming 호출
    public void watchOrderStatus(String orderId, Consumer<OrderStatusUpdate> handler) {
        WatchOrderRequest request = WatchOrderRequest.newBuilder()
                .setOrderId(orderId)
                .build();
        
        CountDownLatch latch = new CountDownLatch(1);
        
        asyncStub.watchOrderStatus(request, new StreamObserver<>() {
            @Override
            public void onNext(OrderStatusUpdate update) {
                handler.accept(update);
            }

            @Override
            public void onError(Throwable t) {
                log.error("상태 감시 오류: orderId={}", orderId, t);
                latch.countDown();
            }

            @Override
            public void onCompleted() {
                log.info("상태 감시 완료: orderId={}", orderId);
                latch.countDown();
            }
        });
    }
}
```

### 5단계: 서버 설정

```yaml
# application.yml (서버)
grpc:
  server:
    port: 9090
    max-inbound-message-size: 10MB
    max-inbound-metadata-size: 1KB
    keep-alive-time: 30s
    keep-alive-timeout: 5s

# application.yml (클라이언트)
grpc:
  client:
    order-service:
      address: 'static://order-service:9090'
      negotiation-type: plaintext  # 개발 환경
      # negotiation-type: tls      # 운영 환경
      keep-alive-time: 30s
      keep-alive-without-calls: true
      deadline: 10s
```

### 인터셉터를 활용한 공통 처리

```java
// LoggingInterceptor.java
@Component
public class GrpcLoggingInterceptor implements ServerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(GrpcLoggingInterceptor.class);

    @Override
    public <Req, Resp> ServerCall.Listener<Req> interceptCall(
            ServerCall<Req, Resp> call,
            Metadata headers,
            ServerCallHandler<Req, Resp> next) {
        
        String methodName = call.getMethodDescriptor().getFullMethodName();
        long startTime = System.currentTimeMillis();
        
        log.info("[gRPC] 요청 수신: method={}", methodName);
        
        // 인증 토큰 검증
        String authToken = headers.get(
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER)
        );
        if (authToken == null) {
            call.close(Status.UNAUTHENTICATED.withDescription("인증 토큰 없음"), new Metadata());
            return new ServerCall.Listener<>() {};
        }

        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<>(
                next.startCall(new ForwardingServerCall.SimpleForwardingServerCall<>(call) {
                    @Override
                    public void close(Status status, Metadata trailers) {
                        long elapsed = System.currentTimeMillis() - startTime;
                        log.info("[gRPC] 요청 완료: method={}, status={}, elapsed={}ms",
                                methodName, status.getCode(), elapsed);
                        super.close(status, trailers);
                    }
                }, headers)) {};
    }
}
```

---

## 주의사항 및 트레이드오프

### 1. 스키마 버전 관리의 중요성

Proto 파일은 계약(Contract)이다. 필드를 삭제하거나 필드 번호를 변경하면 하위 호환성이 깨진다.

```protobuf
// ❌ 잘못된 방식 - 필드 번호 재사용 금지
message OrderRequest {
  // string product_id = 1; // 삭제된 필드
  string item_id = 1;  // 같은 번호를 다른 용도로 재사용 -> 데이터 오염!
}

// ✅ 올바른 방식 - reserved로 보호
message OrderRequest {
  reserved 1;
  reserved "product_id";
  string item_id = 2;  // 새 번호 사용
}
```

### 2. 오류 처리와 gRPC 상태 코드

gRPC는 HTTP 상태 코드 대신 자체 상태 코드를 사용한다. 적절한 상태 코드 매핑이 클라이언트 측 재시도 전략에 영향을 미친다.

| gRPC Status | HTTP 상태 | 용도 |
|-------------|----------|------|
| OK | 200 | 성공 |
| NOT_FOUND | 404 | 리소스 없음 |
| INVALID_ARGUMENT | 400 | 잘못된 입력 |
| UNAVAILABLE | 503 | 서비스 불가 (재시도 권장) |
| DEADLINE_EXCEEDED | 504 | 타임아웃 |
| INTERNAL | 500 | 서버 내부 오류 |

### 3. 브라우저 직접 통신의 한계

gRPC는 HTTP/2의 저수준 제어가 필요해 브라우저에서 직접 호출이 불가능하다. 브라우저 지원이 필요한 경우 **gRPC-Web** 또는 **gRPC-Gateway**를 통해 REST 변환 레이어를 추가해야 한다.

### 4. 운영 환경에서의 고려사항

- **TLS 설정 필수**: 서비스 간 통신에 mTLS(상호 인증) 적용 권장
- **Keep-alive 튜닝**: 네트워크 장비의 유휴 연결 차단 정책에 맞게 설정
- **로드밸런서 호환성**: L7 로드밸런서가 HTTP/2를 지원해야 함 (AWS ALB, Envoy 등)
- **메시지 크기 제한**: 기본 4MB 제한을 초과하는 대용량 데이터는 청크 분할 처리

### 5. REST vs gRPC 선택 기준

| 상황 | 권장 방식 |
|------|----------|
| 외부 API (브라우저/모바일 클라이언트) | REST |
| 내부 마이크로서비스 간 통신 | gRPC |
| 실시간 양방향 통신 | gRPC Streaming |
| 단순한 CRUD 서비스 | REST |
| 고성능, 저지연이 중요한 서비스 | gRPC |

---

## 정리

gRPC와 Protocol Buffers는 마이크로서비스 간 고성능 통신이 필요한 환경에서 REST의 강력한 대안이다. 핵심 장점을 요약하면 다음과 같다.

1. **성능**: HTTP/2 멀티플렉싱과 protobuf 바이너리 직렬화로 REST 대비 월등한 처리량
2. **타입 안전성**: `.proto` 스키마가 API 계약을 강제해 런타임 오류 감소
3. **다양한 통신 패턴**: Unary, 서버/클라이언트/양방향 스트리밍 기본 지원
4. **코드 자동 생성**: 다국어 클라이언트/서버 스텁 자동 생성으로 개발 생산성 향상

다만 브라우저 직접 연동의 어려움, 스키마 버전 관리의 엄격함, HTTP/2 지원 인프라 필요 등은 도입 전 충분히 검토해야 할 트레이드오프다.

실무에서는 외부 API는 REST로 노출하되, 내부 서비스 간 통신은 gRPC로 구성하는 **하이브리드 아키텍처**가 가장 현실적인 선택인 경우가 많다. 팀의 숙련도와 인프라 환경을 고려해 점진적으로 도입하는 전략을 권장한다.
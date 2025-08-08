# 04 | 외부 연동이 문제일 때 살펴봐야 할 것들

> 접수 시스템에서 결제 외부 서비스 연동 중 502 Bad Gateway와 no healthy upstream 오류를 반복적으로 마주했다면? 이런 상황에서 서버가 무한정 대기한다면 어떻게 될까?

<p align="center">
  <img src="https://github.com/user-attachments/assets/1f57ccbe-9b36-4fed-a06b-4ea6cf9d4733" width="32%" />
  <img src="https://github.com/user-attachments/assets/41713289-b2ea-4a95-93b9-54a9e0ac4fe5" width="32%" />
  <img src="https://github.com/user-attachments/assets/61ba601d-a787-40ba-9b06-6a0699f75fbe" width="32%" />
</p>

## 1. 타임아웃(Timeout)

### 타임아웃이란?

- **타임아웃(Timeout)**: 외부 서비스 연동 시 응답을 기다리는 최대 시간을 설정하는 것
- 타임아웃을 설정하지 않으면 외부 서비스가 응답하지 않을 때 서버가 무한정 대기하게 됨. 결국 모든 스레드가 대기 상태가 되어 서비스 전체가 마비될 수 있음. 타임아웃 설정으로 서버 부하를 완화하고 장애 전파를 방지할 수 있음.

<img width="1280" height="314" alt="image" src="https://github.com/user-attachments/assets/0bca0a22-96f9-4023-b599-ba8b5b5b813e" />

### 타임아웃의 종류

외부 서비스 연동 시 설정해야 할 타임아웃은 크게 2가지로 나뉨.

#### 1) 연결 타임아웃(Connection Timeout)
- TCP 연결을 수립하는 데 걸리는 최대 시간
- 권장값: 3-5초
- 네트워크 상태가 좋지 않거나 서버가 과부하일 때 연결 자체가 느려질 수 있음

#### 2) 읽기 타임아웃(Read Timeout)
- 연결 후 데이터를 읽어오는 데 걸리는 최대 시간
- 권장값: 10-30초
- 외부 서비스의 처리 시간에 따라 조정 필요

### Spring Boot 3.x에서 타임아웃 설정

Spring Boot 환경에서는 RestClient를 사용하는 것이 권장됨.

```java
@Configuration
public class RestClientConfig {
    
    @Bean
    public RestClient restClient() {
        return RestClient.builder()
            .requestFactory(clientHttpRequestFactory())
            .build();
    }
    
    @Bean
    public ClientHttpRequestFactory clientHttpRequestFactory() {
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory();
        
        factory.setConnectTimeout(Duration.ofSeconds(5));  // 연결 타임아웃
        factory.setReadTimeout(Duration.ofSeconds(10));    // 읽기 타임아웃
        
        return factory;
    }
}
```

### WebClient를 활용한 Reactive 방식

비동기 처리가 중요한 서비스에서는 WebClient를 활용할 수 있음.

```java
@Component
public class PaymentClient {
    
    private final WebClient webClient;
    
    public PaymentClient() {
        this.webClient = WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(
                HttpClient.create()
                    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000) // 연결 타임아웃
                    .responseTimeout(Duration.ofSeconds(10)) // 응답 전체 타임아웃
            ))
            .build();
    }
    
    public Mono<PaymentResponse> processPayment(PaymentRequest request) {
        return webClient.post()
            .uri("/api/payments")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(PaymentResponse.class)
            .timeout(Duration.ofSeconds(15)); // Operator 레벨 타임아웃
    }
}
```

### 타임아웃 설정 시 주의사항

- **타임아웃이 너무 짧으면** 정상적인 처리도 에러로 판단될 수 있음
- **타임아웃이 너무 길면** 장애 상황에서 서버 부하가 지속될 수 있음
- **권장 접근법**: 외부 서비스의 응답 시간을 모니터링하고 점진적으로 최적화

## 2. 재시도(Retry)

접수 시 발생한 오류들을 보면 대부분 일시적인 문제였을 가능성이 높음. 이런 상황에서 재시도(Retry)는 문제를 해결할 수 있는 효과적인 방법임.

<img width="1280" height="472" alt="image" src="https://github.com/user-attachments/assets/6e1c5d4c-44e6-4da8-b408-63bdb51cd517" />

### 재시도 가능한 조건

모든 오류에 대해 재시도를 하면 안 됨. 재시도가 가능한 조건은 다음과 같음.

- **단순 조회 기능**: GET 요청과 같이 서버 상태를 변경하지 않는 작업
- **연결 타임아웃**: 네트워크 일시적 장애나 서버 부하로 인한 연결 실패
- **멱등성을 가진 변경 기능**: 여러 번 실행해도 같은 결과를 보장하는 작업
- **5xx 서버 오류**: 일시적인 서버 내부 오류 (502, 503 등)

### 재시도 불가능한 조건

- **4xx 클라이언트 오류**: 요청 자체에 문제가 있는 경우
- **인증 오류**: 401 Unauthorized, 403 Forbidden
- **비즈니스 로직 오류**: 잔액 부족, 상품 품절 등
- **비멱등 작업**: 중복 실행 시 부작용이 있는 결제, 주문 등

### Spring Retry를 활용한 구현

Spring Boot 환경에서는 @Retryable 어노테이션으로 간단하게 재시도 로직을 구현할 수 있음.

```java
@Service
public class PaymentService {
    
    @Retryable(
        retryFor = {ConnectTimeoutException.class, SocketTimeoutException.class},
        noRetryFor = {IllegalArgumentException.class, AuthenticationException.class},
        maxAttempts = 3,
        backoff = @Backoff(
            delay = 1000,           // 1초 대기
            multiplier = 2.0,       // 지수 백오프
            maxDelay = 10000        // 최대 10초
        )
    )
    public PaymentResult processPayment(PaymentRequest request) {
        // 외부 결제 API 호출
        return paymentClient.charge(request);
    }
    
    @Recover
    public PaymentResult recover(Exception ex, PaymentRequest request) {
        log.error("결제 재시도 실패: {}", ex.getMessage());
        return PaymentResult.failed("일시적으로 결제 서비스에 접근할 수 없습니다.");
    }
}
```

### 백오프(Backoff) 전략

재시도 간격을 어떻게 설정하느냐에 따라 시스템의 안정성이 크게 달라짐.

#### 1) 고정 간격 재시도

매번 동일한 간격으로 재시도
- **단점**: 여러 클라이언트가 동시에 재시도하면 서버에 부하 집중
- **사용 시기**: 단순한 케이스나 트래픽이 적은 환경

#### 2) 지수 백오프

- 재시도할 때마다 대기 시간을 2배씩 늘림 (1초 → 2초 → 4초)
- **장점**: 서버가 회복할 시간을 제공하고 부하 분산 효과
- **사용 시기**: 대부분의 상황에서 권장

#### 3) 지터(Jitter) 적용

- 대기 시간에 랜덤한 값을 추가하여 동시 요청 분산
- **효과**: Thundering Herd 문제 방지
- **구현**: 대기 시간 ± (0~500ms) 랜덤 값

### 재시도 설정 가이드라인

- **재시도 횟수**: 3-5회 (너무 많으면 사용자 대기 시간 증가)
- **초기 대기 시간**: 500ms-1초
- **최대 대기 시간**: 30초 (사용자 경험 고려)
- **전체 재시도 시간**: 60초 이내 권장

## 3. 서킷 브레이커(Circuit Breaker)

서킷 브레이커(Circuit Breaker)는 가정용 누전 차단기와 비슷하게 동작함. 외부 서비스에 과도한 오류가 발생하면 연동을 자동으로 중지시켜 전체 시스템을 보호함.

<img width="1280" height="357" alt="image" src="https://github.com/user-attachments/assets/e842a7db-ddef-4ac6-9e5e-2477017dd2f9" />

### 서킷 브레이커의 3가지 상태

#### 1) CLOSED (정상 상태)
- 모든 요청이 외부 서비스로 전달됨
- 실패율을 지속적으로 모니터링
- 실패율이 임계치를 초과하면 OPEN 상태로 전환

#### 2) OPEN (차단 상태)
- 모든 요청을 즉시 차단하고 에러 응답
- 외부 서비스 호출을 하지 않아 빠른 실패 처리
- 설정된 시간이 지나면 HALF_OPEN 상태로 전환

#### 3) HALF_OPEN (반개방 상태)
- 제한된 수의 요청만 외부 서비스로 전달
- 성공하면 CLOSED, 실패하면 OPEN 상태로 전환
- 외부 서비스 회복 여부를 테스트하는 상태

### Resilience4j를 활용한 구현

Netflix Hystrix가 유지보수 모드로 전환된 후, Resilience4j가 표준으로 자리잡음.

```java
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreaker paymentCircuitBreaker() {
        return CircuitBreaker.of("payment-service", 
            CircuitBreakerConfig.custom()
                .failureRateThreshold(50.0f)        // 실패율 50% 초과 시 OPEN
                .waitDurationInOpenState(Duration.ofSeconds(30))  // 30초 대기
                .slidingWindowSize(10)              // 최근 10개 요청으로 판단
                .minimumNumberOfCalls(5)            // 최소 5개 호출 후 판단
                .permittedNumberOfCallsInHalfOpenState(3)  // HALF_OPEN에서 3개 테스트
                .slowCallRateThreshold(80.0f)       // 느린 호출 80% 초과 시 실패로 판단
                .slowCallDurationThreshold(Duration.ofSeconds(5))  // 5초 이상을 느린 호출로 판단
                .build()
        );
    }
}
```

### 서비스에서 서킷 브레이커 적용

```java
@Service
public class ResilientPaymentService {
    
    private final PaymentClient paymentClient;
    private final CircuitBreaker circuitBreaker;
    
    public ResilientPaymentService(PaymentClient paymentClient, 
                                 CircuitBreaker paymentCircuitBreaker) {
        this.paymentClient = paymentClient;
        this.circuitBreaker = paymentCircuitBreaker;
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        Supplier<PaymentResult> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> {
                return paymentClient.processPayment(request);
            });
            
        try {
            return decoratedSupplier.get();
        } catch (CallNotPermittedException e) {
            // 서킷 브레이커가 OPEN 상태
            log.warn("결제 서비스 일시 중단: {}", e.getMessage());
            return PaymentResult.unavailable("결제 서비스가 일시적으로 중단되었습니다.");
        }
    }
}
```

### 서킷 브레이커 설정 가이드라인

- **실패율 임계치**: 50-70% (외부 서비스 특성에 따라 조정)
- **대기 시간**: 30-60초 (외부 서비스 회복 시간 고려)
- **슬라이딩 윈도우**: 10-20개 (충분한 샘플 수 확보)
- **최소 호출 수**: 5-10개 (통계적 신뢰성 확보)

## 4. 동시 요청 제한: 과부하 방지를 위한 Rate Limiting

외부 서비스나 우리 서비스에 과도한 요청이 몰리면 503 Service Unavailable 상태 코드를 사용해 과부하 상황임을 클라이언트에 알릴 수 있음.

### Rate Limiter 구현

```java
@Component
public class PaymentRateLimiter {
    
    private final RateLimiter rateLimiter;
    
    public PaymentRateLimiter() {
        this.rateLimiter = RateLimiter.of("payment-api",
            RateLimiterConfig.custom()
                .limitRefreshPeriod(Duration.ofSeconds(1))  // 1초마다 갱신
                .limitForPeriod(10)                         // 초당 10개 요청 허용
                .timeoutDuration(Duration.ofSeconds(3))     // 3초 대기
                .build()
        );
    }
    
    public PaymentResult processWithRateLimit(PaymentRequest request) {
        Supplier<PaymentResult> decoratedSupplier = RateLimiter
            .decorateSupplier(rateLimiter, () -> {
                return paymentService.process(request);
            });
            
        return decoratedSupplier.get();
    }
}
```

### HTTP 상태 코드 활용

```java
@ControllerAdvice
public class PaymentExceptionHandler {
    
    @ExceptionHandler(RequestNotPermitted.class)
    public ResponseEntity<ErrorResponse> handleRateLimitExceeded(
        RequestNotPermitted ex) {
        
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)  // 503 상태 코드
            .header("Retry-After", "60")             // 60초 후 재시도 안내
            .body(ErrorResponse.builder()
                .message("요청이 너무 많습니다. 잠시 후 다시 시도해주세요.")
                .code("RATE_LIMIT_EXCEEDED")
                .build());
    }
    
    @ExceptionHandler(CallNotPermittedException.class)
    public ResponseEntity<ErrorResponse> handleCircuitBreakerOpen(
        CallNotPermittedException ex) {
        
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)  // 503 상태 코드
            .body(ErrorResponse.builder()
                .message("서비스가 일시적으로 이용할 수 없습니다.")
                .code("SERVICE_UNAVAILABLE")
                .build());
    }
}
```

## 5. 정리

### 지향해야 할 행동

- **모든 외부 연동에 타임아웃 설정**: 무한 대기 방지
- **멱등한 작업에만 재시도 적용**: 부작용 방지
- **적절한 백오프 전략 사용**: 지수 백오프 + 지터
- **서킷 브레이커로 장애 전파 차단**: 전체 시스템 보호
- **의미있는 HTTP 상태 코드 사용**: 클라이언트에게 명확한 정보 제공
- **모니터링과 알림 구축**: 장애 상황 신속 감지

### 피해야 할 행동

- **무한 재시도**: 반드시 최대 횟수 제한
- **비멱등 작업 재시도**: 중복 결제 등 부작용 발생 가능
- **고정 간격 재시도**: Thundering Herd 문제 발생
- **과도하게 짧은 타임아웃**: 정상 처리도 실패로 처리
- **모니터링 없는 운영**: 문제 상황 인지 불가
- **모든 오류에 재시도**: 4xx 오류에는 재시도 금지

## 참고 자료

1. [Resilience4j 공식 문서, "Circuit Breaker"](https://resilience4j.readme.io/docs/circuitbreaker)
2. [Spring Boot 공식 문서, "RestClient"](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-restclient)
3. [AWS, "Timeouts, retries, and backoff with jitter"](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
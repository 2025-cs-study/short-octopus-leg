# 05 | 비동기 연동, 언제 어떻게 써야 할까

## 1. 비동기 연동이란?

**비동기 연동**은 요청을 보낸 후 응답을 기다리지 않고 다른 작업을 계속할 수 있는 통신 방식을 말함.

### 일상생활 예시

**동기 방식: 카페에서 주문**
```
1. 카운터에서 커피 주문
2. 커피가 나올 때까지 대기 (아무것도 못함)
3. 커피 받고 나서야 다른 일 가능
```

**비동기 방식: 배달 주문**
```
1. 앱으로 음식 주문
2. 주문 완료 후 바로 다른 일 진행 (청소, 업무 등)
3. 음식이 도착하면 알림으로 확인
```

### 언제 사용할까?
- 외부 API 호출이나 이메일/SMS 발송 같은 **시간이 오래 걸리는 작업**
- 사용자 응답과 **직접적인 관련이 없는 부가 작업**
- **독립적으로 실행 가능한 여러 작업**들

### 주의할 점
- 트랜잭션이 중요한 핵심 로직은 동기로 유지
- 같은 클래스 내에서 @Async 메서드 호출 시 비동기 동작 안함
- 적절한 예외 처리와 모니터링 필수

## 2. 동기 vs 비동기 비교

| | 동기 처리 | 비동기 처리 |
|------|-----------|-------------|
| **실행 방식** | 순차적 실행 | 병렬 실행 |
| **대기 시간** | 각 작업 완료까지 대기 | 대기하지 않음 |
| **처리 시간** | 모든 작업 시간의 합 | 가장 긴 작업 시간 |
| **구현 복잡도** | 간단 | 상대적으로 복잡 |
| **사용 케이스** | 순서가 중요한 작업 | 독립적인 작업들 |

### 동기(Synchronous) 처리

```java
// 동기 방식 - 각 작업이 순차적으로 실행됨
@Service
public class UserService {
    
    public void registerUser(String email, String name) {
        // 1. 사용자 저장 (1초)
        saveUser(email, name);
        
        // 2. 이메일 발송 (3초) - 위 작업이 완료된 후 실행
        sendWelcomeEmail(email);
        
        // 3. SMS 발송 (2초) - 위 작업이 완료된 후 실행
        sendWelcomeSms(name);
        
        // 총 소요 시간: 6초
    }
}
```

### 비동기(Asynchronous) 처리

```java
// 비동기 방식 - 여러 작업이 동시에 실행됨
@Service
public class UserService {
    
    public void registerUser(String email, String name) {
        // 1. 사용자 저장 (1초)
        saveUser(email, name);
        
        // 2. 이메일과 SMS를 동시에 처리 시작
        sendWelcomeEmailAsync(email);    // 3초 (비동기)
        sendWelcomeSmsAsync(name);       // 2초 (비동기)
        
        // 총 소요 시간: 약 3초
    }
}
```

## 3. 블로킹 vs 논블로킹 비교

### 관점의 차이

- **동기/비동기**: 작업의 **순서**와 **완료 시점**에 대한 관점
- **블로킹/논블로킹**: **제어권**을 누가 가지고 있는지에 대한 관점

### 블로킹(Blocking)

```java
// 블로킹 방식 - 결과가 올 때까지 기다림
public String callExternalApi() {
    String result = httpClient.get("http://api.example.com/data");
    // 여기서 응답이 올 때까지 기다림 (제어권을 외부 API가 가짐)
    return result; // 응답을 받은 후에 실행됨
}
```

### 논블로킹(Non-Blocking)

```java
// 논블로킹 방식 - 결과를 기다리지 않음
public CompletableFuture<String> callExternalApiAsync() {
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        return httpClient.get("http://api.example.com/data");
    });
    
    // 여기서 바로 리턴 (제어권을 유지)
    return future; // 호출자가 원할 때 결과 확인 가능
}
```

## 4. 비동기 연동 필요성

### 왜 비동기가 필요할까?

#### 1. 사용자 경험 개선

**Before (동기):**
```java
@RestController
public class OrderController {
    
    @PostMapping("/orders")
    public ResponseEntity<String> createOrder(@RequestBody OrderRequest request) {
        // 주문 생성 (0.5초)
        orderService.createOrder(request);
        
        // 결제 처리 (3초) - 사용자는 3초간 대기
        paymentService.processPayment(request.getPaymentInfo());
        
        // 이메일 발송 (2초) - 사용자는 추가로 2초 대기
        emailService.sendOrderConfirmation(request.getEmail());
        
        return ResponseEntity.ok("주문이 완료되었습니다"); // 5.5초 후 응답
    }
}
```

**After (비동기):**
```java
@RestController
public class OrderController {
    
    @PostMapping("/orders")
    public ResponseEntity<String> createOrder(@RequestBody OrderRequest request) {
        // 주문 생성 (0.5초)
        orderService.createOrder(request);
        
        // 결제와 이메일을 비동기로 처리
        paymentService.processPaymentAsync(request.getPaymentInfo());
        emailService.sendOrderConfirmationAsync(request.getEmail());
        
        return ResponseEntity.ok("주문이 접수되었습니다"); // 0.5초 후 바로 응답
    }
}
```

#### 2. 시스템 성능 향상

```java
// 동기 방식: 100명의 사용자 정보를 순차 조회
public List<UserInfo> getUserInfos(List<Long> userIds) {
    List<UserInfo> results = new ArrayList<>();
    
    for (Long userId : userIds) {
        UserInfo info = externalApiClient.getUserInfo(userId); // 각각 1초씩
        results.add(info);
    }
    
    return results; // 총 100초 소요
}

// 비동기 방식: 100명의 사용자 정보를 병렬 조회
public List<UserInfo> getUserInfosAsync(List<Long> userIds) {
    List<CompletableFuture<UserInfo>> futures = userIds.stream()
        .map(userId -> CompletableFuture.supplyAsync(() -> 
            externalApiClient.getUserInfo(userId) // 동시에 실행
        ))
        .collect(Collectors.toList());
    
    return futures.stream()
        .map(CompletableFuture::join)
        .collect(Collectors.toList()); // 총 1~2초 소요
}
```

#### 3. 장애 격리

```java
// 동기 방식: 이메일 서버 장애 시 전체 회원가입 실패
public void registerUser(UserRequest request) {
    userRepository.save(createUser(request));     // 성공
    emailService.sendWelcomeEmail(request);       // 실패 → 전체 실패
    smsService.sendWelcomeSms(request);          // 실행되지 않음
}

// 비동기 방식: 이메일 서버 장애와 무관하게 회원가입 성공
public void registerUserAsync(UserRequest request) {
    userRepository.save(createUser(request));     // 성공
    
    // 부가 기능들은 독립적으로 처리
    CompletableFuture.runAsync(() -> 
        emailService.sendWelcomeEmail(request)    // 실패해도 다른 기능에 영향 없음
    );
    CompletableFuture.runAsync(() -> 
        smsService.sendWelcomeSms(request)        // 정상 실행
    );
}
```

## 5. 스프링 부트에서 적용

### 5.1 기본 설정

#### @EnableAsync 설정

```java
@Configuration
@EnableAsync  // 비동기 기능 활성화
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);      // 기본 스레드 수
        executor.setMaxPoolSize(5);       // 최대 스레드 수
        executor.setQueueCapacity(100);   // 큐 크기
        executor.setThreadNamePrefix("Async-"); // 스레드 이름
        executor.initialize();
        return executor;
    }
}
```

### 5.2 @Async 기본 사용법

#### 1) 반환값이 없는 비동기 메서드

```java
@Service
public class EmailService {
    
    private static final Logger log = LoggerFactory.getLogger(EmailService.class);
    
    @Async("taskExecutor")
    public void sendEmailAsync(String to, String subject, String content) {
        log.info("이메일 발송 시작: {} - Thread: {}", to, Thread.currentThread().getName());
        
        try {
            // 이메일 발송 시뮬레이션
            Thread.sleep(2000);
            log.info("이메일 발송 완료: {}", to);
        } catch (InterruptedException e) {
            log.error("이메일 발송 중 오류", e);
        }
    }
}
```

#### 2) CompletableFuture로 결과값 반환

```java
@Service
public class UserService {
    
    @Async("taskExecutor")
    public CompletableFuture<String> getUserNameAsync(Long userId) {
        try {
            // 사용자 정보 조회 시뮬레이션
            Thread.sleep(1000);
            String userName = "사용자" + userId;
            return CompletableFuture.completedFuture(userName);
        } catch (InterruptedException e) {
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

### 5.3 여러 비동기 작업 조합하기

```java
@RestController
public class UserController {
    
    private final UserService userService;
    private final EmailService emailService;
    
    @GetMapping("/users/{userId}/profile")
    public ResponseEntity<UserProfile> getUserProfile(@PathVariable Long userId) {
        long startTime = System.currentTimeMillis();
        
        try {
            // 여러 작업을 동시에 실행
            CompletableFuture<String> nameFuture = userService.getUserNameAsync(userId);
            CompletableFuture<String> addressFuture = userService.getAddressAsync(userId);
            CompletableFuture<List<Order>> ordersFuture = orderService.getOrdersAsync(userId);
            
            // 모든 작업이 완료될 때까지 대기
            CompletableFuture.allOf(nameFuture, addressFuture, ordersFuture).join();
            
            // 결과 조합
            UserProfile profile = UserProfile.builder()
                .name(nameFuture.get())
                .address(addressFuture.get())
                .orders(ordersFuture.get())
                .build();
            
            long endTime = System.currentTimeMillis();
            log.info("프로필 조회 완료: {}ms", endTime - startTime);
            
            return ResponseEntity.ok(profile);
            
        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
        }
    }
}
```

### 5.4 스프링 이벤트 활용

#### 이벤트 클래스 정의

```java
@Getter
@AllArgsConstructor
public class UserRegisteredEvent {
    private final Long userId;
    private final String email;
    private final String name;
    private final LocalDateTime registeredAt;
}
```

#### 이벤트 발행

```java
@Service
@Transactional
public class UserService {
    
    private final ApplicationEventPublisher eventPublisher;
    private final UserRepository userRepository;
    
    public User registerUser(RegisterUserRequest request) {
        // 핵심 로직: 사용자 생성
        User user = User.builder()
            .email(request.getEmail())
            .name(request.getName())
            .build();
        
        User savedUser = userRepository.save(user);
        
        // 이벤트 발행 (부가 작업들을 비동기로 처리)
        UserRegisteredEvent event = new UserRegisteredEvent(
            savedUser.getId(),
            savedUser.getEmail(), 
            savedUser.getName(),
            LocalDateTime.now()
        );
        
        eventPublisher.publishEvent(event);
        
        return savedUser;
    }
}
```

#### 이벤트 리스너

```java
@Component
public class UserEventListener {
    
    private final EmailService emailService;
    private final SmsService smsService;
    private final CouponService couponService;
    
    private static final Logger log = LoggerFactory.getLogger(UserEventListener.class);
    
    @EventListener
    @Async("taskExecutor")
    public void handleUserRegistered(UserRegisteredEvent event) {
        log.info("사용자 등록 이벤트 처리 시작: {}", event.getUserId());
        
        try {
            // 여러 작업을 병렬로 처리
            CompletableFuture<Void> emailTask = CompletableFuture.runAsync(() -> 
                emailService.sendWelcomeEmail(event.getEmail())
            );
            
            CompletableFuture<Void> smsTask = CompletableFuture.runAsync(() -> 
                smsService.sendWelcomeSms(event.getName())
            );
            
            CompletableFuture<Void> couponTask = CompletableFuture.runAsync(() -> 
                couponService.issueWelcomeCoupon(event.getUserId())
            );
            
            // 모든 작업 완료 대기
            CompletableFuture.allOf(emailTask, smsTask, couponTask).join();
            
            log.info("사용자 등록 후속 작업 완료: {}", event.getUserId());
            
        } catch (Exception e) {
            log.error("사용자 등록 후속 작업 실패: {}", event.getUserId(), e);
        }
    }
}
```

### 5.5 주의사항

#### 1) 자기 호출 문제

```java
// 잘못된 예시 - 같은 클래스 내에서 @Async 메서드 호출
@Service
public class OrderService {
    
    public void createOrder() {
        // 이 호출은 비동기로 동작하지 않음!
        this.sendNotificationAsync();
    }
    
    @Async
    public void sendNotificationAsync() {
        // 동기적으로 실행됨
    }
}

// 올바른 예시 - 다른 서비스에서 호출
@Service 
public class NotificationService {
    
    @Async
    public void sendNotificationAsync() {
        // 정상적으로 비동기 실행됨
    }
}

@Service
public class OrderService {
    
    private final NotificationService notificationService;
    
    public void createOrder() {
        // 다른 빈에서 호출하므로 비동기로 동작함
        notificationService.sendNotificationAsync();
    }
}
```

#### 2) 반환 타입 제한

```java
@Service
public class AsyncService {
    
    // 올바른 반환 타입들
    @Async
    public void methodWithVoid() { }
    
    @Async
    public CompletableFuture<String> methodWithCompletableFuture() {
        return CompletableFuture.completedFuture("결과");
    }
    
    @Async
    public Future<String> methodWithFuture() {
        return new AsyncResult<>("결과");
    }
    
    // 잘못된 반환 타입 - 비동기 효과 없음
    @Async
    public String methodWithString() {
        return "결과"; // 동기적으로 실행됨
    }
}
```

## 6. 비동기 연동 적용 사례

### 6.1 회원가입 프로세스

#### Before: 동기 처리 방식

```java
@Service
public class UserRegistrationService {
    
    @Transactional
    public void registerUser(RegisterUserRequest request) {
        // 1. 사용자 저장 (200ms)
        User user = createAndSaveUser(request);
        
        // 2. 웰컴 이메일 발송 (2000ms)
        emailService.sendWelcomeEmail(user.getEmail());
        
        // 3. SMS 발송 (1500ms)  
        smsService.sendWelcomeSms(user.getPhoneNumber());
        
        // 4. 웰컴 쿠폰 발급 (300ms)
        couponService.issueWelcomeCoupon(user.getId());
        
        // 5. 추천인 포인트 지급 (100ms)
        if (request.getReferrerId() != null) {
            pointService.giveReferralPoints(request.getReferrerId());
        }
        
        // 총 소요 시간: 약 4100ms
        // 사용자는 4초 이상 기다려야 함
    }
}
```

#### After: 비동기 처리 방식

```java
@Service
public class UserRegistrationService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public void registerUser(RegisterUserRequest request) {
        // 1. 핵심 로직만 동기 처리 (200ms)
        User user = createAndSaveUser(request);
        
        // 2. 후속 처리는 이벤트로 위임
        UserRegisteredEvent event = new UserRegisteredEvent(
            user.getId(), 
            user.getEmail(), 
            user.getPhoneNumber(),
            request.getReferrerId()
        );
        
        eventPublisher.publishEvent(event);
        
        // 총 소요 시간: 약 200ms (95% 단축!)
        // 사용자는 즉시 응답받고 다른 작업 가능
    }
}

@Component
public class UserRegistrationEventHandler {
    
    @EventListener
    @Async("taskExecutor")
    public void handleUserRegistered(UserRegisteredEvent event) {
        // 모든 후속 작업을 병렬로 처리
        List<CompletableFuture<Void>> tasks = Arrays.asList(
            CompletableFuture.runAsync(() -> 
                emailService.sendWelcomeEmail(event.getEmail())
            ),
            CompletableFuture.runAsync(() -> 
                smsService.sendWelcomeSms(event.getPhoneNumber())
            ),
            CompletableFuture.runAsync(() -> 
                couponService.issueWelcomeCoupon(event.getUserId())
            ),
            CompletableFuture.runAsync(() -> {
                if (event.getReferrerId() != null) {
                    pointService.giveReferralPoints(event.getReferrerId());
                }
            })
        );
        
        // 모든 작업 완료 대기
        CompletableFuture.allOf(tasks.toArray(new CompletableFuture[0])).join();
        
        log.info("사용자 등록 후속 처리 완료: {}", event.getUserId());
    }
}
```

### 6.2 주문 처리 시스템

```java
@Service
@Transactional
public class OrderService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    public Order processOrder(CreateOrderRequest request) {
        // 1. 핵심 비즈니스 로직 (동기 처리 - 실패 시 롤백되어야 함)
        validateOrderRequest(request);
        
        Order order = createOrder(request);
        
        // 재고 확인 및 차감 (실패 시 주문 실패되어야 함)
        inventoryService.reserveItems(order.getItems());
        
        // 결제 처리 (실패 시 주문 실패되어야 함)
        Payment payment = paymentService.processPayment(request.getPaymentInfo());
        order.setPayment(payment);
        
        Order savedOrder = orderRepository.save(order);
        
        // 2. 후속 처리 (비동기 - 실패해도 주문에 영향 없음)
        OrderCreatedEvent event = new OrderCreatedEvent(
            savedOrder.getId(),
            savedOrder.getUserId(), 
            savedOrder.getTotalAmount()
        );
        
        eventPublisher.publishEvent(event);
        
        return savedOrder;
    }
}

@Component
public class OrderEventHandler {
    
    @EventListener
    @Async("taskExecutor")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 주문 후속 처리들을 병렬 실행
        CompletableFuture<Void> emailTask = CompletableFuture.runAsync(() -> 
            emailService.sendOrderConfirmation(event.getOrderId())
        );
        
        CompletableFuture<Void> smsTask = CompletableFuture.runAsync(() -> 
            smsService.sendOrderNotification(event.getOrderId())
        );
        
        CompletableFuture<Void> loyaltyTask = CompletableFuture.runAsync(() -> 
            loyaltyService.awardPoints(event.getUserId(), event.getTotalAmount())
        );
        
        CompletableFuture<Void> analyticsTask = CompletableFuture.runAsync(() -> 
            analyticsService.recordOrderEvent(event)
        );
        
        // 모든 후속 처리 완료 대기
        CompletableFuture.allOf(emailTask, smsTask, loyaltyTask, analyticsTask)
            .thenRun(() -> log.info("주문 후속 처리 완료: {}", event.getOrderId()))
            .exceptionally(throwable -> {
                log.error("주문 후속 처리 중 오류: {}", event.getOrderId(), throwable);
                return null;
            });
    }
}
```

### 6.3 대용량 데이터 처리

```java
@Service
public class UserDataExportService {
    
    public void exportUserData(Long userId) {
        // 1. 여러 소스에서 사용자 데이터 병렬 수집
        CompletableFuture<UserProfile> profileFuture = 
            CompletableFuture.supplyAsync(() -> userService.getUserProfile(userId));
            
        CompletableFuture<List<Order>> ordersFuture = 
            CompletableFuture.supplyAsync(() -> orderService.getUserOrders(userId));
            
        CompletableFuture<List<Review>> reviewsFuture = 
            CompletableFuture.supplyAsync(() -> reviewService.getUserReviews(userId));
            
        CompletableFuture<PaymentInfo> paymentFuture = 
            CompletableFuture.supplyAsync(() -> paymentService.getPaymentInfo(userId));
        
        // 2. 모든 데이터 수집 완료 후 결합
        CompletableFuture.allOf(profileFuture, ordersFuture, reviewsFuture, paymentFuture)
            .thenRun(() -> {
                try {
                    UserExportData exportData = UserExportData.builder()
                        .profile(profileFuture.get())
                        .orders(ordersFuture.get())
                        .reviews(reviewsFuture.get())
                        .paymentInfo(paymentFuture.get())
                        .build();
                    
                    // 3. 파일 생성 및 이메일 발송 (비동기)
                    generateAndSendExportFile(userId, exportData);
                    
                } catch (Exception e) {
                    log.error("사용자 데이터 내보내기 실패: {}", userId, e);
                }
            });
    }
    
    @Async
    private void generateAndSendExportFile(Long userId, UserExportData data) {
        try {
            // 엑셀 파일 생성
            String filePath = excelService.createUserDataFile(data);
            
            // 이메일 발송  
            emailService.sendExportCompletionEmail(userId, filePath);
            
            log.info("사용자 데이터 내보내기 완료: {}", userId);
        } catch (Exception e) {
            log.error("파일 생성 또는 이메일 발송 실패: {}", userId, e);
        }
    }
}
```

### 6.4 실시간 알림 시스템

```java
@RestController
public class NotificationController {
    
    private final NotificationService notificationService;
    
    @PostMapping("/notifications/send")
    public ResponseEntity<String> sendNotification(@RequestBody NotificationRequest request) {
        
        // 즉시 응답 반환
        String notificationId = UUID.randomUUID().toString();
        
        // 알림 발송은 비동기로 처리
        notificationService.sendNotificationAsync(notificationId, request);
        
        return ResponseEntity.ok("알림 발송이 요청되었습니다. ID: " + notificationId);
    }
}

@Service
public class NotificationService {
    
    @Async("taskExecutor")
    public void sendNotificationAsync(String notificationId, NotificationRequest request) {
        log.info("알림 발송 시작: {}", notificationId);
        
        List<CompletableFuture<Void>> tasks = new ArrayList<>();
        
        // 다양한 채널로 동시 발송
        if (request.isSendEmail()) {
            tasks.add(CompletableFuture.runAsync(() -> 
                emailService.sendEmail(request.getEmail(), request.getTitle(), request.getContent())
            ));
        }
        
        if (request.isSendSms()) {
            tasks.add(CompletableFuture.runAsync(() -> 
                smsService.sendSms(request.getPhoneNumber(), request.getContent())
            ));
        }
        
        if (request.isSendPush()) {
            tasks.add(CompletableFuture.runAsync(() -> 
                pushService.sendPush(request.getUserId(), request.getTitle(), request.getContent())
            ));
        }
        
        // 모든 채널 발송 완료 후 결과 로깅
        CompletableFuture.allOf(tasks.toArray(new CompletableFuture[0]))
            .thenRun(() -> log.info("알림 발송 완료: {}", notificationId))
            .exceptionally(throwable -> {
                log.error("알림 발송 실패: {}", notificationId, throwable);
                return null;
            });
    }
}
```

## 참고 자료

1. [Spring Boot @Async 어떻게 동작하는가?, 2020.05.23](https://brunch.co.kr/@2MrI/401)
2. [Java CompletableFuture로 비동기 적용하기, 11번가 기술블로그, 2024.01.04](https://11st-tech.github.io/2024/01/04/completablefuture/)
3. [스프링과 CompletableFuture를 활용한 비동기 처리 방법, F-Lab](https://f-lab.kr/insight/spring-and-completablefuture-for-asynchronous-processing)
4. [이메일 비동기 전송 관련 CompletableFuture 와 Async 의 장점을 누리려면, 2023](https://velog.io/@qkrtkdwns3410/이메일-비동기-전송-관련-CompletableFuture-와-Async-의-장점을-누리려면)
5. [스프링 이벤트를 활용해 로직간 강결합을 해결하는 방법, 2023](https://velog.io/@eastperson/스프링-이벤트를-활용해-로직간-강결합을-해결하는-방법)
6. [예외 먹는 @TransactionalEventListener, 렌딧 기술블로그, 2022.08.11](https://lenditkr.github.io/spring/transactional-event-listener/)
7. [Spring Events, 2023](https://velog.io/@ljinsk3/Spring-Events)
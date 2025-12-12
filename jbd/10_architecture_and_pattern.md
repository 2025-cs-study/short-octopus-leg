# 11 | 자주 쓰는 서버 구조와 설계 패턴

## 1. 3계층 아키텍처

대부분의 웹 애플리케이션은 **3계층 아키텍처**(3-Tier Architecture)를 따라 설계됨.

### 3계층이란?

<img width="936" height="556" alt="image" src="https://github.com/user-attachments/assets/fcd70ff9-79d6-4bfb-8265-852cc314edca" />

*출처: [AWS Architecture Blog - Building a three-tier architecture on a budget](https://aws.amazon.com/ko/blogs/architecture/building-a-three-tier-architecture-on-a-budget/)*

애플리케이션을 역할에 따라 3개의 계층으로 분리하는 설계 방식임.

1. **프레젠테이션 계층(Presentation Layer)**: 사용자 인터페이스를 담당함. 사용자의 요청을 받고 결과를 표시함.
2. **비즈니스 로직 계층(Business Logic Layer)**: 핵심 비즈니스 로직을 처리함. 데이터 검증, 계산, 의사결정 등을 수행함.
3. **데이터 접근 계층(Data Access Layer)**: 데이터베이스와의 통신을 담당함. CRUD 작업을 수행함.

**장점**:
- 각 계층의 독립적인 개발과 유지보수가 가능함.
- 특정 계층의 변경이 다른 계층에 미치는 영향을 최소화함.
- 테스트가 용이함.
- 코드 재사용성이 높아짐.

**단점**:
- 초기 설계가 복잡함.
- 작은 프로젝트에서는 과도한 구조일 수 있음.

```python
# 3계층 아키텍처 예시 (Python/Flask)

# 1. 프레젠테이션 계층 (Controller)
@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = user_service.get_user_by_id(user_id)
    return jsonify(user), 200

# 2. 비즈니스 로직 계층 (Service)
class UserService:
    def get_user_by_id(self, user_id):
        user = user_repository.find_by_id(user_id)
        if not user:
            raise UserNotFoundException()
        return self._to_dto(user)

# 3. 데이터 접근 계층 (Repository)
class UserRepository:
    def find_by_id(self, user_id):
        return db.session.query(User).filter_by(id=user_id).first()
```

## 2. MVC 패턴

<img width="801" height="401" alt="image" src="https://github.com/user-attachments/assets/cadf7915-f318-4e58-994b-d26c28803195" />

*출처: [GeeksforGeeks - MVC Design Pattern](https://www.geeksforgeeks.org/system-design/mvc-design-pattern/)*

**MVC 패턴**은 애플리케이션을 Model, View, Controller로 분리하는 디자인 패턴임.

### MVC 구성 요소

- **Model**: 데이터와 비즈니스 로직을 담당함. 데이터의 상태를 관리하고 변경을 알림.
- **View**: 사용자에게 보이는 화면을 담당함. Model의 데이터를 시각화함.
- **Controller**: 사용자 입력을 처리하고 Model과 View를 연결함. 요청을 받아 적절한 Model을 호출하고 View를 선택함.

**장점**:
- 관심사의 분리로 코드 구조가 명확함.
- 각 컴포넌트의 독립적인 개발이 가능함.
- 테스트와 유지보수가 쉬움.

**단점**:
- 작은 애플리케이션에서는 복잡도가 증가함.
- Controller가 비대해질 수 있음.

```java
// MVC 패턴 예시 (Java/Spring)

// Model
@Entity
public class Product {
    @Id
    private Long id;
    private String name;
    private int price;
    // getters, setters
}

// Controller
@RestController
@RequestMapping("/products")
public class ProductController {
    @Autowired
    private ProductService productService;
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable Long id) {
        ProductDto product = productService.findById(id);
        return ResponseEntity.ok(product);
    }
}

// View (JSON 응답)
{
    "id": 1,
    "name": "노트북",
    "price": 1500000
}
```

## 3. 마이크로서비스 아키텍처

<img width="512" height="411" alt="image" src="https://github.com/user-attachments/assets/450da8d8-ab87-46a2-b998-bd5fde2677a4" />

*출처: [Uber Engineering Blog - Introducing Domain-Oriented Microservice Architecture](https://www.uber.com/en-KR/blog/microservice-architecture/)*

**마이크로서비스 아키텍처**는 하나의 큰 애플리케이션을 여러 개의 작은 서비스로 분리하는 설계 방식임.

### 특징

- **서비스 독립성**: 각 서비스는 독립적으로 개발, 배포, 확장됨.
- **데이터베이스 분리**: 각 서비스가 자체 데이터베이스를 가짐.
- **API 통신**: 서비스 간 HTTP API나 메시지 큐로 통신함.
- **기술 스택 자유**: 각 서비스마다 다른 언어와 프레임워크 사용 가능함.

**장점**:
- 서비스별 독립적인 배포와 확장이 가능함.
- 장애가 특정 서비스에 국한됨.
- 팀별로 독립적인 개발이 가능함.
- 기술 선택의 자유도가 높음.

**단점**:
- 시스템 복잡도가 크게 증가함.
- 서비스 간 통신으로 인한 네트워크 오버헤드 발생함.
- 분산 시스템의 디버깅과 모니터링이 어려움.
- 데이터 일관성 관리가 복잡함.

**사용 사례**: 대규모 트래픽을 처리하는 서비스, 빠른 기능 개발이 필요한 서비스, 여러 팀이 협업하는 조직

### 모놀리식 vs 마이크로서비스

- **모놀리식**: 모든 기능이 하나의 애플리케이션에 통합됨. 작은 팀과 초기 단계에 적합함.
- **마이크로서비스**: 기능별로 서비스를 분리함. 대규모 조직과 복잡한 시스템에 적합함.

## 4. API 게이트웨이

<img width="2294" height="1331" alt="image" src="https://github.com/user-attachments/assets/43723a05-5e80-496f-a451-7d31094a5ea1" />

*출처: [Microsoft Learn - The API gateway pattern versus the direct client-to-microservice communication](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/direct-client-to-microservice-communication-versus-the-api-gateway-pattern)*

**API 게이트웨이**는 클라이언트와 백엔드 서비스 사이에 위치한 단일 진입점임.

### 주요 기능

- **라우팅**: 클라이언트 요청을 적절한 서비스로 전달함.
- **인증/인가**: 요청의 권한을 검증함.
- **속도 제한**: API 호출 횟수를 제한함.
- **로드 밸런싱**: 여러 서버로 요청을 분산함.
- **캐싱**: 자주 요청되는 데이터를 캐시로 제공함.
- **로깅/모니터링**: 모든 요청을 기록하고 모니터링함.

**장점**:
- 클라이언트가 여러 서비스를 직접 호출할 필요가 없음.
- 공통 기능을 한 곳에서 관리함.
- 보안과 모니터링이 용이함.

**단점**:
- 단일 장애점이 될 수 있음.
- 추가적인 네트워크 홉으로 지연 시간 증가함.
- API 게이트웨이 자체가 병목이 될 수 있음.

**구현 기술**: AWS API Gateway, Kong, Nginx, Spring Cloud Gateway 등

## 5. 이벤트 기반 아키텍처


<img width="758" height="292" alt="image" src="https://github.com/user-attachments/assets/631683f6-1ecd-470b-aec0-e7d48723cb0a" />

*출처: [Microsoft Learn - Event-driven architecture style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)*

**이벤트 기반 아키텍처**(Event-Driven Architecture)는 이벤트를 중심으로 시스템이 동작하는 설계 방식임.

### 특징

- **이벤트 발행**: 특정 상황이 발생하면 이벤트를 생성함.
- **비동기 처리**: 이벤트 발행자는 결과를 기다리지 않음.
- **느슨한 결합**: 서비스 간 직접적인 의존성이 없음.

**장점**:
- 서비스 간 결합도가 낮아짐.
- 확장성이 뛰어남.
- 실시간 처리가 가능함.
- 시스템 유연성이 높아짐.

**단점**:
- 디버깅과 추적이 어려움.
- 이벤트 순서 보장이 어려울 수 있음.
- 데이터 일관성 관리가 복잡함.

**사용 사례**: 주문 처리 시스템, 실시간 알림, IoT 데이터 처리

**구현 기술**: Apache Kafka, RabbitMQ, AWS SQS, Redis Pub/Sub 등

```javascript
// 이벤트 기반 아키텍처 예시 (Node.js)

// 이벤트 발행자 (주문 서비스)
class OrderService {
    async createOrder(orderData) {
        const order = await orderRepository.save(orderData);
        
        // 주문 생성 이벤트 발행
        eventBus.publish('order.created', {
            orderId: order.id,
            userId: order.userId,
            amount: order.amount
        });
        
        return order;
    }
}

// 이벤트 구독자 (재고 서비스)
eventBus.subscribe('order.created', async (event) => {
    await inventoryService.reduceStock(event.orderId);
});

// 이벤트 구독자 (알림 서비스)
eventBus.subscribe('order.created', async (event) => {
    await notificationService.sendOrderConfirmation(event.userId);
});
```

## 6. 서킷 브레이커 패턴

<img width="1196" height="632" alt="image" src="https://github.com/user-attachments/assets/02dbadd7-2585-4a69-b073-551a2b51e878" />

*출처: [Microsoft Azure - Circuit Breaker Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)*

**서킷 브레이커**(Circuit Breaker)는 장애가 발생한 서비스로의 호출을 차단하여 시스템을 보호하는 패턴임.

### 동작 방식

1. **Closed 상태**: 정상 동작. 모든 요청을 전달함.
2. **Open 상태**: 오류 임계값 초과 시. 요청을 즉시 실패 처리함.
3. **Half-Open 상태**: 일정 시간 후. 일부 요청을 허용하여 복구 여부를 확인함.

**장점**:
- 장애 전파를 방지함.
- 빠른 실패로 사용자 경험 개선함.
- 장애 서비스에 불필요한 부하를 주지 않음.

**단점**:
- 구현이 복잡함.
- 임계값 설정이 어려움.

**구현 기술**: Netflix Hystrix, Resilience4j, Polly 등

## 7. 그 외 알아두면 좋은 패턴

### CQRS (Command Query Responsibility Segregation)
- 명령(쓰기)과 조회(읽기)를 분리하는 패턴
- 예) 읽기/쓰기 비율이 크게 다른 시스템 (예: 대시보드, 분석 시스템)

### 사가 패턴 (Saga Pattern)
- 분산 트랜잭션을 여러 개의 로컬 트랜잭션으로 분할하여 관리
- 예) 마이크로서비스 환경에서 여러 서비스에 걸친 트랜잭션 처리

### BFF (Backend for Frontend)
- 각 프론트엔드(웹, 모바일 등)를 위한 전용 백엔드 API 제공
- 예) 다양한 클라이언트 타입이 존재하고 각각 다른 데이터 요구사항이 있을 때

### 사이드카 패턴 (Sidecar Pattern)
- 메인 애플리케이션 옆에 보조 컨테이너를 배치하여 공통 기능 처리
- 예) Kubernetes 환경에서 로깅, 모니터링, 보안 등 공통 관심사 분리

### 스트랭글러 피그 패턴 (Strangler Fig Pattern)
- 레거시 시스템을 점진적으로 새 시스템으로 마이그레이션
- 예) 모놀리식 → 마이크로서비스 전환, 대규모 시스템 리팩토링

### 서버리스 아키텍처 (Serverless Architecture)
- 서버 관리 없이 이벤트 기반으로 코드 실행 (AWS Lambda, Azure Functions)
- 예) 간헐적 워크로드, 빠른 프로토타이핑, 이벤트 처리

## 참고 자료

1. [Microsoft, "아키텍처 스타일", Azure Architecture Center](https://learn.microsoft.com/ko-kr/azure/architecture/guide/architecture-styles/)
2. [AWS, "아키텍처 센터", AWS Architecture Center](https://aws.amazon.com/ko/architecture/)
3. [Microsoft, "클라우드 디자인 패턴", Azure Architecture Center](https://learn.microsoft.com/ko-kr/azure/architecture/patterns/)
4. [GeeksforGeeks, "System Design"](https://www.geeksforgeeks.org/system-design-tutorial/)
5. [ByteByteGo, "System Design 101"](https://blog.bytebytego.com/)
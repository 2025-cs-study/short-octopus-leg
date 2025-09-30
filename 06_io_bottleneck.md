# 07 | IO 병목, 어떻게 해결하지

## 1. IO 병목이란?

**IO(Input/Output) 병목**은 데이터를 읽고 쓰는 작업에서 발생하는 성능 저하를 말함. CPU는 빠른데 데이터를 주고받는 속도가 느려서 전체 시스템이 느려지는 현상을 말함.

### 일상생활 예: 고속도로 톨게이트
```
- 차량(데이터)은 빠르게 달릴 수 있음
- 하지만 톨게이트(IO)에서 병목이 발생함
- 톨게이트가 1개면 → 긴 대기 시간
- 톨게이트를 늘리면 → 대기 시간 감소
```

### IO 병목 발생 지점

| 지점 | 예시 | 응답 시간 |
|------|------|-------------------|
| **캐시** | Redis 조회 | 1ms ~ 5ms |
| **데이터베이스** | SELECT 쿼리 | 10ms ~ 100ms |
| **파일 시스템** | 파일 읽기/쓰기 | 10ms ~ 500ms |
| **외부 API** | HTTP 요청 | 100ms ~ 5s |
| **네트워크** | 원격 서버 통신 | 50ms ~ 2s |

> **참고**: 위 시간은 실제 환경에 따라 달라질 수 있음.

## 2. IO 병목 문제

```java
@RestController
public class ProductController {

    @GetMapping("/products/{id}")
    public ResponseEntity<ProductDetail> getProduct(@PathVariable Long id) {
        long startTime = System.currentTimeMillis();

        // 1. DB에서 상품 기본 정보 조회 (이름, 가격, 설명 등)
        Product product = productRepository.findById(id);

        // 2. DB에서 해당 상품의 고객 리뷰 조회 (별점, 댓글 등)
        List<Review> reviews = reviewRepository.findByProductId(id);

        // 3. 외부 재고 관리 API를 호출해서 현재 재고 수량 확인 (다른 회사 시스템이나 물류센터 시스템과 통신)
        Stock stock = externalApi.getStock(product.getSkuCode());

        // 4. 외부 배송 API를 호출해서 예상 배송일 조회 (택배사 시스템과 통신)
        Delivery delivery = externalApi.getDeliveryInfo(product.getId());

        // 5. DB에서 이 상품과 관련된 추천 상품 목록 조회 (함께 보면 좋은 상품, 비슷한 상품 등)
        List<Product> related = productRepository.findRelatedProducts(id);

        long endTime = System.currentTimeMillis();
        log.info("순차 처리 총 소요 시간: {}ms", endTime - startTime);

        return ResponseEntity.ok(new ProductDetail(product, reviews, stock, delivery, related));
    }
}
```

**문제점**
- 각 IO 작업이 순차적으로 실행됨. (1번 끝나야 2번 시작, 2번 끝나야 3번 시작...)
- 모든 작업의 시간이 누적됨. (T1 + T2 + T3 + T4 + T5)
- 예: 각 작업이 100ms씩 걸리면 총 500ms 소요
- 트래픽이 증가하면 응답 시간이 더 길어짐.

## 3. IO 병목 문제 해결 방법

**SKU (Stock Keeping Unit) 코드**
- 상품을 구별하기 위한 고유 식별 코드
- 예: "SHOES-NI-AIR-BLK-270" (나이키 에어맥스 270 검정색)
- 재고 관리, 주문 처리 등에서 상품을 정확히 식별하는 데 사용됨.

**TTL (Time To Live)**
- 데이터가 캐시에 유효하게 저장되는 시간
- 예: 5분 TTL = 데이터를 5분간 캐시에 보관하고, 5분 후 자동 삭제
- 시간이 지나면 최신 데이터를 다시 조회하게 됨.

### 3.1 비동기 병렬 처리

```java
@RestController
public class ProductController {

    @GetMapping("/products/{id}")
    public ResponseEntity<ProductDetail> getProduct(@PathVariable Long id) {
        long startTime = System.currentTimeMillis();

        // 상품 정보를 가장 먼저 조회 시작
        CompletableFuture<Product> productFuture =
                CompletableFuture.supplyAsync(() -> productRepository.findById(id));

        // 리뷰 조회 (상품 ID로 바로 조회 가능)
        CompletableFuture<List<Review>> reviewsFuture =
                CompletableFuture.supplyAsync(() -> reviewRepository.findByProductId(id));

        // 배송 정보 조회 (상품 ID로 바로 조회 가능)
        CompletableFuture<Delivery> deliveryFuture =
                CompletableFuture.supplyAsync(() -> externalApi.getDeliveryInfo(id));

        // 연관 상품 조회 (상품 ID로 바로 조회 가능)
        CompletableFuture<List<Product>> relatedFuture =
                CompletableFuture.supplyAsync(() -> productRepository.findRelatedProducts(id));

        // 재고 조회는 상품 정보의 SKU 코드가 필요하므로 상품 조회가 완료된 후에 실행 (thenApplyAsync 사용)
        CompletableFuture<Stock> stockFuture = productFuture.thenApplyAsync(
                product -> externalApi.getStock(product.getSkuCode())
        );

        // 모든 작업이 완료될 때까지 대기 (join(): 모든 작업이 끝날 때까지 기다리는 메서드)
        CompletableFuture.allOf(
                productFuture, reviewsFuture, stockFuture, deliveryFuture, relatedFuture
        ).join();

        long endTime = System.currentTimeMillis();
        log.info("병렬 처리 총 소요 시간: {}ms", endTime - startTime);

        // 총 소요 시간은 가장 오래 걸린 작업의 시간과 비슷함
        // 예: 상품 조회(100ms) + 재고 조회(100ms) = 약 200ms
        return ResponseEntity.ok(new ProductDetail(
                productFuture.join(),      // 완료된 상품 정보 가져오기
                reviewsFuture.join(),      // 완료된 리뷰 가져오기
                stockFuture.join(),        // 완료된 재고 정보 가져오기
                deliveryFuture.join(),     // 완료된 배송 정보 가져오기
                relatedFuture.join()       // 완료된 연관 상품 가져오기
        ));
    }
}
```

- 독립적인 작업들이 동시에 실행됨
- 총 소요 시간 = 가장 긴 작업 경로의 시간
- 일반적으로 40~60% 성능 향상

### 3.2 캐싱 전략

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    // Before: 매번 DB에서 조회 (느림)
    public Product getProduct(Long id) {
        return productRepository.findById(id);
    }

    // After: 캐시 적용 (자주 조회되는 데이터를 메모리에 저장)
    @Cacheable(value = "products", key = "#id")
    public Product getProductWithCache(Long id) {
        // 첫 조회: DB에서 가져와서 캐시에 저장 (약 10~100ms)
        // 이후 조회: 캐시에서 바로 반환 (약 1~5ms)
        return productRepository.findById(id);
    }

    // 상품 정보 수정 시: 캐시 갱신
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    // 상품 삭제 시: 캐시 제거
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
}
```

**application.yml 캐시 설정**
```yaml
spring:
  cache:
    type: redis  # Redis를 캐시 저장소로 사용
    redis:
      time-to-live: 300000  # 5분 TTL
```

- 캐시 히트 시: 메모리(Redis)에서 읽기 → 약 1~5ms
- 캐시 미스 시: DB 조회 후 캐시에 저장 → 약 10~100ms
- 반복 조회가 많은 데이터에 효과적 (인기 상품, 설정값 등)
- TTL 설정으로 오래된 데이터 방지 (5분 후 자동 삭제)

### 3.3 커넥션 풀 최적화

```java
@Configuration
public class DataSourceConfig {

    // Before: 기본 설정 (커넥션 10개 사용)
    public DataSource defaultDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("user");
        config.setPassword("password");
        // 커넥션이 부족하면 대기 발생
        return new HikariDataSource(config);
    }

    // After: 트래픽에 맞춰 최적화된 설정
    @Bean
    public DataSource optimizedDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("user");
        config.setPassword("password");

        // 커넥션 풀 크기 설정
        config.setMinimumIdle(10); // 항상 10개의 커넥션 준비

        config.setMaximumPoolSize(50); // 최대 50개까지 커넥션 생성 가능

        config.setConnectionTimeout(20000); // 커넥션을 못 받으면 20초 대기 후 에러 발생

        config.setIdleTimeout(300000); // 사용하지 않는 커넥션은 5분 후 제거

        config.setMaxLifetime(1200000); // 커넥션 최대 생존 시간 20분

        // DB 쿼리 성능 최적화 옵션
        config.addDataSourceProperty("cachePrepStmts", "true"); // SQL 쿼리 캐싱 활성화

        config.addDataSourceProperty("prepStmtCacheSize", "250"); // 최대 250개 쿼리 캐시

        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048"); // 2048자 이하 쿼리만 캐시

        return new HikariDataSource(config);
    }
}
```

### 3.4 배치 처리

```java
@Service
public class OrderService {

    // Before: 주문을 하나씩 개별 처리
    public void processOrdersOneByOne(List<Order> orders) {
        long startTime = System.currentTimeMillis();

        for (Order order : orders) {
            // 주문 1개마다 DB에 1번씩 저장
            // 예: 100개 주문 = 100번 DB 통신
            orderRepository.save(order);
        }

        long endTime = System.currentTimeMillis();
        log.info("개별 처리 시간: {}ms, 주문 수: {}",
                endTime - startTime, orders.size());
        // 예상 시간: 100개 × 10ms = 1000ms (1초)
    }

    // After: 주문을 모아서 배치로 처리
    public void processOrdersInBatch(List<Order> orders) {
        long startTime = System.currentTimeMillis();

        int batchSize = 100;  // 100개씩 묶어서 처리
        List<Order> batch = new ArrayList<>();

        for (Order order : orders) {
            batch.add(order);  // 주문을 임시 목록에 추가

            // 100개가 모이면 한 번에 DB에 저장
            if (batch.size() >= batchSize) {
                orderRepository.saveAll(batch);  // 1번의 DB 통신으로 100개 저장
                batch.clear();  // 목록 비우기
            }
        }

        // 남은 주문들 처리 (100개 미만)
        // 예: 250개 주문이면 100개, 100개, 50개로 3번 저장
        if (!batch.isEmpty()) {
            orderRepository.saveAll(batch);
        }

        long endTime = System.currentTimeMillis();
        log.info("배치 처리 시간: {}ms, 주문 수: {}", endTime - startTime, orders.size());
        // 예상 시간: (100개 ÷ 100) × 10ms = 10ms (0.01초)
    }
}
```

**application.yml 배치 설정**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 100 # 한 번에 처리할 데이터 개수 (100개씩 묶음)

        order_inserts: true # INSERT 쿼리를 정렬해서 더 효율적으로 실행

        order_updates: true # UPDATE 쿼리도 정렬해서 효율적으로 실행

        batch_versioned_data: true # 버전 관리하는 데이터도 배치로 처리 가능하게 설정
```

- 1000개 항목 개별 처리: 1000번의 DB 호출 = 10초
- 1000개 항목 배치 처리 (100개씩): 10번의 DB 호출 = 0.1초
- 일반적으로 80~90% 성능 향상

## 4. 체크리스트

IO 병목을 해결하기 위해 점검할 항목들

- [ ] **비동기 처리**: 독립적인 IO 작업들을 병렬로 실행하고 있는가?
- [ ] **캐싱**: 자주 조회되고 변경이 적은 데이터를 캐싱하고 있는가?
- [ ] **커넥션 풀**: 적절한 크기로 설정되어 있는가?
- [ ] **배치 처리**: 대량 데이터 처리 시 배치로 처리하고 있는가?
- [ ] **쿼리 최적화**: N+1 문제가 없는가? 필요한 인덱스가 있는가?
- [ ] **페이지네이션**: 대량 데이터 조회 시 페이징을 적용하고 있는가?
- [ ] **모니터링**: 성능 지표를 측정하고 있는가?
- [ ] **타임아웃**: 적절한 타임아웃이 설정되어 있는가?

## 참고 자료

1. [HikariCP Dead lock에서 벗어나기, 우아한형제들 기술블로그, 2020](https://techblog.woowahan.com/2664)
2. [JPA N+1 문제와 해결방법, 2023](https://incheol-jung.gitbook.io/docs/q-and-a/spring/n+1)
3. [Spring Batch 성능 최적화, 우아한형제들 기술블로그, 2019](https://techblog.woowahan.com/2662)
4. [CompletableFuture를 이용한 비동기 프로그래밍, 2024](https://11st-tech.github.io/2024/01/04/completablefuture)
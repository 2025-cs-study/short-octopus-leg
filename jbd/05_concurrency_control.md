# 06 | 동시성, 데이터가 꼬이기 전에 잡아야 한다

## 1. 동시성 문제란?

**동시성 문제**는 여러 사용자가 동시에 같은 데이터를 수정할 때 발생하는 예측할 수 없는 결과가 발생하거나 데이터 무결성이 깨지는 현상을 말함.

### 일상생활 예: 쇼핑몰 재고 관리
```
상황: 재고 10개인 상품에 동시에 15개 주문이 들어옴
예상 결과: 10개만 주문 성공, 5개는 재고 부족으로 실패
실제 결과: 15개 모두 주문 성공, 재고 -5개 (데이터 손상)
```

### 언제 발생할까?
- **여러 사용자가 동시에 같은 데이터 수정**할 때
- **재고 관리, 좋아요 수, 조회수** 등의 카운터 업데이트
- **결제, 포인트 적립** 등 금전 관련 처리
- **예약, 선착순 이벤트** 등 한정된 자원에 대한 경쟁

```java
// 문제 발생 코드 - Race Condition 발생
@Service
@Transactional
public class ProductService {
    
    public void purchaseProduct(Long productId, int quantity) {
        // 1. 재고 조회
        Product product = productRepository.findById(productId).orElseThrow();
        
        // 2. 재고 확인
        if (product.getStock() < quantity) {
            throw new OutOfStockException("재고 부족");
        }
        
        // 3. 재고 차감 (동시성 문제 발생)
        product.decreaseStock(quantity);
        productRepository.save(product);
    }
}
```

## 2. 해결책 1: 비관적 락(Pessimistic Lock)

**비관적 락**은 데이터를 읽는 순간부터 락을 걸어서 다른 트랜잭션이 접근하지 못하게 하는 방식임.

```java
// Repository에서 락 설정
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithLock(@Param("id") Long id);
}

// 서비스에서 락 사용
@Service
@Transactional
public class SafeProductService {
    
    public void purchaseProduct(Long productId, int quantity) {
        // 락으로 동시 접근 차단
        Product product = productRepository.findByIdWithLock(productId)
            .orElseThrow();
        
        if (product.getStock() < quantity) {
            throw new OutOfStockException("재고 부족");
        }
        
        // 한 번에 하나의 트랜잭션만 실행됨
        product.decreaseStock(quantity);
        productRepository.save(product);
    }
}
```

- **장점**: 확실한 동시성 제어, 데이터 정합성 보장
- **단점**: 성능 저하, 데드락 위험
- **사용**: 충돌이 빈번하거나 데이터 정합성이 매우 중요한 경우

## 3. 해결책 2: 낙관적 락(Optimistic Lock)

**낙관적 락**은 버전 관리를 통해 업데이트할 때만 충돌을 확인하는 방식임.

```java
// Entity에 버전 필드 추가
@Entity
public class Product {
    private Long id;
    private String name;
    private int stock;
    
    @Version // JPA 낙관적 락을 위한 버전 필드
    private Long version;
    
    public void decreaseStock(int quantity) {
        if (this.stock < quantity) {
            throw new OutOfStockException("재고 부족");
        }
        this.stock -= quantity;
    }
}

// 서비스에서 낙관적 락 처리
@Service
@Transactional
public class OptimisticProductService {
    
    public void purchaseProduct(Long productId, int quantity) {
        try {
            Product product = productRepository.findById(productId).orElseThrow();
            product.decreaseStock(quantity);
            productRepository.save(product);
            
        } catch (OptimisticLockingFailureException e) {
            // 버전 충돌 시 재시도 또는 실패 처리
            throw new ConcurrentModificationException("다시 시도해주세요");
        }
    }
}
```

- **장점**: 좋은 성능, 데드락 없음
- **단점**: 충돌 시 재시도 필요, 복잡한 예외 처리
- **사용**: 충돌이 적고 읽기가 많은 경우

## 4. 해결책 3: 원자적 연산(Atomic Operation)

**원자적 연산**은 SQL 쿼리 한 번으로 조건 확인과 업데이트를 동시에 처리하는 가장 간단하고 효과적인 방법임.

### 구현 방법

```java
// Repository에서 원자적 쿼리 정의
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    
    @Modifying
    @Query("UPDATE Product p SET p.stock = p.stock - :quantity " +
           "WHERE p.id = :id AND p.stock >= :quantity")
    int decreaseStockIfAvailable(@Param("id") Long id, @Param("quantity") int quantity);
}

// 서비스에서 원자적 연산 사용
@Service
@Transactional
public class AtomicProductService {
    
    public void purchaseProduct(Long productId, int quantity) {
        // 원자적 연산으로 재고 차감
        int updatedRows = productRepository.decreaseStockIfAvailable(productId, quantity);
        
        if (updatedRows == 0) {
            throw new OutOfStockException("재고 부족");
        }
        
        // 주문 생성
        createOrder(productId, quantity);
    }
}
```

- **장점**: 가장 빠른 성능, 간단한 구현, 확실한 동시성 제어
- **단점**: 복잡한 비즈니스 로직 적용 어려움
- **사용**: 단순한 카운터 증감 작업

## 5. 적용 사례: 좋아요 기능 구현

### 좋아요 버튼을 연속으로 빠르게 클릭할 때 중복 좋아요가 생성되는 문제 해결 방법

```java
// 유니크 제약조건으로 중복 방지
@Entity
@Table(uniqueConstraints = {
    @UniqueConstraint(columnNames = {"post_id", "user_id"})
})
public class PostLike {
    private Long id;
    private Long postId;
    private Long userId;
    private LocalDateTime createdAt;
}

@Service
@Transactional
public class PostService {
    
    public void likePost(Long postId, Long userId) {
        try {
            PostLike like = new PostLike(postId, userId);
            postLikeRepository.save(like);
            
            // 좋아요 수 원자적 증가
            postRepository.increaseLikeCount(postId);
            
        } catch (DataIntegrityViolationException e) {
            throw new DuplicateLikeException("이미 좋아요를 눌렀습니다");
        }
    }
    
    public void unlikePost(Long postId, Long userId) {
        PostLike like = postLikeRepository.findByPostIdAndUserId(postId, userId)
            .orElseThrow(() -> new LikeNotFoundException());
        
        postLikeRepository.delete(like);
        postRepository.decreaseLikeCount(postId);
    }
}

@Repository
public interface PostRepository extends JpaRepository<Post, Long> {
    
    @Modifying
    @Query("UPDATE Post p SET p.likeCount = p.likeCount + 1 WHERE p.id = :id")
    void increaseLikeCount(@Param("id") Long id);
    
    @Modifying  
    @Query("UPDATE Post p SET p.likeCount = p.likeCount - 1 WHERE p.id = :id")
    void decreaseLikeCount(@Param("id") Long id);
}
```

## 6. 주의사항: 데드락 방지

### 데드락이 발생하는 경우

```java
// 잘못된 예시 - 데드락 발생 가능
@Service
@Transactional
public class BadTransferService {
    
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        // Thread A: Account 1 → Account 2 순서로 락 획득
        // Thread B: Account 2 → Account 1 순서로 락 획득
        // → 데드락 발생!
        
        Account fromAccount = accountRepository.findByIdWithLock(fromAccountId);
        Account toAccount = accountRepository.findByIdWithLock(toAccountId);
        
        fromAccount.withdraw(amount);
        toAccount.deposit(amount);
    }
}
```

### 데드락 방지 방법

```java
// 올바른 예시 - 일관된 락 순서로 데드락 방지
@Service
@Transactional
public class SafeTransferService {
    
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        // ID 순서로 정렬하여 일관된 락 순서 보장
        Long firstId = Math.min(fromAccountId, toAccountId);
        Long secondId = Math.max(fromAccountId, toAccountId);
        
        Account firstAccount = accountRepository.findByIdWithLock(firstId);
        Account secondAccount = accountRepository.findByIdWithLock(secondId);
        
        // 실제 계좌 구분
        Account fromAccount = firstId.equals(fromAccountId) ? firstAccount : secondAccount;
        Account toAccount = firstId.equals(toAccountId) ? firstAccount : secondAccount;
        
        fromAccount.withdraw(amount);
        toAccount.deposit(amount);
    }
}
```

## 참고 자료

1. [JPA 락 사용법과 주의사항, 우아한형제들 기술블로그, 2019.05.30](https://techblog.woowahan.com/2631/)
2. [동시성 제어 기법 총정리, NHN Cloud, 2022.03.17](https://meetup.nhncloud.com/posts/317)
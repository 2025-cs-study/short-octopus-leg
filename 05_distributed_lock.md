# 06 | 동시성, 데이터가 꼬이기 전에 잡아야 한다

### 경쟁 상태
- 여러 스레드가 동시에 공유 자원에 접근할 때, 접근 순서에 따라 결과가 달라지는 상황

## 프로세스 수준에서의 동시 접근 제어
- 잠금(lock)을 사용하는 방법 : 공유 자원에 접근하는 스레드를 한 번에 하나로 제한

### synchronized와 ReentrantLock
- synchronized는 간편하지만 ReentrantLock은 synchronized에는 없는 기능(잠금 획득 대기 시간을 지정) 등을 제공
- 가상 스레드는 자바 24버전부터 synchronized를 지원함
- 두 방식 중 어떤 것을 사용해도 문제는 없지만 한가지 방식만 쓰는 것이 좋음

### 뮤텍스
- 뮤텍스를 다른 말로 잠금이라고 함
- 자바 언어는 이름이 Lock인 타입을 사용함

### 세마포어
- 동시에 실행할 수 있는 스레드 수를 제한
- 자바 세마포어 구현체는 퍼밋이라고 표현함
- 세마포어에서 퍼밋을 획득하면 허용 가능 숫자가 1 감소하고, 퍼밋을 반환하면 허용 가능 숫자가 1 증가함

### 읽기 쓰기 잠금
- 쓰기는 한 번에 한 스레드만, 읽기는 한 번에 여러 스레드가 획득할 수 있음
- 한 스레드가 쓰기 잠금을 획득했다면 쓰기 잠금이 해제될 때까지 읽기 잠금을 구할 수 없음
- 읽기 잠금을 획득한 모든 스레드가 읽기 잠금을 해제할 때까지 쓰기 잠금을 구할 수 없음

### 원자적 타입 사용
- 자바에는 AtomicInteger, AtomicLong, AtomicBoolean과 같은 타입이 존재하고 이 타입을 사용하면 동시성 문제없이 데이터를 변경할 수 있음
- 이 타입을 사용하면 스레드가 대기하지 않으면서 동시 접근 문제를 해결할 수 있음

### 동시성 지원 컬렉션 - 동기화된 컬렉션을 사용
- 자바 Collections 클래스는 동기화된 컬렉션을 생성하는 메소드를 제공함 ex. Collections.synchronizedMap()
- 이 메소드를 사용하면 기존 컬렉션 객체를 동기화된 컬렉션 객체로 변환할 수 있음
- 가상 스레드는 자바 24부터 synchronized를 지원하기 때문에 그 이전 버전의 자바의 가상 스레드에서는 저 메소드를 사용하면 안됨.

### 동시성 지원 컬렉션 - 동시성 자체를 지원하는 컬렉션 타입을 사용
- HashMap 대신 ConcurrentHashMap을 사용하면 됨.
- 이 타입은 데이터를 변경할 때 잠금 범위를 최소화해서 동기화된 맵을 사용하는 것보다 더 나은 성능을 제공함.

### 불변값
- 불변 값은 말 그대로 바뀌지 않는 값이라 동시에 여러 스레드가 접근해도 문제가 발생하지 않음.

## DB 수준에서의 동시 접근 제어
### 비관적 vs 낙관적 잠금
- 선점 잠금(비관적 잠금)은 동일한 레코드에 대해 한 번에 하나의 트랜잭션만 접근할 수 있도록 제어
    - 먼저 접근한 트랜잭션이 잠금을 획득함.
    - 트랜잭션에 외부 연동이 있다면 선점 잠금을 고려하는 것이 좋음.
- 비선점(낙관적 잠금)은 값을 비교해서 수정함. => 쿼리 자체는 막지 않으면서 데이터 잘못 변경되는 것을 막음.
    - 명시적으로 잠금을 사용하지 않고, 정수 타입의 버전 칼럼을 사용함.
    - 트랜잭션 범위 내에 외부 연동을 비선점 잠금으로 사용하고 싶다면 트랜잭션 아웃박스 패턴을 적용하는 방법도 있음.
- 비관적은 실패할 가능성이 높아서 비관적이고, 낙관적은 성공할 가능성이 높아서 낙관적임

### 분산 잠금
분산 잠금은 여러 프로세스가 동시에 동일한 자원에 접근하지 못하도록 막는 방법임.
분산 잠금은 여러 프로세스 간에 잠금 처리를 한다는 점에서 차이가 있음
주로 레디스를 사용해서 구현함

#### Redis 분산 잠금 구현 원리

**구조**
```java
   🖥️ 서버 A           🖥️ 서버 B           🖥️ 서버 C
       |                  |                  |
       |                  |                  |
       └──────────────────┼──────────────────┘
                          |
                    📦 Redis Server
                     (공유 잠금 저장소)
```

**방법 1: SETNX + EXPIRE**
```bash
# 1단계: 키 설정 (키가 없을 때만)
SETNX lock_key unique_value

# 2단계: 만료 시간 설정 (별도 명령)
EXPIRE lock_key 10

# 문제점: 2단계 명령어라서 원자적이지 않음
# SETNX 성공 후 EXPIRE 전에 프로세스가 죽으면 데드락 위험
```

**방법 2: SET NX EX (권장)**
```bash
# 한 번의 명령어로 원자적 처리
SET lock_key unique_value NX EX 10
# NX: 키가 존재하지 않을 때만 설정
# EX 10: 10초 후 자동 만료 (데드락 방지)
# 키 설정과 TTL 설정을 동시에 처리하여 안전함
```

#### 실무 구현: 선착순 이벤트 쿠폰 발급
```java
@Component
public class RedisDistributedLock {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public boolean tryLock(String lockKey, String requestId, long expireTime) {
        // SET lockKey requestId NX EX expireTime
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, requestId, Duration.ofSeconds(expireTime));
        return Boolean.TRUE.equals(result);
    }
    
    public void unlock(String lockKey, String requestId) {
        // Lua 스크립트로 원자적 삭제 (잘못된 락 해제 방지)
        String script = 
            "if redis.call('GET', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('DEL', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        
        redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(lockKey),
            requestId
        );
    }
}

@Service
public class EventCouponService {
    
    @Autowired
    private RedisDistributedLock distributedLock;
    
    public boolean issueLimitedCoupon(Long eventId, Long userId) {
        String lockKey = "event_coupon:" + eventId;
        String requestId = Thread.currentThread().getName() + ":" + System.nanoTime();
        
        try {
            // 3초 동안 락 획득 시도
            if (distributedLock.tryLock(lockKey, requestId, 3)) {
                try {
                    // 여러 서버에서 동시 실행되어도 한 번만 처리됨
                    return processCouponIssue(eventId, userId);
                    
                } finally {
                    // 반드시 락 해제
                    distributedLock.unlock(lockKey, requestId);
                }
            } else {
                throw new CouponIssueException("쿠폰 발급이 지연되고 있습니다");
            }
        } catch (Exception e) {
            log.error("쿠폰 발급 중 오류 발생: eventId={}, userId={}", eventId, userId, e);
            return false;
        }
    }
    
    private boolean processCouponIssue(Long eventId, Long userId) {
        // 1. 쿠폰 수량 확인
        int remainingCount = getCouponCount(eventId);
        if (remainingCount <= 0) {
            return false;
        }
        
        // 2. 쿠폰 발급 처리
        issueCoupon(eventId, userId);
        decreaseCouponCount(eventId);
        
        return true;
    }
}
```

#### Redisson을 이용한 고급 분산 잠금 (권장)
```java
// build.gradle
// implementation 'org.redisson:redisson-spring-boot-starter:3.24.3'

@Service
public class RedissonLockService {
    
    @Autowired
    private RedissonClient redissonClient;
    
    public void executeWithLock(String lockName, Runnable task) {
        RLock lock = redissonClient.getLock(lockName);
        
        try {
            // 5초 대기, 10초 후 자동 해제
            if (lock.tryLock(5, 10, TimeUnit.SECONDS)) {
                try {
                    task.run(); // 실제 비즈니스 로직 실행
                } finally {
                    if (lock.isHeldByCurrentThread()) {
                        lock.unlock(); // 현재 스레드가 소유한 락만 해제
                    }
                }
            } else {
                throw new LockAcquisitionException("락 획득 실패");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("락 대기 중 인터럽트 발생", e);
        }
    }
}

// 사용 예시
@Service
public class OrderService {
    
    public void processPayment(Long orderId) {
        lockService.executeWithLock("payment:" + orderId, () -> {
            // 중복 결제 방지가 필요한 로직
            validatePayment(orderId);
            processActualPayment(orderId);
            updateOrderStatus(orderId);
        });
    }
}
```

#### 분산 잠금 사용 시 주의사항
```java
// 잘못된 락 해제 - 다른 프로세스의 락 삭제 가능
redisTemplate.delete("lock_key"); // 위험!

// 올바른 락 해제 - Lua 스크립트로 원자적 처리
String script = "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end";
redisTemplate.execute(new DefaultRedisScript<>(script, Long.class), Arrays.asList("lock_key"), "my_value");
```

**주요 포인트:**
- **TTL 설정 필수**: 프로세스 죽을 경우 락이 영원히 남는 것 방지
- **고유한 값 사용**: 다른 프로세스의 락을 실수로 해제하는 것 방지
- **원자적 해제**: GET + DEL을 Lua 스크립트로 한 번에 처리
- **타임아웃 설정**: 무한 대기 방지

### 증분 쿼리
- `update subject set joinCount = joinCount + 1 where id = ?` 와 같은 쿼리를 사용하면 DB가 이를 원자적 연산으로 처리함.

## 잠금 사용 시의 주의 사항
- finally 블록에서 잠금을 해제하는 코드 작성하기
- 대기 시간 지정
- 교착 상태 피하기 ➡️ 교착 상태를 해소하는 방법 중 하나가 대기 시간을 제한하는 것임

### 라이브락(livelock)
- 마주친 두 사람이 같은 방향으로 계속 피해가려고 할 때와 같은 상황을 말함
- 우선 순위를 두거나 중재자를 두거나 임의성을 두는 방법이 있음

## 단일 스레드로 처리하는 방법
- 작업 큐를 두어서 큐에서 필요한 작업만 꺼내서 수행함.

### 참고 자료
- [Redis docs - SET](https://redis.io/docs/latest/commands/set/)

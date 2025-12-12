# 05 | 비동기 연동, 언제 어떻게 써야 할까

## 비동기 연동 방식

비동기 연동을 구현하는 주요 방식들:

- **별도 스레드 실행** - @Async 어노테이션 활용
- **메시징 시스템** - SQS, Kafka 등 활용
- **트랜잭션 아웃박스 패턴** - 데이터 일관성 보장
- **배치 연동** - 주기적 데이터 전송
- **CDC(Change Data Capture)** - 실시간 데이터 변경 추적


## 비동기와 트랜잭션

### 주의사항
> **네이밍 규칙**: 비동기 메소드 이름에는 비동기 실행 관련 단어 추가  
> 예시: `sendMailAsync()`, `processDataAsync()`

> **예외 처리**: 비동기 메소드 호출 시 예외가 발생해도 호출부의 catch 블록이 실행되지 않을 수 있음

#### 실제 겪은 문제 사례

<details>
<summary>🤔 @Async와 트랜잭션 문제</summary>

```java
@Slf4j
@RequiredArgsConstructor
@Service
public class DrivingServiceImpl implements DrivingService {

    private final DrivingRecordRepository drivingRecordRepository;
    private final AthenaQueryService athenaQueryService;
    private final RedisStreamService redisStreamService;

    private final static int TIME_FOR_WAIT = 150000;

    @Async("taskExecutor")
    @Override
    public void summarize(DrivingCompletionRequestDto requestDto) {
        DrivingRecord drivingRecord = DrivingRecord.createInitialRecord(
            requestDto.getDrivingId(), requestDto.getUserId());

        DrivingRecord savedRecord = drivingRecordRepository.save(drivingRecord);

        processAfterDelay(savedRecord.getId(), requestDto.getDrivingId(), requestDto.getUserId());
    }

    @Async("taskExecutor")
    public void processAfterDelay(Long recordId, String drivingId, Long userId) {
        try {
            Thread.sleep(TIME_FOR_WAIT);
            processAnalysis(recordId, drivingId, userId);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("주행 분석 처리 중 인터럽트 발생: drivingId={}", drivingId, e);
        } catch (Exception e) {
            log.error("주행 분석 처리 중 오류 발생: drivingId={}", drivingId, e);
        }
    }

    @Transactional
    public void processAnalysis(Long recordId, String drivingId, Long userId) {
        try {
            DrivingAnalysisResultDto drivingResult = athenaQueryService.getDrivingAnalysis(drivingId);
            EventAnalysisResultDto eventResult = athenaQueryService.getEventAnalysis(drivingId);

            updateDrivingRecord(recordId, drivingResult, eventResult);
        } catch (Exception e) {
            log.error("아테나 쿼리 처리 중 오류 발생: drivingId={}", drivingId, e);
            throw e;
        }
    }

    private void updateDrivingRecord(Long recordId, DrivingAnalysisResultDto drivingResult, 
                                   EventAnalysisResultDto eventResult) {
        DrivingRecord record = drivingRecordRepository.findById(recordId)
            .orElseThrow(() -> new BusinessException(DrivingErrorCode.DRIVING_RECORD_NOT_FOUND,
                "주행 기록을 찾을 수 없습니다: " + recordId));

        record.update(drivingResult, eventResult); 
        // ➡️ update() 시 자동 저장이 안 됨. 아래 줄을 써줘야 update가 됨
        drivingRecordRepository.save(record);

        redisStreamService.publishDrivingEvent(DrivingEventDto.of(record));
        log.info("주행 분석 완료: drivingId={}, recordId={}", record.getDrivingId(), recordId);
    }
}
```

**문제점**: 비동기와 트랜잭션이 꼬여서 `update()` 함수의 Dirty Checking이 제대로 동작하지 않음  
**원인**: 추가 분석 예정...

</details>


## 메시징 시스템을 이용

### 장점
- **시스템 격리**: 두 시스템이 서로 영향을 주지 않음
- **확장성**: 시스템 확장이 용이함

### AWS SQS 개요

| AWS SQS  | SQS 구성도 |
|:---:|:---:|
| <img src="/img/04_1.png" width="180px"> | <img src="/img/04_2.png"> |

#### SQS 큐 타입
- **Standard 큐**: 순서 보장 안됨, 높은 처리량
- **FIFO 큐**: 순서 보장됨, 정확히 1회 처리

#### SQS vs Kafka 비교

| 구분 | SQS | Kafka |
|------|-----|-------|
| **패턴** | 1:1 (경쟁적 소비자) | 1:N (구독 기반) |
| **구조** | 단순 큐 | 토픽 기반 |
| **사용 사례** | 작업 큐, 디커플링 | 이벤트 스트리밍 |

#### 궁극적 일관성(Eventual Consistency)

- 두 데이터 저장소 간의 일관성을 보장하되, 즉시가 아닌 일정 시간 후에 일관성이 맞춰지는 특성


## 아웃박스 패턴 (Transactional Outbox Pattern)

### 목적
메시지 데이터 유실 방지 및 정확히 1회 전송 보장

### 동작 방식
1. **DB에 메시지 저장**: 비즈니스 로직과 같은 트랜잭션에서 메시지 테이블에 저장
2. **별도 프로세스로 발송**: 저장된 메시지를 읽어 메시징 시스템에 전송
3. **발송 상태 관리**: `발송 완료` 칼럼으로 전송 상태 추적

### 장점
- 메시지 유실 방지
- 정확히 1회 전송 보장
- 모니터링 가능

## 배치 연동
- 일정 간격으로 대량의 데이터를 한 번에 전송하는 방식

### 사용 사례
- 일일 매출 데이터 동기화
- 주기적 백업 데이터 전송
- 대용량 데이터 마이그레이션


## CDC (Change Data Capture) 이용
- 변경된 데이터를 추적하고 판별해서 변경된 데이터로 작업을 수행할 수 있도록 하는 소프트웨어 설계 패턴

### 구현 방법

#### 1. Pull 방식
```sql
-- 마지막 동기화 이후 변경된 데이터 조회
SELECT * FROM orders 
WHERE updated_at > '2024-01-01 10:00:00';
```

#### 2. Push 방식
- 데이터베이스 트리거 활용
- 변경 시점에 즉시 알림

#### 3. Log 방식
- 데이터베이스 로그 파일 분석
- MySQL binlog, PostgreSQL WAL 등 활용

### 참고자료

- [Amazon Simple Queue Service란 무엇인가요?](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [무중단 데이터 마이그레이션을 위한 필수 솔루션, CDC](https://www.samsungsds.com/kr/insights/migration_cdc.html)
- [CDC(Change-Data-Capture) 개념과 구축해보기](https://suminii.tistory.com/entry/CDCChange-Data-Capture-%EA%B0%9C%EB%85%90%EA%B3%BC-%EA%B5%AC%EC%B6%95%ED%95%B4%EB%B3%B4%EA%B8%B0)

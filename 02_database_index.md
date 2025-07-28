# 03 | 성능을 좌우하는 DB 설계와 쿼리

## 단일 인덱스와 복합 인덱스

- 특정 사용자의 일자별 활동 내역을 조회할 때 사용자별 활동량이 적다면 `userId` 인덱스만으로 충분함.
- 하지만 사용자별 활동량이 많다면(사용자당 수만 건이 넘는 데이터가 쌓일 경우) `userId`, `activityDate` 컬럼을 조합한 복합 인덱스 사용을 고려해야 함.

## 선택도를 고려한 인덱스 칼럼 선택

- 선택도는 인덱스에서 특정 칼럼의 고유한 값 비율을 나타냄.
- 선택도가 높으면 해당 칼럼에 고유한 값이 많다는 뜻이며, 선택도가 낮으면 고유한 값이 적다는 뜻임.
- 선택도가 높을수록 인덱스를 이용한 조회 효율이 높아짐.
- `gender` 컬럼이 M(50만 개), F(50만 개), N(천 개)의 3개 값 중 하나를 가질 때 이 칼럼을 인덱스로 사용하면 여전히 50만 개의 데이터를 확인해야 해서 인덱스 효율이 낮아짐.
- 그렇지만 선택도가 항상 높아야 하는 것은 아님.
- 선택도가 낮은 컬럼이라도 작업 큐(Work Queue)의 상태(status)처럼 주로 특정 값으로 조회하는 경우 인덱스로 활용하면 효율적일 수 있음.

## 커버링 인덱스 활용하기

- 커버링 인덱스는 특정 쿼리를 실행하는 데 필요한 컬럼을 모두 포함하는 인덱스를 말함.
- 아래 쿼리는 `activityDate`, `activityType` 컬럼값이 모두 인덱스에 포함되어 있으므로, 실제 데이터에 접근할 필요가 없어 조회 성능이 향상됨.
```sql
SELECT activityDate, activityType
FROM activityLog
WHERE activityDate = "2024-07-31" AND activityType = "VISIT";
```

## 인덱스는 필요한 만큼만 만들기

- 효과가 적은 인덱스를 추가하면 오히려 성능이 저하될 수 있음.
- 인덱스는 조회 속도를 높여주지만, 데이터 추가, 변경, 삭제 시에는 인덱스 관리에 따른 비용이 추가되기 때문임.
- 또한 인덱스 자체도 데이터이므로 인덱스가 많아질수록 메모리와 디스크 사용량도 함께 증가함.
- 요구사항을 일부 변경하면 인덱스를 추가하지 않아도 될 수 있으므로 이 방법도 고려해 보는 것이 좋음. (예: 예약자 이름으로 조회 → 특정 일자의 예약자 이름으로 조회)

## 인덱스 안티패턴

### 함수 사용으로 인한 인덱스 무효화
컬럼에 함수를 적용하면 인덱스 사용 불가능
```sql
WHERE YEAR(created_at) = 2024
WHERE UPPER(name) = 'JOHN'
WHERE user_id + 1 = 124
```

인덱스 사용 가능
```sql
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
WHERE name = 'JOHN' -- 대소문자 구분 필요 시 collation 설정 활용
WHERE user_id = 123
```

### LIKE 패턴의 잘못된 사용
앞쪽 와일드카드는 인덱스 사용 불가능
```sql
WHERE name LIKE '%김%'
WHERE email LIKE '%@gmail.com'
```

앞쪽이 고정된 패턴은 인덱스 사용 가능
```sql
WHERE name LIKE '김%'
WHERE email LIKE 'admin@%'
```

### 부정 조건 사용
부정 조건은 인덱스 효율성 저하
```sql
WHERE user_id != 123
WHERE name NOT LIKE 'admin%'
```

긍정 조건으로 변경
```sql
WHERE user_id IN (124, 125, 126, ...)  -- ⚠️가능한 값들이 적을 때(값이 많아지면 인덱스 사용X)
WHERE status IN ('ACTIVE', 'PENDING')  -- 가능한 상태값들
```

### NULL 처리의 함정
ISNULL 함수 사용으로 인덱스 비효율
```sql
WHERE ISNULL(phone_number, '') = '010-1234-5678'
```

IS NULL과 등호 조건 분리
```sql
WHERE phone_number = '010-1234-5678'
OR (phone_number IS NULL AND '010-1234-5678' = '')
```

## 페이징 최적화

### 기존 OFFSET의 문제점
```sql
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 0; -- 첫 페이지를 읽어오는 쿼리
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 2000000; -- 마지막 페이지를 읽어오는 쿼리
```
- 페이지가 깊어질수록 건너뛸 데이터가 기하급수적으로 증가
- DB가 200만 개의 레코드를 건너뛰고 나서 20개 데이터를 조회함.
- 데이터를 건너뛰는 시간만큼 실행 시간이 증가하게 됨.

### ID 기반 커서 페이징
```sql
SELECT * FROM article
WHERE id < 9985 AND deleted = false
ORDER BY id DESC
LIMIT 10;
```
- id는 인덱스이므로 9985보다 작은 값인 9984를 바로 찾을 수 있음.
- 오프셋을 사용했을 때는 지정한 오프셋만큼 데이터를 건너뛰는 시간이 필요하지만 이 과정이 생략되어 실행 시간이 빨라짐.

### 페이징 정리
- ID 기반 커서 페이징은 성능이 일정해서 무한 스크롤 방식에 적합함.
- 하지만 원하는 위치로 바로 이동하기 어렵다는 단점이 있어 "3페이지로 가기", "마지막 페이지로 가기" 기능 구현 어려움.

### 참고자료
- [SQL 안티패턴](https://sonra.io/mastering-sql-how-to-detect-and-avoid-34-common-sql-antipatterns/#use-an-alias-for-derived-columns)
- [인덱스를 안타는 쿼리들](https://dkswnkk.tistory.com/694)
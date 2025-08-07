# 03 | 성능을 좌우하는 DB 설계와 쿼리

## 1. 인덱스가 필요한 이유

DB 서버 성능 문제의 주요 원인 중 하나는 **풀 스캔**임. 호출 빈도가 높은 기능에 풀 스캔을 유발하는 쿼리가 있으면 사용자가 조금만 증가해도 DB 장비의 CPU 사용률이 90%를 넘김. DB에 문제가 생기면 전체 서비스에 영향을 주기 때문에 신경써야 함.

### 풀 스캔이란?

- **풀 스캔**: 테이블의 모든 데이터를 순차적으로 읽는 것
- 일반적인 시스템에서는 조회 기능의 실행 비율이 높음 (게시판 - 게시글 조회)
- 풀 스캔이 발생하지 않도록 하려면 조회 패턴을 기준으로 인덱스를 설계해야 함

### 인덱스 없이 실행되는 쿼리 예시

책에서 나오는 것과 같은 article 테이블을 예로 들어보자.

![article 테이블](https://github.com/user-attachments/assets/e4ea46de-794c-4b89-8041-8e0093dd69f0)

JPA Repository에서 다음과 같은 메서드들은 인덱스 없이 실행하면 풀 스캔이 발생함.

```java
@Repository
public interface ArticleRepository extends JpaRepository<Article, Long> {
    // 카테고리별 게시글 조회 -> 풀 스캔
    List<Article> findByCategory(Integer category);
    
    // 특정 작성자의 글 조회 -> 풀 스캔
    List<Article> findByWriterId(Long writerId);
    
    // 카테고리별 최신 글 조회 -> 성능 문제
    List<Article> findByCategoryOrderByRegdtDesc(Integer category);
}
```

## 2. 단일 인덱스

**단일 인덱스**는 하나의 컬럼에만 만드는 가장 기본적인 인덱스임.

![단일 인덱스](https://github.com/user-attachments/assets/82aa41c4-ac96-498b-bbdb-f6b3a357ead1)

게시판에서 카테고리별 게시글 목록을 조회하는 패턴이 존재하므로 category 컬럼에 인덱스를 추가해서 조회 성능을 개선할 수 있음.

```sql
-- 카테고리 조회용 단일 인덱스
CREATE INDEX idx_article_category ON article(category);

-- 작성자 조회용 단일 인덱스 (내가 작성한 글 목록 보기용)
CREATE INDEX idx_article_writer_id ON article(writerId);
```

```java
// JPA에서 인덱스 설정
@Entity
@Table(name = "article", indexes = {
    @Index(name = "idx_article_category", columnList = "category"),
    @Index(name = "idx_article_writer_id", columnList = "writerId")
})
public class Article {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private Integer category;
    private Long writerId;
    private String title;
    
    @Lob
    private String content;
    
    private LocalDateTime regdt;
}
```

### 단일 인덱스의 특징

**장점**:
- 구현이 간단하고 이해하기 쉬움.
- 메모리 사용량이 적음.
- 단일 조건 쿼리에서 성능이 빠름.

**단점**:
- 복합 조건 쿼리에서는 효과가 제한적임.
- 여러 인덱스를 조합해서 사용할 때 비효율적일 수 있음.

## 3. 복합 인덱스

**복합 인덱스**는 여러 컬럼을 조합해서 만드는 인덱스임. 여러 조건을 함께 사용하는 쿼리나 WHERE + ORDER BY 조합에 효과적임.

![복합 인덱스](https://github.com/user-attachments/assets/2362af0a-1bf6-434c-bd08-b93ed7b32625)

실제 서비스에서는 다음과 같은 복합 조건의 쿼리가 필요함.

```java
// 카테고리별 최신 글 조회 (WHERE + ORDER BY)
List<Article> findByCategoryOrderByRegdtDesc(Integer category);

// 특정 작성자의 특정 카테고리 글 조회 (WHERE + WHERE)
@Query("SELECT a FROM Article a WHERE a.category = :category AND a.writerId = :writerId")
List<Article> findByCategoryAndWriterId(@Param("category") Integer category, 
                                       @Param("writerId") Long writerId);
```

```sql
-- 카테고리 + 등록일 복합 인덱스 (정렬 쿼리 최적화)
CREATE INDEX idx_article_category_regdt ON article(category, regdt);

-- 카테고리 + 작성자 복합 인덱스 (복합 조건 최적화)
CREATE INDEX idx_article_category_writer ON article(category, writerId);
```

### 선택도를 고려한 인덱스 컬럼 선택

**선택도**(Selectivity)는 인덱스에서 특정 컬럼의 고유한 값 비율임. 선택도를 고려해서 컬럼 순서를 결정해야 함.

```sql
-- 선택도 분석
SELECT 
    COUNT(DISTINCT category) / COUNT(*) as category_selectivity,
    COUNT(DISTINCT writerId) / COUNT(*) as writer_selectivity,
    COUNT(DISTINCT title) / COUNT(*) as title_selectivity
FROM article;
```

> **💡 Q. 항상 선택도가 높은 컬럼을 앞에 둬야 할까?**<br>
> : 아님. 선택도도 중요하지만 실제 쿼리 패턴이 더 중요함. 작업 큐를 구현한 테이블처럼 선택도가 낮아도 인덱스 컬럼으로 적합한 상황도 있음. 예를 들어 status 컬럼이 PENDING, PROCESSING, COMPLETED 3개 값만 가져서 선택도가 낮지만, PENDING 상태의 작업만 주로 조회한다면 유효한 인덱스가 됨.

### 복합 인덱스에서 컬럼 순서

복합 인덱스는 **전화번호부**와 비슷함. 전화번호부는 "성 → 이름" 순으로 정렬되어 있어서, "김"씨를 찾기는 쉽지만 "철수"라는 이름으로 찾기는 어려움.

```sql
-- 좋은 예: WHERE 조건에 자주 쓰이는 컬럼을 앞에
CREATE INDEX idx_good ON article(category, regdt);

-- 나쁜 예: ORDER BY에만 쓰이는 컬럼을 앞에
CREATE INDEX idx_bad ON article(regdt, category);
```

## 4. 커버링 인덱스

**커버링 인덱스**는 특정 쿼리를 실행하는 데 필요한 컬럼을 모두 포함하는 인덱스임.

![커버링 인덱스](https://github.com/user-attachments/assets/7f4cbe99-0823-40bf-9d05-557aa6b3c87b)

커버링 인덱스를 사용하면 실제 데이터를 읽지 않기 때문에 조회 시간을 단축할 수 있음. 게시글 목록 페이지에서는 보통 모든 데이터가 아니라 일부 정보만 필요함.

```java
// 목록용 DTO 설계
public class ArticleSummaryDto {
    private Long id;
    private String title;
    private Long writerId;
    private LocalDateTime regdt;
    
    public ArticleSummaryDto(Long id, String title, Long writerId, LocalDateTime regdt) {
        this.id = id;
        this.title = title;
        this.writerId = writerId;
        this.regdt = regdt;
    }
}

// DTO 프로젝션을 활용한 최적화된 쿼리
@Query("SELECT new com.example.dto.ArticleSummaryDto(a.id, a.title, a.writerId, a.regdt) " +
       "FROM Article a WHERE a.category = :category ORDER BY a.regdt DESC")
Page<ArticleSummaryDto> findArticleSummaryByCategory(
    @Param("category") Integer category, 
    Pageable pageable
);
```

```sql
-- 목록 조회에 필요한 모든 컬럼을 포함한 커버링 인덱스
CREATE INDEX idx_article_covering ON article(category, regdt, id, title, writerId);
```

### 커버링 인덱스 활용 사례

커버링 인덱스는 다음과 같은 상황에서 사용함.

- 목록 조회처럼 일부 컬럼만 필요한 경우
- COUNT나 집계 쿼리
- 페이징 처리 시 성능 최적화

```java
// COUNT 쿼리도 커버링 인덱스로 최적화 가능
@Query("SELECT COUNT(a) FROM Article a WHERE a.category = :category")
Long countByCategory(@Param("category") Integer category);

// 통계 조회
@Query("SELECT a.category, COUNT(a), MAX(a.regdt) " +
       "FROM Article a GROUP BY a.category")
List<Object[]> getArticleStats();
```

## 5. 인덱스 운영 전략

### 인덱스는 필요한 만큼만 만들기

인덱스 자체도 데이터이기 때문에 인덱스가 많아질수록 메모리와 디스크 사용량도 함께 증가함.

```java
// 나쁜 예: 과도한 인덱스 생성
@Table(name = "article", indexes = {
    @Index(name = "idx1", columnList = "category"),
    @Index(name = "idx2", columnList = "writerId"), 
    @Index(name = "idx3", columnList = "regdt"),
    @Index(name = "idx4", columnList = "category, writerId"),
    @Index(name = "idx5", columnList = "category, regdt"),
    // 너무 많은 인덱스는 메모리 낭비와 INSERT/UPDATE 성능 저하
})

// 좋은 예: 효율적인 인덱스 설계
@Table(name = "article", indexes = {
    @Index(name = "idx_category_covering", columnList = "category, regdt, id, title, writerId"),
    @Index(name = "idx_writer_date", columnList = "writerId, regdt")
    // 대부분의 쿼리 패턴을 커버하는 2개의 효율적인 인덱스
})
```

### 요구사항 조정을 통한 최적화

새로 추가할 쿼리가 기존에 존재하는 인덱스를 사용하지 않을 때는 요구사항을 일부 변경할 수 있는지 검토해보자. 작은 변경만으로 인덱스를 활용할 수 있음.

```java
// 이전 요구사항: 복잡한 다중 조건 검색
@Query("SELECT a FROM Article a WHERE " +
       "(a.category = :category OR :category IS NULL) AND " +
       "(a.writerId = :writerId OR :writerId IS NULL) AND " +
       "(a.title LIKE %:keyword% OR :keyword IS NULL)")
List<Article> complexSearch(Integer category, Long writerId, String keyword);

// 변경된 요구사항: 단계별 검색으로 기존 인덱스 활용
// 1단계: 카테고리로 1차 필터링
// 2단계: 작성자로 2차 필터링  
// 3단계: 제목 검색은 별도 기능으로 분리
```

### 인덱스 모니터링과 유지보수

인덱스의 효과를 지속적으로 확인하고 관리하는 것이 중요함.

```sql
-- 인덱스 사용률 확인 (MySQL 예시)
SELECT 
    schema_name,
    table_name,
    index_name,
    stat_value as pages_accessed
FROM information_schema.INNODB_SYS_TABLESTATS 
WHERE stat_name = 'n_page_gets';

-- 실행 계획 확인
EXPLAIN SELECT * FROM article WHERE category = 1 ORDER BY regdt DESC;
```

> **💡 Q. 동일한 효율을 보이는 인덱스는 왜 피해야 할까?**<br>
> : 인덱스는 INSERT, UPDATE, DELETE 할 때마다 함께 업데이트되어야 함. 불필요한 인덱스가 많으면 쓰기 성능이 저하되고 메모리도 낭비됨. 또한 옵티마이저가 잘못된 인덱스를 선택할 가능성도 높아짐.

## 참고 자료

1. [이성욱, "Real MySQL 8.0", 위키북스, 2021](https://product.kyobobook.co.kr/detail/S000001766482)
2. [우아한형제들 기술블로그, "MySQL 인덱스 정리 및 팁", 2020.07.14](https://techblog.woowahan.com/2627/)
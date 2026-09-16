# 탐색 페이지 필터가 20초 걸리던 이유 — `IS NULL OR` 한 줄이 PostgreSQL 플래너를 어떻게 묶었나

> **TL;DR** 게임 탭에서 "스팀 리뷰 N개 이상" 필터를 걸면 요청 1회에 약 20초가 걸렸다. 원인은 인덱스도, 서브쿼리도 아니었다. 필터 8개를 쿼리 하나로 처리하려고 넣은 `(:param IS NULL OR 조건)` 스위치가, 바인딩 파라미터 상황에서 플래너의 최적화를 통째로 봉인해 **19만 행 전수 스캔 + 행별 프로브 17만 번**을 강제하고 있었다. "켜진 조건만 SQL에 넣는" 동적 조립으로 바꾸자 같은 요청이 **E2E 21초 → 1.4초(콜드) / 0.4초(웜)** 가 됐다. 이 글은 그 과정을 실제 `EXPLAIN ANALYZE` 출력과 코드로 따라간다.

---

## 1. 상황

**AOD(All of Dopamine)** 는 영화·TV·게임·웹툰·웹소설 정보를 여러 플랫폼에서 크롤링해 모아 보여주는 서비스다. 백엔드는 Spring Boot 3 + PostgreSQL(AWS RDS), `contents`(마스터, 약 23만 행) 아래에 도메인별 자식 테이블(`game_contents` 등)이 1:1로 붙는 구조다. 게임은 Steam에서 수집해 **GAME 도메인만 약 19만 행**으로 가장 크다.

탐색 페이지(`/explore`)에는 도메인·장르·플랫폼·출시일 등 필터가 있고, 2026-08에 게임 탭 전용으로 "스팀 리뷰 개수 하한(`reviewCountMin`)" 필터를 추가했다. 문제는 이 필터를 켜는 순간이었다.

- 다른 필터는 느려도 결과가 뜨는데, 리뷰 개수 필터는 **"아예 작동하지 않는" 수준**
- 프론트 API 클라이언트의 타임아웃이 30초라, 체감상 "안 됨"의 실체는 타임아웃이었다

관찰을 문제 정의로 바꾸면 이렇다: *"GAME 도메인 + `reviewCountMin` 파라미터가 있을 때 `/api/works` 응답이 수십 초."* 특정 파라미터가 붙을 때만 느리다면, 용의자는 그 파라미터가 타는 코드 경로다.

## 2. 쿼리 찾기

파라미터 이름 `reviewCountMin`으로 백엔드를 검색하면 경로가 한 줄로 나온다.

```
GET /api/works?domain=GAME&reviewCountMin=1000&page=0&size=20
  → WorkController.getWorks          파라미터 수집 → WorkFilters record
  → WorkApiService.getWorks           filters.hasAny() ? 필터 경로 : 무필터 경로
  → getWorksWithDbFiltering           contentRepository.findWorks(...)   ← 여기
  → enrichSummaries                   20건에 자식 테이블·platform_data 배치 조회 (가벼움)
```

`findWorks`는 필터 축 8개를 **네이티브 쿼리 하나**로 처리한다. 각 축은 `(:param IS NULL OR 조건)` 스위치로 켜고 끈다.

```sql
SELECT c.* FROM contents c
WHERE c.domain = :domain
  AND c.is_adult = false
  AND (CAST(:genres AS text[]) IS NULL OR c.genres @> CAST(:genres AS text[]))
  AND (CAST(:platforms AS text[]) IS NULL OR c.platforms && CAST(:platforms AS text[]))
  AND (CAST(:keyword AS text) IS NULL OR c.master_title ILIKE ('%' || :keyword || '%')
       OR c.original_title ILIKE ('%' || :keyword || '%'))
  AND (CAST(:releaseFrom AS date) IS NULL OR c.release_date >= CAST(:releaseFrom AS date))
  AND (CAST(:releaseTo AS date) IS NULL OR c.release_date <= CAST(:releaseTo AS date))
  AND ((CAST(:status AS text) IS NULL AND CAST(:weekdays AS text[]) IS NULL AND CAST(:ageRatings AS text[]) IS NULL)
       OR EXISTS (SELECT 1 FROM webtoon_contents w WHERE w.content_id = c.content_id
           AND (CAST(:status AS text) IS NULL OR w.status = :status)
           AND (CAST(:weekdays AS text[]) IS NULL OR w.weekday = ANY(CAST(:weekdays AS text[])))
           AND (CAST(:ageRatings AS text[]) IS NULL OR w.age_rating = ANY(CAST(:ageRatings AS text[])))))
  AND (CAST(:reviewCountMin AS integer) IS NULL                                   -- ★ 문제의 축
       OR EXISTS (SELECT 1 FROM game_contents g WHERE g.content_id = c.content_id
           AND g.review_count >= CAST(:reviewCountMin AS integer)))
ORDER BY c.release_date DESC NULLS LAST, c.content_id ASC
-- Spring Data가 LIMIT 20 OFFSET n 을 붙이고, 같은 WHERE로 SELECT COUNT(*) 를 한 번 더 실행한다
```

이 설계에는 이유가 있었다. 도메인별로 따로 있던 `findWorks` 5개를 통합하면서(2026-07), 필터 조합이 몇 개가 되든 자바 쪽에서 조건을 조립하지 않아도 되게 하려는 것. 필터를 안 쓰면 NULL이 바인딩되어 `IS NULL`이 참이 되고, 그 축은 통과한다. 자바는 단순해졌다. **그 대가를 누가 치르고 있는지**가 이 글의 주제다.

## 3. 분석

### 3-1. 인덱스 현황부터

플랜을 읽으려면 "쓸 수 있는 인덱스"를 먼저 알아야 한다.

```sql
SELECT tablename, indexname, indexdef
FROM pg_indexes WHERE tablename IN ('contents', 'game_contents');
```

| 테이블 | 인덱스 | 정의 |
|---|---|---|
| contents | contents_pkey | btree (content_id) |
| contents | idx_contents_lookup | btree (domain, master_title, release_date) |
| contents | idx_contents_genres / platforms | gin (genres) / gin (platforms) |
| game_contents | game_contents_pkey | btree (content_id) |
| game_contents | idx_game_genres / platforms | gin |

두 가지가 없다. `game_contents.review_count` 인덱스, 그리고 `ORDER BY release_date`를 지원하는 인덱스(`idx_contents_lookup`은 domain 다음이 master_title이라 출시일 순서를 주지 못한다).

### 3-2. 첫 측정 — 값을 직접 넣고 재보니 0.1초

가장 단순한 방법부터. 리뷰 하한을 SQL에 직접 적어서(`>= 1000`) 실행 계획을 떠봤다.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.* FROM contents c
WHERE c.domain = 'GAME' AND c.is_adult = false
  AND EXISTS (SELECT 1 FROM game_contents g
              WHERE g.content_id = c.content_id AND g.review_count >= 1000)
ORDER BY c.release_date DESC NULLS LAST, c.content_id ASC
LIMIT 20 OFFSET 0;
```

결과는 **113ms**. 플랜을 말로 풀면 두 줄이다.

1. 게임 장부(`game_contents`)를 훑어서 리뷰 1000개 이상인 게임 **8,759건**을 먼저 뽑는다
2. 그 8,759건만 작품 서류철(`contents`)에서 번호(PK)로 찾아온다

```
-> Nested Loop
     -> Parallel Seq Scan on game_contents g   Filter: (review_count >= 1000)   ← ① 게임 장부부터
     -> Index Scan using contents_pkey on contents c  (loops=8759)               ← ② 8,759건만 역조회
Execution Time: 113.328 ms
```

19만 건짜리 서류철을 8,759번만 펼치니 빠를 수밖에 없다. 좋은 플랜이다.

**그런데 제보는 20초다.** 측정 0.1초와 제보 20초 — 둘 중 하나는 실제 조건이 아니다. 나는 내 측정을 의심했다. 이유는 간단하다. **앱은 SQL에 값을 적어서 보내지 않는다.** JDBC는 `review_count >= ?`처럼 빈칸으로 보내고 값은 나중에 따로 끼워 넣는다(바인딩). 즉 내 실험은 DB에 "값이 1000이야"라고 미리 알려준 것이고, 앱은 그 정보를 주지 않는다. 값을 미리 알면 할 수 있는 최적화가 있다면, 값을 모를 때 못 하는 최적화도 있을 것이다. 그걸 재현해야 했다.

### 3-3. 앱처럼 재현하려다 한 번 넘어짐

바인딩 상황을 흉내 내려고 `PREPARE`(빈칸이 있는 문장을 미리 등록) + `EXECUTE`로 실행했더니 결과가 이상했다.

```
Result  One-Time Filter: false   rows=0   Execution Time: 0.037 ms
```

"조건이 항상 거짓이라 0건." 방금 8,760건이 있다는 걸 확인했는데 0건이 나올 수는 없다. **결과가 논리적으로 불가능하면 쿼리가 아니라 측정 도구를 의심해야 한다.** 서버에 실제로 등록된 문장을 꺼내봤다.

```sql
SELECT name, statement FROM pg_prepared_statements;
-- ... WHERE c.domain = NULL AND ...
```

내가 쓴 `$1`(첫 번째 빈칸)이 서버에는 `NULL`로 도착해 있었다. DB 클라이언트 툴이 `$1`을 자기 변수로 해석해 빈 값으로 바꿔치기한 것. `domain = NULL`은 항상 거짓이니 DB가 "볼 것도 없이 0건"이라고 답한 게 `One-Time Filter: false`의 정체였다. 이 측정은 통째로 무효.

> 교훈: "서버가 실제로 받은 것"을 확인하라. 툴은 시스템의 일부다.

### 3-4. 앱처럼 재현 성공 — 9.87초

툴이 건드릴 수 없는 방법을 썼다. 값을 `(SELECT 1000)`처럼 **서브쿼리로 감싸면** DB는 이걸 "실행해봐야 아는 값"으로 취급해서, 계획을 짤 때 값을 못 본다. 앱의 빈칸(`?`)과 같은 상황이다. 앱이 보내는 쿼리에는 스위치가 8개 있으니 8개 전부 이렇게 바꿨다(안 쓰는 필터는 앱과 똑같이 NULL).

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.* FROM contents c
WHERE c.domain = (SELECT 'GAME'::text)
  AND c.is_adult = false
  AND ((SELECT NULL::text[]) IS NULL OR c.genres @> (SELECT NULL::text[]))          -- 장르 (꺼짐)
  AND ((SELECT NULL::text[]) IS NULL OR c.platforms && (SELECT NULL::text[]))       -- 플랫폼 (꺼짐)
  -- ... 키워드·출시일·웹툰 스위치도 같은 방식으로 꺼진 상태 ...
  AND ((SELECT 1000) IS NULL                                                        -- 리뷰 (켜짐)
       OR EXISTS (SELECT 1 FROM game_contents g WHERE g.content_id = c.content_id
           AND g.review_count >= (SELECT 1000)))
ORDER BY c.release_date DESC NULLS LAST, c.content_id ASC
LIMIT 20 OFFSET 0;
```

이번엔 **9.87초.** 같은 조건의 count 쿼리가 **9.96초.** 합쳐서 약 20초 — 제보와 맞아떨어졌다.

이 플랜은 3-2와 **일하는 순서가 정반대**다. 말로 풀면 세 줄이다.

1. 작품 서류철에서 GAME 작품 **19만 건을 전부** 꺼낸다 (8,759건이 아니라)
2. 한 건씩 "이 작품 리뷰 1000개 넘어?"를 게임 장부에서 찾아본다 — **17만 6천 번**
3. 19만 건을 꺼내는 데 필요한 읽기의 대부분이 메모리가 아닌 **디스크**에서 왔다 — 9.87초 중 8.86초

플랜에서 그 세 줄에 해당하는 부분만 남기면:

```
Buffers: shared hit=718629 read=18035
I/O Timings: shared read=8856.712                    ← ③ 디스크 대기 8.86초 (전체의 90%)
-> Bitmap Heap Scan on contents c  (rows=1 예상) (actual rows=8758)
     Filter: (... ((InitPlan 30).col1 IS NULL) OR EXISTS(SubPlan 32) ...)   ← IS NULL OR 가 그대로 남아 행마다 검사
     Rows Removed by Filter: 176797                  ← ① 19만 건 읽고 17.7만 건 버림
     SubPlan 32
       -> Index Scan using game_contents_pkey on game_contents g   loops=176292   ← ② 17만 6천 번 찾아봄
```

### 3-5. 왜 DB는 이렇게 일했나 — 행정실 직원 비유

DB를 행정실 직원이라고 생각하자. 직원 앞에는 **게임 장부**(게임마다 리뷰 수가 적힌 18만 장)와 **작품 서류철**(모든 작품의 상세 서류, GAME만 19만 장)이 있다. 직원의 규칙 하나: **작업 순서를 먼저 정한 다음에 손을 움직인다.** 중간에 순서를 바꾸지 않는다.

**3-2의 요청서**: "리뷰 1000개 이상인 게임만."
직원: "게임 장부에서 1000 이상인 것만 추리자 → 8,759장. 그 번호로 서류철에서 딱 그만큼만 꺼내자." → 0.1초.

**3-4(=앱)의 요청서**: "**봉투 안의 값이 비어 있으면 게임 전부, 비어 있지 않으면 그 값 이상인 게임만.** 봉투는 작업을 시작한 뒤에 열어볼 것."
직원: "봉투가 비어 있으면 전부를 줘야 하는데, 게임 장부에서 8,759장만 뽑는 순서로 정해버리면 그때 틀린 답이 된다. 어느 쪽이든 안전한 순서는 하나뿐 — **서류철에서 GAME 19만 장을 전부 꺼내놓고, 봉투를 연 다음 한 장씩 대조하자.**"

봉투를 열어보니 1000이었지만 순서는 이미 정해진 뒤다. 그래서 19만 장을 꺼내고, 17만 7천 장을 버렸다. 이게 **이유 ①: `IS NULL OR`는 "값을 모른 채 계획하라"는 요구이고, 그 요구에 대한 유일하게 안전한 답이 전수조사다.** 3-2와 3-4의 유일한 차이는 요청서에 값이 적혀 있었느냐다.

**이유 ②, 17만 번 찾아본 것.** 직원은 순서를 정할 때 작업량도 어림한다. "스위치 8개 각각 절반쯤 통과하겠지"를 곱하다 보니 "끝까지 살아남는 건 **1장**"이라는 엉터리 어림이 나왔다(실제 8,758장). 1장이면 굳이 리뷰 1000+ 게임 체크리스트를 미리 만들 필요가 없다 — "그 1장이 나오면 그때 장부를 한 번 찾아보지." 그런데 실제로는 17만 6천 장이 대조 대상이었고, 장부를 17만 6천 번 찾아보게 됐다. 값을 모르니 어림이 무너지고, 무너진 어림이 세부 전략까지 잘못 고르게 한 것이다.

**이유 ③, 디스크.** 19만 장은 약 130MB다. 이 서버의 메모리 캐시에는 안 들어간다. 그래서 매번 창고(디스크)에서 꺼내 온다 — 9.87초 중 8.86초. 그리고 화면 하단의 "전체 N건"을 위해 count 쿼리가 같은 19만 장을 **한 번 더** 꺼낸다(직전에 읽은 건 벌써 캐시에서 밀려났다). 그래서 ×2, 약 20초.

한 줄로: **`IS NULL OR`가 전수조사를 강제하고 → 전수조사가 캐시를 넘어 디스크를 때리고 → count가 그걸 두 번 하게 했다.**

### 3-6. "인덱스만 추가하면 안 되나?"

`game_contents(review_count)` 인덱스를 만들면 빨라지는 부분은 `SubPlan`의 game_contents 쪽뿐이고, 그 부분은 시간의 극히 일부다. 8.9초의 디스크 읽기(contents 전수 스캔)는 ①의 구조가 강제하는 것이라 인덱스로 사라지지 않는다. 인덱스는 좋은 플랜이 선택된 다음에 각 단계를 가속하는 증폭기이지, 나쁜 플랜을 좋은 플랜으로 바꿔주지 않는다.

## 4. 결론과 설계 결정

원인이 "값을 계획 시점에 모르게 만드는 구조"라면 해법은 하나다.

> **DB에는 실제로 켜진 조건만 보낸다.**

자바에서 `if (reviewCountMin != null)`이면 `AND EXISTS (...)`를 붙이고, 아니면 아예 붙이지 않는다. 그러면 DB는 항상 확정된 조건만 받고, `IS NULL`이 없으니 EXISTS는 최상위 AND → 세미조인 변환 가능 → 추정도 실제 통계로 돌아온다. ①②가 한 번에 사라진다.

도구 선택도 검토했다. QueryDSL·Specification·MyBatis·jOOQ 어느 것이든 "켜진 조건만 넣는다"는 원리는 같아서 성능은 도구의 기준이 못 되고, 남는 기준은 **PostgreSQL 전용 문법(`@>`, `&&`, 상관 EXISTS, `NULLS LAST`)을 얼마나 자연스럽게 다루느냐**였다. QueryDSL은 JPQL 경유라 배열 연산자를 결국 `Expressions.booleanTemplate(...)` 문자열로 써야 해서 타입 안전성의 이득이 핵심 조건에서 사라지고, MyBatis는 JPA 프로젝트에 두 번째 영속성 스택을 들이는 비용, jOOQ는 쿼리 하나에 비해 과투자였다. 이 프로젝트가 이미 네이티브 SQL 기반이라는 점까지 더해, **값은 바인딩 파라미터로만 넘기는 조건부 문자열 조립**을 택했다. 조립기를 순수 클래스로 분리해 DB 없이 단위 테스트가 가능하게 한 것이 이 선택의 핵심 근거다.

## 5. 개선 — Before / After

### Before

```java
// ContentRepository (shared 모듈)
String WORKS_FILTER =
       "c.domain = :domain AND c.is_adult = false " +
       "AND (CAST(:genres AS text[]) IS NULL OR c.genres @> CAST(:genres AS text[])) " +
       // ... 스위치 7개 더 ...
       "AND (CAST(:reviewCountMin AS integer) IS NULL " +
       "     OR EXISTS (SELECT 1 FROM game_contents g WHERE g.content_id = c.content_id " +
       "         AND g.review_count >= CAST(:reviewCountMin AS integer))) ";

@Query(value = "SELECT c.* FROM contents c WHERE " + WORKS_FILTER +
       "ORDER BY c.release_date DESC NULLS LAST, c.content_id ASC",
       countQuery = "SELECT COUNT(*) FROM contents c WHERE " + WORKS_FILTER,
       nativeQuery = true)
Page<Content> findWorks(@Param("domain") String domain, @Param("genres") String[] genres,
                        /* ... 8개 더 ... */ @Param("reviewCountMin") Integer reviewCountMin,
                        Pageable pageable);
```

```java
// WorkApiService
Page<Content> page = contentRepository.findWorks(domain.name(),
        toArr(filters.genres()), toArr(filters.platforms()), kw,
        blankToNull(filters.releaseFrom()), blankToNull(filters.releaseTo()),
        blankToNull(filters.status()), toArr(filters.weekdays()), toArr(filters.ageRatings()),
        filters.reviewCountMin(), pageReq);
```

### After

**조립기 — 순수 함수, DB 없이 테스트 가능**

```java
// WorksQueryBuilder (shared 모듈)
public static Built build(WorksFilterCriteria c) {
    StringBuilder where = new StringBuilder("c.domain = :domain AND c.is_adult = false");
    Map<String, Object> params = new LinkedHashMap<>();
    params.put("domain", c.domain());

    if (notEmpty(c.genres())) {
        where.append(" AND c.genres @> CAST(:genres AS text[])");
        params.put("genres", toArray(c.genres()));
    }
    if (notEmpty(c.platforms())) {
        where.append(" AND c.platforms && CAST(:platforms AS text[])");
        params.put("platforms", toArray(c.platforms()));
    }
    if (notBlank(c.keyword())) {
        where.append(" AND (c.master_title ILIKE :keyword OR c.original_title ILIKE :keyword)");
        params.put("keyword", "%" + c.keyword().trim() + "%");
    }
    if (c.releaseFrom() != null) { where.append(" AND c.release_date >= :releaseFrom"); params.put("releaseFrom", c.releaseFrom()); }
    if (c.releaseTo()   != null) { where.append(" AND c.release_date <= :releaseTo");   params.put("releaseTo",   c.releaseTo()); }

    boolean status = notBlank(c.status()), weekdays = notEmpty(c.weekdays()), ageRatings = notEmpty(c.ageRatings());
    if (status || weekdays || ageRatings) {
        where.append(" AND EXISTS (SELECT 1 FROM webtoon_contents w WHERE w.content_id = c.content_id");
        if (status)     { where.append(" AND w.status = :status");                              params.put("status", c.status().trim()); }
        if (weekdays)   { where.append(" AND w.weekday = ANY(CAST(:weekdays AS text[]))");       params.put("weekdays", toArray(c.weekdays())); }
        if (ageRatings) { where.append(" AND w.age_rating = ANY(CAST(:ageRatings AS text[]))");  params.put("ageRatings", toArray(c.ageRatings())); }
        where.append(")");
    }
    if (c.reviewCountMin() != null) {
        where.append(" AND EXISTS (SELECT 1 FROM game_contents g"
                + " WHERE g.content_id = c.content_id AND g.review_count >= :reviewCountMin)");
        params.put("reviewCountMin", c.reviewCountMin());
    }
    return new Built(SELECT + where + ORDER_BY, COUNT + where, params);
}
```

**실행 — EntityManager로 네이티브 실행, count는 Spring Data와 같은 규칙으로 생략 가능**

```java
// ContentRepositoryImpl (Spring Data custom repository)
public Page<Content> findWorks(WorksFilterCriteria criteria, Pageable pageable) {
    WorksQueryBuilder.Built built = WorksQueryBuilder.build(criteria);

    Query main = em.createNativeQuery(built.sql(), Content.class);
    built.params().forEach(main::setParameter);
    main.setFirstResult((int) pageable.getOffset());
    main.setMaxResults(pageable.getPageSize());
    List<Content> rows = main.getResultList();

    return PageableExecutionUtils.getPage(rows, pageable, () -> {
        Query count = em.createNativeQuery(built.countSql());
        built.params().forEach(count::setParameter);
        return ((Number) count.getSingleResult()).longValue();
    });
}
```

```java
// WorkApiService — 축을 record로 묶어 전달 (날짜는 LocalDate 바인딩, SQL CAST 제거)
WorksFilterCriteria criteria = new WorksFilterCriteria(domain.name(),
        filters.genres(), filters.platforms(), kw,
        toDate(filters.releaseFrom()), toDate(filters.releaseTo()),
        blankToNull(filters.status()), filters.weekdays(), filters.ageRatings(),
        filters.reviewCountMin());
Page<Content> page = contentRepository.findWorks(criteria, pageReq);
```

리뷰 필터만 켜졌을 때 DB에 도착하는 SQL은 이제 이렇다.

```sql
SELECT c.* FROM contents c
WHERE c.domain = :domain AND c.is_adult = false
  AND EXISTS (SELECT 1 FROM game_contents g
              WHERE g.content_id = c.content_id AND g.review_count >= :reviewCountMin)
ORDER BY c.release_date DESC NULLS LAST, c.content_id ASC
```

**테스트**는 조립기에 9건을 먼저 썼다(RED → GREEN). 무필터면 기본 조건만, 리뷰 축은 최상위 AND EXISTS, 배열 연산자·ILIKE·날짜·웹툰 3축, 빈 값 정규화, count가 WHERE를 공유, 성인 제외 상시 — 그리고 회귀 가드 하나: **SQL 텍스트에 `IS NULL`이 다시 등장하면 실패한다.** 구 `WORKS_FILTER`와 `@Query findWorks`는 삭제했다.

## 6. 측정

### 6-1. DB 레벨 — 같은 재현 방식으로 새 SQL

값을 여전히 `(SELECT …)`로 감싸 바인딩 상황을 유지한 채, 동적 조립이 생성하는 SQL을 측정했다.

```
Limit  (actual time=1623.8..1705.3 rows=20)
  Buffers: shared hit=393 read=33346                 ← 완전 콜드
  I/O Timings: shared read=4250.501
  -> Gather Merge (Workers 2)
     -> Sort (top-N heapsort)
        -> Parallel Hash Join  Hash Cond: (c.content_id = g.content_id)   ← EXISTS가 조인으로 변환됨
             -> Parallel Seq Scan on contents c  (rows=23324 예상, actual 58764 ×3)
                  Filter: ((NOT is_adult) AND ((domain)::text = (InitPlan 1).col1))
                  Rows Removed by Filter: 19309       ← GAME이 아닌 행만 폐기
             -> Parallel Hash
                  -> Parallel Seq Scan on game_contents g  (rows=25244 예상, actual 2920 ×3)
                       Filter: (review_count >= (InitPlan 2).col1)
Execution Time: 1709.388 ms
```

| | Before | After | |
|---|---|---|---|
| 본 쿼리 | 9,872ms | **1,709ms** | 5.8× |
| count | 9,962ms | **908ms** | 11× |
| 요청 1회 | ≈19.8s | **≈2.6s** | 7.6× |

`IS NULL`도 SubPlan도 사라지고 `Hash Join`이 됐다. 다만 3-2의 Nested Loop(1.2초)가 아니라 contents 전체 Seq Scan + Hash Join이 선택됐는데, 이유는 여전히 값을 모르는 `review_count >= (InitPlan)`의 추정(`25,244` vs 실제 `2,920`)이다 — 바인딩의 세 번째, 가장 온화한 영향. 이 재현은 "값을 모르는 상태"를 강제한 보수적 상한이고, 실제 PostgreSQL은 바인딩 쿼리도 처음 5회는 실제 값으로 계획(custom plan)한다. 그래서 E2E가 필요했다.

### 6-2. E2E — 같은 배포에서 구/신 쿼리 나란히

구 쿼리를 원문 그대로 `findWorksLegacy`로 복원하고 `GET /api/works?impl=legacy` 파라미터로만 라우팅하는 임시 경로를 배포했다(측정 후 제거). 같은 서버, 같은 DB, 같은 캐시 상태에서 두 SQL만 다르다. 측정은 운영 백엔드 원본에 `curl`로:

```bash
BASE="https://api.<domain>/api/works?domain=GAME&reviewCountMin=100&page=0&size=20"
for impl in legacy dynamic; do
  [ "$impl" = legacy ] && U="$BASE&impl=legacy" || U="$BASE"
  for i in 1 2 3; do
    curl -s -o /dev/null -w "$impl run$i: total=%{time_total}s bytes=%{size_download}\n" "$U"
  done
done
```

**기본 측정 (`reviewCountMin=100`)**

| | 1회 (콜드) | 2회 | 3회 | 순서 뒤집어 재측정 |
|---|---|---|---|---|
| legacy | **21.06s** | 1.13s | 0.46s | 7.38s → 0.49s |
| dynamic | 1.48s | 1.49s | 1.42s | 1.42s → 1.46s |

legacy 1회차 21초는 DB 재현값 19.8초와 일치했고, 두 경로의 응답 바이트가 같아 결과 동일성도 확인됐다.

**값 스윕 (각 1회, dynamic → legacy 순)**

| `reviewCountMin` | dynamic | legacy |
|---|---|---|
| 500 | 0.63s | 6.79s |
| 1,000 | 0.40s | 8.59s |
| 5,000 | 0.32s | 6.86s |
| 50,000 | 0.26s | 6.09s |

**콜드/웜 매트릭스** (콜드 = 직전에 dynamic(100)으로 캐시를 밀어냄, 이어서 2회가 웜)

| `reviewCountMin` | dynamic 콜드 | dynamic 웜 | legacy 콜드 | legacy 웜1 | legacy 웜2 |
|---|---|---|---|---|---|
| 500 | 0.81s | 0.31s | **10.75s** | 3.28s | 0.37s |
| 5,000 | 0.43s | 0.30s | **7.94s** | 6.42s | 0.35s |
| 50,000 | 0.32s | 0.20s | **5.55s** | 0.32s | 0.34s |

**legacy 반복성 검증**: 연속 5회 5.25 → 1.44 → 0.36 → 0.38 → 0.36s. 75초 대기 후 0.41s(시간만으론 안 식음). 그런데 legacy 0.35s → **dynamic(100) 1회** → legacy **5.51s**. 캐시를 밀어내는 건 시간이 아니라 다른 대량 스캔이다.

### 6-3. E2E가 말해주는 것

- **legacy는 값과 무관하게 평평하고(6~7초), 캐시 운에 따라 쌍봉이다(0.4초 또는 5~21초).** 조건이 빡세도 전수 스캔을 하는 구조라 선택도가 시간에 반영되지 않는다 — 3-5 ①의 E2E 증거. 웜에 도달하려면 전수 스캔을 2~3회 반복해야 하고, 그 캐시는 느슨한 필터 요청 하나·크롤러 배치 한 번에 사라진다. 운영에서 "웜 legacy"는 사용자가 거의 만나지 못하는 상태다.
- **dynamic은 조건에 비례하고 캐시와 무관하다.** 100: 1.45s → 500: 0.63 → 1,000: 0.40 → 5,000: 0.32 → 50,000: 0.26. "조건 맞는 게임부터 찾는" 플랜이라 매칭이 줄면 일도 준다. 콜드 페널티는 0.5초 이하.
- **정직하게 남는 것**: 웜 한정으로 느슨한 100 조건은 legacy(0.46s)가 dynamic(1.45s)보다 빠르다. dynamic(100)의 1.45초는 6-1에서 본 "contents 전 도메인 Seq Scan + Hash Join ×2"이며, `ORDER BY release_date`를 지원하는 인덱스가 없어 매 요청 후보 전체를 읽고 정렬해야 하기 때문이다. 이 스캔은 자기 자신도 캐시 이득을 못 보고 남의 캐시까지 밀어낸다.

## 7. 다음 단계

원인 사슬(구조 → 읽기 양 → 횟수)의 순서대로, 한 단계씩 바꾸고 재측정한다.

| 단계 | 조치 | 상태 |
|---|---|---|
| ① 구조 | `IS NULL OR` 제거, 동적 조립 | 배포 완료 — 21s → 1.4s(콜드) / 0.4s(웜) |
| ② 읽기 양 | `contents (domain, release_date DESC NULLS LAST, content_id) WHERE is_adult = false` + `game_contents (review_count) INCLUDE (content_id)` | PR 올림. 느슨한 필터의 "20건 차면 중단" 플랜과 무필터 탭의 전체 정렬 제거가 목표 |
| ③ 횟수 | count 쿼리 제거(`Slice`) | 보류 — 탐색 페이지가 번호 페이지네이션(`totalPages`)이라 프론트와 함께 결정 |
| ④ 모델 | `review_count`를 `contents`로 승격 | ②로 부족할 때 |

## 8. 배운 것

1. **베스트 케이스 측정의 함정.** 리터럴 EXPLAIN만 믿었다면 "쿼리는 빠른데?"로 끝났다. 제보와 측정이 안 맞으면 측정 조건을 의심한다. 바인딩 쿼리는 `(SELECT 값)`으로 재현한다.
2. **결과가 불가능하면 도구를 의심한다.** `One-Time Filter: false`는 쿼리의 문제가 아니라 클라이언트가 `$1`을 먹은 것이었다. `pg_prepared_statements`가 진실을 보여줬다.
3. **플랜은 정해진 순서로 읽는다.** `Execution Time` → `Buffers hit/read`·`I/O Timings` → 가장 비싼 노드 → `rows` 예상 vs 실제 → `Rows Removed`. 가설은 플랜에 맞춰 버린다(행별 프로브 가설을 한 번 접었다가 전체 쿼리 재현에서 되살렸다).
4. **원인은 연쇄다.** 어느 한 고리만 보면 "인덱스 추가"나 "RDS 증설" 같은 반쪽 처방이 나온다.
5. **편의 설계의 대가는 누군가 치른다.** 단일 쿼리 + NULL 스위치는 자바를 단순하게 했지만 비용을 플래너에 떠넘겼다. 통합 자체는 옳았고, 다음 단계(조건부 조립)가 필요했을 뿐이다.
6. **구 경로를 남겨 A/B 하라.** 같은 배포·같은 DB에서 두 SQL만 다르게 재는 것이 가장 설득력 있는 비교였다.

---

### 부록 A. 재현 쿼리 모음

```sql
-- 인덱스 현황
SELECT tablename, indexname, indexdef FROM pg_indexes WHERE tablename IN ('contents','game_contents');

-- 베스트 케이스 (리터럴)
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.* FROM contents c
WHERE c.domain = 'GAME' AND c.is_adult = false
  AND EXISTS (SELECT 1 FROM game_contents g WHERE g.content_id = c.content_id AND g.review_count >= 1000)
ORDER BY c.release_date DESC NULLS LAST, c.content_id ASC LIMIT 20 OFFSET 0;

-- 바인딩 재현 (스위치 1개 버전 — 실제보다 온화함, 전체 버전은 §3-4)
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.* FROM contents c
WHERE c.domain = 'GAME' AND c.is_adult = false
  AND ((SELECT 1000) IS NULL
       OR EXISTS (SELECT 1 FROM game_contents g WHERE g.content_id = c.content_id AND g.review_count >= (SELECT 1000)))
ORDER BY c.release_date DESC NULLS LAST, c.content_id ASC LIMIT 20 OFFSET 0;

-- 개선 후 SQL (바인딩 재현)
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.* FROM contents c
WHERE c.domain = (SELECT 'GAME'::text) AND c.is_adult = false
  AND EXISTS (SELECT 1 FROM game_contents g WHERE g.content_id = c.content_id AND g.review_count >= (SELECT 1000))
ORDER BY c.release_date DESC NULLS LAST, c.content_id ASC LIMIT 20 OFFSET 0;

-- 측정 도구 의심 시
SELECT name, statement, parameter_types FROM pg_prepared_statements;
SHOW plan_cache_mode;
```

### 부록 B. EXPLAIN 읽기 메모

- `(cost=A..B rows=N width=W)`는 예상, `(actual time=X..Y rows=N loops=L)`는 실측. 실제 총 행 수 = rows × loops.
- `Buffers: shared hit` = 캐시, `shared read` = 디스크. `I/O Timings`는 프로세스별 대기의 합산이라 실행 시간보다 클 수 있다.
- `Index Cond`는 인덱스를 타고 내려갈 때 쓴 조건, `Filter`는 행을 가져온 뒤 검사한 조건. `Rows Removed by Filter`가 크면 "필요보다 많이 읽었다".
- `Filter:` 줄에 `… IS NULL) OR EXISTS(SubPlan …)`이 보이면 이 글의 병이다. `rows=` 예상과 `actual rows`가 수십 배 다르면 추정 붕괴다.

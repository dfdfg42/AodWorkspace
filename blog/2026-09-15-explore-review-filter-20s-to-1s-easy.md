# 탐색 페이지가 20초 걸린 이유를 아주 쉽게 이해하기

> 원문: `2026-09-15-explore-review-filter-20s-to-1s.md`

이 문서는 원문을 처음 보는 사람도 따라올 수 있도록 다시 풀어쓴 버전이다.
PostgreSQL 실행 계획을 읽으면서, 왜 같은 검색 조건이 어떤 때는 1초 안팎이고 어떤 때는 20초 가까이 걸렸는지 설명한다.

---

## 1. 결론부터

AOD 탐색 페이지에서 사용자가 다음 조건을 선택했다고 하자.

```
게임만 보기
성인 콘텐츠 제외
스팀 리뷰 1,000개 이상
최신 출시일 순으로 20개
```

처음 요청은 약 20초가 걸렸다. 프론트엔드 타임아웃이 30초라서 사용자는 “검색이 작동하지 않는다”고 느꼈다.

원인은 다음 SQL 패턴이었다.

`(:reviewCountMin IS NULL OR 조건)`

기능적으로는 편하다.

- 값이 `NULL`이면 필터를 끈다.
- 값이 있으면 필터를 켠다.

하지만 PostgreSQL 플래너는 실행 계획을 만들 때 `:reviewCountMin`이 항상 1000이라고 보장할 수 없다. `NULL`일 수도 있기 때문이다.

그래서 다음처럼 처리했다.

```
GAME 콘텐츠 약 19만 개를 먼저 읽음
→ 각 행마다 리뷰 조건 확인
→ 대부분 버림
→ 남은 행 정렬
→ 20개 반환
```

실제로는 1000을 보냈어도, DB는 “혹시 NULL일 수도 있다”고 생각해야 했다.

해결책은:

> 값이 있는 조건만 SQL에 추가한다.

---

## 2. 테이블 구조와 검색 의미

테이블은 대략 다음처럼 연결되어 있다.

```
contents                  공통 콘텐츠 정보
  ├─ game_contents         게임 전용 정보
  ├─ webtoon_contents      웹툰 전용 정보
  └─ ...
```

두 테이블은 `content_id`로 연결된다.

| content_id | domain | is_adult | review_count |
|---:|---|---|---:|
| 101 | GAME | false | 15,000 |
| 102 | GAME | false | 300 |
| 103 | GAME | true | 20,000 |
| 104 | MOVIE | false | 없음 |

쿼리는 다음 의미다.

```sql
SELECT c.*
FROM contents c
WHERE c.domain = 'GAME'
  AND c.is_adult = false
  AND EXISTS (
      SELECT 1
      FROM game_contents g
      WHERE g.content_id = c.content_id
        AND g.review_count >= 1000
  )
ORDER BY c.release_date DESC NULLS LAST, c.content_id ASC
LIMIT 20;
```

사람 말로 바꾸면:

> 게임이면서 성인 콘텐츠가 아니고, 같은 ID의 게임 정보가 있으며, 리뷰가 1,000개 이상인 콘텐츠를 최신순으로 20개 보여줘.

`EXISTS`는 실제로 숫자 1을 가져오는 표현이 아니다. 조건에 맞는 행이 하나라도 존재하는지만 확인한다.

---

## 3. 실행 계획 용어를 먼저 이해하기

`EXPLAIN (ANALYZE, BUFFERS)`는 SQL을 실제로 실행한 뒤 처리 과정을 보여준다.

### `cost`

`cost=100..200`은 밀리초가 아니다. 여러 실행 계획을 비교하기 위한 상대적인 비용 점수다.

### `actual time`

`actual time=10..120`에서 앞 숫자는 첫 결과가 나온 시간이고 뒤 숫자는 노드 처리가 끝난 시간이다.

### `rows`, `loops`

`rows=2000 loops=3`이면 대략 2,000행을 3번 처리했다는 뜻이다. 병렬 계획에서는 루프당 평균으로 보일 수 있다.

### `Buffers`

```
Buffers: shared hit=30000 read=9000
```

- `hit`: PostgreSQL 캐시에 이미 있던 페이지
- `read`: 캐시에 없어 새로 읽은 페이지

기본 페이지가 8KB라면 `read=9000`은 약 72MB에 해당한다. 같은 페이지를 반복해서 읽으면 반복 횟수만큼 집계될 수 있다.

### `I/O Timings`

페이지를 읽느라 기다린 누적 시간이다. 병렬 작업자의 시간을 합할 수 있어 전체 실행시간보다 클 수도 있다.

---

## 4. 값이 확정된 경우의 좋은 계획

다음처럼 SQL에 1000이 직접 들어 있으면 플래너가 조건을 확실히 안다.

`g.review_count >= 1000`

실행 흐름은 다음과 같다.

```
game_contents 전체 확인
→ 리뷰 1,000개 이상인 약 8,759개 발견
→ 각 content_id로 contents 기본키 조회
→ 최신순 정렬
→ 20개 반환
```

실행 계획은 대략 이렇다.

```
Nested Loop
  -> Parallel Seq Scan on game_contents
       Filter: review_count >= 1000
  -> Index Scan using contents_pkey on contents
       Index Cond: content_id = g.content_id
```

### 4-1. `game_contents`를 먼저 읽는다

`review_count` 인덱스가 없어서 전체를 읽지만, 이 단계에서 8,759개로 후보를 줄인다.

### 4-2. 후보 ID만 `contents`에서 찾는다

`contents_pkey`로 필요한 ID만 약 8,759번 조회한다. 전체 `contents` 23만 행을 읽는 것보다 훨씬 적다.

이처럼 바깥쪽 결과 하나마다 안쪽 인덱스를 조회하는 방식을 `Nested Loop`라고 한다.

### 4-3. 정렬은 작은 비용이다

`Sort Method: top-N heapsort`는 `LIMIT 20`을 보고 최신 20개 후보만 유지하는 방식이다. 메모리 사용량도 약 60KB였다.

---

## 5. `IS NULL OR`를 넣자 계획이 달라졌다

앱에서는 리뷰 필터가 선택 사항이라 다음과 같이 작성했다.

```sql
AND (
    :reviewCountMin IS NULL
    OR EXISTS (
        SELECT 1
        FROM game_contents g
        WHERE g.content_id = c.content_id
          AND g.review_count >= :reviewCountMin
    )
)
```

의미는 정확하다.

```
reviewCountMin이 NULL → 모든 게임 허용
reviewCountMin이 1000 → 리뷰 1000개 이상만 허용
```

하지만 계획을 만드는 순간에는 두 경우를 모두 보존해야 한다.

만약 PostgreSQL이 `EXISTS`를 무조건 조인으로 바꾸면, `NULL`일 때 원래 통과해야 하는 행을 잃을 수 있다. 그래서 `EXISTS`를 필터로 남겨둔다.

결과적으로 먼저 많은 `contents` 행을 읽고, 각 행마다 조건을 검사한다.

---

## 6. 나쁜 실행 계획을 아래에서 위로 읽기

단순화한 한 축 재현은 약 12.6초가 걸렸다. 앱의 전체 쿼리에서는 본문과 `COUNT(*)`가 각각 비슷한 작업을 해서 총 약 20초가 됐다.

### 6-1. `InitPlan`: 1000을 한 번 계산

```
InitPlan 1
  -> Result
     rows=1
```

`(SELECT 1000)`을 한 번 계산했다는 뜻이다. 실제 실행값은 1000이지만, 계획 시점에는 “나중에 계산될 값”처럼 보인다.

### 6-2. `game_contents` 전체 스캔

```
Seq Scan on game_contents g
Filter: review_count >= (InitPlan 4).col1
rows=8759
Rows Removed by Filter: 176796
```

약 18만 행을 확인해 8,759개를 찾는다. 이 작업은 약 0.4초 정도여서 가장 큰 병목은 아니다.

### 6-3. `GAME` 행의 위치를 찾음

```
Bitmap Index Scan on idx_contents_lookup
Index Cond: domain = 'GAME'
rows=191067
```

이 인덱스는 `domain`만 처리한다. `is_adult`와 리뷰 조건은 나중에 테이블 행을 읽은 뒤 검사한다.

즉 “게임인 행의 주소”를 얻은 것이지, 최종 결과만 얻은 것이 아니다.

### 6-4. 실제 테이블을 많이 읽음

```
Bitmap Heap Scan on contents c
actual rows=8758
Rows Removed by Filter: 176797
Heap Blocks: exact=15862
```

DB는 약 19만 개의 GAME 후보를 읽었다. 그중 17만 개 이상을 버리고 8,758개만 남겼다.

```
읽은 후보: 약 185,555행
남은 행: 8,758행
버린 행: 176,797행
```

### 6-5. 시간이 오래 걸린 진짜 이유

```
Buffers: shared hit=6541 read=16067
I/O Timings: shared read=11331.758
```

약 16,067개 페이지를 새로 읽었고, 읽기 대기시간만 약 11.3초였다. 12.6초 중 대부분이 테이블 페이지 읽기다.

반면 정렬은 다음처럼 작았다.

`Sort Method: top-N heapsort Memory: 59kB`

따라서 범인은 정렬이 아니라 “필요한 행보다 훨씬 많은 행을 먼저 읽은 것”이다.

---

## 7. 왜 행별 조회가 수십만 번 생기는가?

플래너의 예상이 실제와 크게 달랐기 때문이다.

```
예상 rows=1
실제 rows=8758
```

플래너가 결과가 1행이라고 생각하면, “각 행을 인덱스로 한 번 찔러보는 방식도 괜찮겠다”고 판단할 수 있다.

하지만 실제로는 후보가 약 19만 개라서 같은 인덱스 조회가 수십만 번 반복된다.

실행 계획에서 다음 같은 줄을 주의해서 봐야 한다.

`Index Scan ... loops=176292`

`Rows Removed by Filter`가 크고 `loops`도 크면 “많이 읽고 많이 버리는” 계획일 가능성이 높다.

---

## 8. 왜 `review_count` 인덱스 하나만으로는 부족한가?

다음 인덱스는 `game_contents`를 찾는 데 도움을 줄 수 있다.

```sql
CREATE INDEX ... ON game_contents (review_count);
```

하지만 이번 실행에서 11초 이상 걸린 부분은 `game_contents` 조건이 아니라 `contents`의 대량 읽기였다.

나쁜 구조는 다음과 같다.

```
contents GAME 행 19만 개 읽기
→ game_contents 조건 확인
```

인덱스를 추가해도 이 순서가 그대로면 `contents`의 대량 읽기는 남을 수 있다. 먼저 좋은 실행 방향을 선택하게 만드는 것이 중요하다.

---

## 9. 해결 방법: 조건부 SQL 조립

기존에는 모든 조건을 넣고 `NULL`로 켰다 껐다 했다.

개선 후에는 자바에서 실제 조건이 있을 때만 SQL 절을 추가한다.

```java
StringBuilder where = new StringBuilder(
    "c.domain = :domain AND c.is_adult = false"
);

params.put("domain", domain);

if (reviewCountMin != null) {
    where.append(" AND EXISTS ("
        + "SELECT 1 FROM game_contents g "
        + "WHERE g.content_id = c.content_id "
        + "AND g.review_count >= :reviewCountMin"
        + ")");
    params.put("reviewCountMin", reviewCountMin);
}
```

리뷰 필터가 없으면:

```sql
WHERE c.domain = :domain
  AND c.is_adult = false
```

리뷰 필터가 있으면:

```sql
WHERE c.domain = :domain
  AND c.is_adult = false
  AND EXISTS (
      SELECT 1
      FROM game_contents g
      WHERE g.content_id = c.content_id
        AND g.review_count >= :reviewCountMin
  )
```

값은 여전히 바인딩 파라미터로 전달한다. 사용자 입력을 SQL 문자열에 직접 이어붙이지 않는다.

### 동적 SQL과 SQL 인젝션은 다르다

위험한 방식:

```java
sql += " AND c.master_title = '" + userInput + "'";
```

안전하게 사용하는 방식:

```java
sql += " AND c.master_title = :keyword";
params.put("keyword", userInput);
```

이번 개선은 두 번째 방식이다. SQL 구조만 조립하고 값은 바인딩한다.

---

## 10. `COUNT(*)` 때문에 한 요청이 두 번 느려진다

Spring Data의 `Page`는 전체 페이지 수를 계산하려고 count 쿼리를 추가로 실행할 수 있다.

```
1. 본문 조회: 최신 콘텐츠 20개
2. count 조회: 전체 결과 개수
```

기존 측정에서는:

```
본문 조회: 약 9.87초
count 조회: 약 9.96초
합계: 약 19.8초
```

전체 페이지 수가 필요 없다면 `Slice`처럼 다음 페이지 존재 여부만 확인하는 방법도 있다. 다만 `totalPages`가 화면에 필요하다면 API 요구사항과 함께 결정해야 한다.

---

## 11. 캐시 때문에 실행시간이 흔들리는 이유

첫 실행에서 읽은 페이지는 캐시에 들어갈 수 있다. 두 번째 실행은 디스크가 아니라 캐시에서 읽어서 빨라진다.

하지만 대량 스캔이 반복되면 다른 페이지가 캐시에서 밀려난다.

```
첫 요청: 21초
두 번째 요청: 1초
세 번째 요청: 0.4초
다른 대량 스캔 직후: 다시 5~10초
```

따라서 웜 캐시에서 한 번 빨랐다는 사실만으로 쿼리가 좋은 것은 아니다. 운영 환경에서는 사용자 요청과 배치 작업이 캐시를 계속 바꾼다.

---

## 12. 측정할 때 생긴 함정

### 리터럴 SQL과 앱 SQL은 다를 수 있다

`g.review_count >= 1000`과 `g.review_count >= ?`는 같은 값으로 실행돼도 플래너가 다른 계획을 선택할 수 있다.

리터럴 `EXPLAIN`만 보고 앱도 빠를 것이라고 결론내리면 안 된다.

### 클라이언트가 `$1`을 가로챌 수 있다

DBeaver 같은 도구에서 `$1`, `$2`를 사용하면 도구가 이를 자체 변수로 처리할 수 있다. DB에는 `c.domain = NULL`이 전달되어 결과가 항상 0행이 될 수도 있다.

실제 서버에 등록된 문장은 다음으로 확인한다.

```sql
SELECT name, statement, parameter_types
FROM pg_prepared_statements;

SHOW plan_cache_mode;
```

`(SELECT 1000)` 방식은 플래너가 값을 실행 시점 값처럼 취급하게 만들어 바인딩 상황을 재현하는 데 사용할 수 있다. 다만 앱의 전체 SQL과 모든 필터를 함께 재현해야 실제와 가까워진다.

---

## 13. 개선 결과

조건부 SQL 조립 후에는:

```
IS NULL OR 조건 사라짐
→ EXISTS가 세미조인/해시 조인으로 바뀔 수 있음
→ 행별 SubPlan 반복 감소
→ 불필요한 contents 읽기 감소
```

측정 예시는 다음과 같다.

| 구분 | 개선 전 | 개선 후 |
|---|---:|---:|
| 본문 쿼리 | 약 9,872ms | 약 1,709ms |
| count 쿼리 | 약 9,962ms | 약 908ms |
| 한 요청 합계 | 약 19.8초 | 약 2.6초 |

운영 E2E 측정도 리뷰 기준이 높아질수록 동적 SQL이 유리했다.

| reviewCountMin | 동적 SQL | 기존 SQL |
|---:|---:|---:|
| 500 | 0.63초 | 6.79초 |
| 1,000 | 0.40초 | 8.59초 |
| 5,000 | 0.32초 | 6.86초 |
| 50,000 | 0.26초 | 6.09초 |

---

## 14. 추가로 검토할 인덱스

구조를 고친 뒤에는 다음 인덱스를 검토할 수 있다.

```sql
CREATE INDEX CONCURRENTLY idx_contents_game_public_release
ON contents (
    release_date DESC NULLS LAST,
    content_id ASC
)
WHERE domain = 'GAME'
  AND is_adult = false;
```

이 인덱스가 있으면 최신 순서로 읽다가 20개를 채운 뒤 멈추는 계획이 가능해질 수 있다.

리뷰 기준이 항상 1,000으로 고정이라면 다음도 검토할 수 있다.

```sql
CREATE INDEX CONCURRENTLY idx_game_contents_reviewed
ON game_contents (content_id)
WHERE review_count >= 1000;
```

리뷰 기준이 매번 바뀐다면:

```sql
CREATE INDEX CONCURRENTLY idx_game_contents_review_count_content
ON game_contents (review_count, content_id);
```

다만 인덱스는 쿼리 구조를 고친 뒤 실제 `EXPLAIN (ANALYZE, BUFFERS)`로 효과를 확인해야 한다.

---

## 15. 실행 계획을 읽는 순서

다음 순서로 보면 된다.

1. `Execution Time`: 전체 실행시간
2. `Buffers read`와 `I/O Timings`: 디스크 읽기 비용
3. `Seq Scan`, `Bitmap Heap Scan`: 많은 행을 읽은 노드
4. `Rows Removed by Filter`: 읽고 버린 행 수
5. `예상 rows`와 `actual rows`의 차이
6. `Index Scan loops`: 반복 조회 횟수

특히 조건이 `Filter`에만 있고 `Rows Removed by Filter`가 크다면, 필요한 것보다 많은 데이터를 읽고 있는 것이다.

---

## 16. 최종 정리

문제의 연쇄는 다음과 같다.

```
여러 필터를 하나의 SQL로 통합
→ :param IS NULL OR 조건 사용
→ 플래너가 파라미터 값을 확정하지 못함
→ EXISTS를 좋은 조인으로 바꾸지 못함
→ contents의 많은 행을 먼저 읽음
→ 대부분의 행을 버림
→ 디스크 I/O 증가
→ 본문과 count가 각각 느려짐
→ 요청 전체가 약 20초
```

개선 후에는:

```
실제로 켜진 필터만 SQL에 추가
→ DB가 조건의 존재를 계획 시점에 앎
→ EXISTS를 조인으로 최적화할 수 있음
→ 불필요한 행 읽기 감소
→ 디스크 I/O 감소
→ 약 1~2초 수준으로 개선
```

가장 중요한 교훈은 이것이다.

> SQL을 짧게 만들려고 모든 선택적 조건을 `IS NULL OR`로 감싸면 애플리케이션 코드는 단순해지지만, 그 비용이 데이터베이스 실행 계획으로 넘어갈 수 있다.

실행 계획에서는 “인덱스를 탔는가?”만 보지 말고 다음을 함께 본다.

```text
몇 행을 읽었는가?
몇 행을 버렸는가?
디스크에서 얼마나 읽었는가?
같은 인덱스를 몇 번 반복 호출했는가?
예상 행 수와 실제 행 수가 얼마나 다른가?
```

이 다섯 가지를 함께 봐야 쿼리가 실제로 왜 느린지 알 수 있다.

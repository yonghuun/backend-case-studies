# 관광지 키워드 검색 성능 분석 기록

> 대상: MySQL 8.4 / InnoDB / utf8mb4 · Spring Boot 3.3 + MyBatis
> 목적: count 쿼리 최적화 이후 남은 키워드 검색 병목 분석과 후속 개선 기록

## 0. 배경

선행 작업에서는 관광지 목록/필터 조회의 주요 병목이 `count` 쿼리의 불필요한 JOIN임을 확인하고, `search` 쿼리와 `count` 쿼리의 FROM 절을 분리했다.

선행 문서:

- [관광지 조회 API 성능 분석 및 개선 기록](./attraction-query-optimization.md)

그 결과 일반 목록/필터 조회는 크게 개선되었다.

| 조회 조건 | 개선 전 total | 개선 후 total | 감소율 |
| --- | --- | --- | --- |
| 전체 조회 | 506ms | 52ms | 약 89.7% |
| 시도 필터 | 144ms | 22ms | 약 84.7% |
| 시도 + 구군 필터 | 35ms | 16ms | 약 54.3% |
| 콘텐츠 타입 필터 | 223ms | 38ms | 약 83.0% |

하지만 이후 키워드 검색 요청에서 다시 900ms대 응답 지연이 발생했다. 따라서 이번 문서에서는 선행 개선 이후에도 남아 있던 `keyword` 검색 병목을 별도로 분석한다.

## 1. 문제 상황

관광지 검색 API에서 키워드 검색 시 응답 지연이 발생했다. 서비스 레이어에 구간별 성능 로그를 추가해 전체 처리 시간, 목록 조회 쿼리 시간, count 쿼리 시간을 분리해서 측정했다.

측정 결과 키워드 검색에서는 `search` 쿼리와 `count` 쿼리가 모두 느렸고, 그중 `search` 쿼리가 가장 큰 병목으로 확인됐다.

| keyword | total | search | count | resultCount | totalCount |
| --- | --- | --- | --- | --- | --- |
| store | 983ms | 585ms | 397ms | 6 | 16 |
| cafe | 940ms | 552ms | 388ms | 6 | 416 |

캡처 자료:

- Slow attraction search 로그: `store`

![keyword store slow log](./attraction-search-performance-images/keyword-log-store.png)

- Slow attraction search 로그: `cafe`

![keyword cafe slow log](./attraction-search-performance-images/keyword-log-cafe.png)

## 2. 현재 검색 SQL

현재 키워드 검색은 `title`, `addr1`, `addr2` 세 컬럼에 `LIKE '%keyword%'` 조건을 OR로 적용한다. count 쿼리의 JOIN은 이미 제거했지만, 키워드 조건 자체는 `search`와 `count`에서 공통으로 사용된다.

```sql
a.title LIKE CONCAT('%', #{keyword}, '%')
OR a.addr1 LIKE CONCAT('%', #{keyword}, '%')
OR a.addr2 LIKE CONCAT('%', #{keyword}, '%')
```

목록 조회(`search`)는 검색 조건에 더해 화면 표시용 지역명/구군명/콘텐츠 타입명을 가져오기 위해 조인을 수행한다.

```sql
FROM attractions a
LEFT JOIN sidos s
       ON a.area_code = s.sido_code
LEFT JOIN guguns g
       ON a.area_code = g.sido_code
      AND a.si_gun_gu_code = g.gugun_code
LEFT JOIN contenttypes ct
       ON a.content_type_id = ct.content_type_id
```

캡처 자료:

- `AttractionMapper.xml`의 `LIKE '%keyword%'` 검색 조건

![keyword like condition](./attraction-search-performance-images/keyword-like-condition.png)

- `AttractionMapper.xml`의 `search` 쿼리 조인 구조

![keyword search join](./attraction-search-performance-images/keyword-search-join.png)

## 3. 실행 계획 확인

`EXPLAIN`으로 현재 검색 쿼리의 실행 계획을 확인했다.

### 3.1 store

```sql
EXPLAIN
SELECT
    a.no,
    a.title,
    a.first_image1 AS image_url,
    s.sido_name AS sido,
    g.gugun_name AS gugun,
    ct.content_type_name AS content_type,
    CONCAT_WS(' ', a.addr1, a.addr2) AS address,
    a.latitude,
    a.longitude
FROM attractions a
LEFT JOIN sidos s
       ON a.area_code = s.sido_code
LEFT JOIN guguns g
       ON a.area_code = g.sido_code
      AND a.si_gun_gu_code = g.gugun_code
LEFT JOIN contenttypes ct
       ON a.content_type_id = ct.content_type_id
WHERE (
    a.title LIKE '%store%'
    OR a.addr1 LIKE '%store%'
    OR a.addr2 LIKE '%store%'
)
ORDER BY a.no DESC
LIMIT 6
OFFSET 0;
```

![store explain result](./attraction-search-performance-images/explain-keyword-store.png)

### 3.2 cafe

```sql
EXPLAIN
SELECT
    a.no,
    a.title,
    a.first_image1 AS image_url,
    s.sido_name AS sido,
    g.gugun_name AS gugun,
    ct.content_type_name AS content_type,
    CONCAT_WS(' ', a.addr1, a.addr2) AS address,
    a.latitude,
    a.longitude
FROM attractions a
LEFT JOIN sidos s
       ON a.area_code = s.sido_code
LEFT JOIN guguns g
       ON a.area_code = g.sido_code
      AND a.si_gun_gu_code = g.gugun_code
LEFT JOIN contenttypes ct
       ON a.content_type_id = ct.content_type_id
WHERE (
    a.title LIKE '%cafe%'
    OR a.addr1 LIKE '%cafe%'
    OR a.addr2 LIKE '%cafe%'
)
ORDER BY a.no DESC
LIMIT 6
OFFSET 0;
```

![cafe explain result](./attraction-search-performance-images/explain-keyword-cafe.png)

### 3.3 EXPLAIN 결과 요약

두 키워드 모두 `attractions` 테이블에서 인덱스를 사용하지 못하고 전체 스캔이 발생했다.

| keyword | table | type | possible_keys | key | rows | Extra |
| --- | --- | --- | --- | --- | --- | --- |
| store | a | ALL | NULL | NULL | 826390 | Using where; Using temporary; Using filesort |
| cafe | a | ALL | NULL | NULL | 826390 | Using where; Using temporary; Using filesort |

해석:

- `type = ALL`: `attractions` 테이블 전체 스캔
- `key = NULL`: 검색에 사용된 인덱스 없음
- `rows = 826390`: 약 82만 건 스캔 예상
- `Using temporary`: 정렬/처리를 위한 임시 테이블 사용
- `Using filesort`: `ORDER BY a.no DESC` 처리에 추가 정렬 비용 발생

## 4. 원인 판단

검색 지연의 주요 원인은 `LIKE '%keyword%'` 조건이다. 앞쪽에 `%`가 붙은 포함 검색은 일반 B-Tree 인덱스를 효율적으로 사용할 수 없고, 현재 검색 대상이 `title`, `addr1`, `addr2` 세 컬럼 OR 조건으로 묶여 있어 스캔 범위가 커진다.

결과적으로 목록 조회 시 다음 비용이 함께 발생한다.

- `attractions` 전체 스캔
- 세 문자열 컬럼에 대한 포함 검색
- 지역/구군/콘텐츠 타입 조인
- `ORDER BY a.no DESC` 정렬
- 페이지네이션을 위한 `LIMIT/OFFSET`

선행 작업에서 count 쿼리의 불필요한 JOIN은 제거되었지만, `LIKE '%keyword%'` 조건은 여전히 `attractions` 테이블의 많은 row를 검사하게 만든다. 따라서 이번 병목은 JOIN 제거만으로 해결하기 어려운 문자열 검색 방식의 문제로 분류한다.

## 5. 개선 후보

| 후보 | 내용 | 장점 | 단점 |
| --- | --- | --- | --- |
| 검색 대상 축소 | 우선 `title`만 검색 | 구현이 단순하고 즉시 빨라질 수 있음 | 주소 검색 기능 축소 |
| Prefix 검색 | `LIKE 'keyword%'` + 일반 인덱스 | B-Tree 인덱스 활용 가능 | 중간 포함 검색 불가 |
| FULLTEXT 검색 | `FULLTEXT(title, addr1, addr2)` + `MATCH AGAINST` | 문자열 검색 전용 인덱스 활용, 포트폴리오 설명 가치 높음 | 한국어 검색은 ngram parser 등 추가 고려 필요 |

현재 프로젝트의 포트폴리오 가치와 성능 개선 경험을 고려하면 `FULLTEXT` 인덱스 기반 개선을 우선 검토한다. 다만 한국어 검색까지 안정적으로 다루려면 MySQL `ngram parser`, 최소 검색어 길이, 프론트 debounce/요청 취소 정책도 함께 검토해야 한다.

## 6. 개선 적용

키워드 검색 부하를 줄이기 위해 입력 제어와 검색 인덱스 개선을 함께 적용했다.

### 6.1 최소 검색어 길이 정책

프론트엔드와 백엔드에서 최소 검색어 길이를 2글자로 맞췄다.

```
0~1글자 keyword: 검색어로 사용하지 않음
2글자 이상 keyword: 키워드 검색 수행
```

프론트엔드는 API 요청을 만들 때 2글자 미만 keyword를 `null`로 처리한다.

```js
const MIN_KEYWORD_LENGTH = 2

const normalizedKeyword = keyword.value.trim()

keyword: normalizedKeyword.length >= MIN_KEYWORD_LENGTH ? normalizedKeyword : null
```

백엔드는 외부 클라이언트가 1글자 keyword를 직접 보내는 경우까지 방어하기 위해 request normalize 단계에서 같은 정책을 적용한다.

```java
private static final int MIN_KEYWORD_LENGTH = 2;

if (keyword.length() < MIN_KEYWORD_LENGTH) {
    keyword = null;
}
```

이 정책은 `c`, `서`처럼 검색 범위가 넓고 품질이 낮은 한 글자 검색이 DB까지 전달되는 것을 막는다.

### 6.2 FULLTEXT 인덱스 추가

`LIKE '%keyword%'` 검색이 일반 B-tree 인덱스를 활용하지 못하는 문제를 해결하기 위해 `title`, `addr1`, `addr2` 컬럼에 FULLTEXT 인덱스를 추가했다.

```sql
ALTER TABLE attractions
    ADD FULLTEXT INDEX ft_attractions_keyword (title, addr1, addr2);
```

![fulltext index create](./attraction-search-performance-images/fulltext-index-create.png)

스키마 변경은 기존 `V1__init.sql`을 수정하지 않고 Flyway V2 migration으로 추가했다.

```
V2__add_attraction_fulltext_index.sql
```

![fulltext index show](./attraction-search-performance-images/fulltext-index-show.png)

### 6.3 검색 조건 변경

기존 LIKE 조건을 FULLTEXT 검색으로 변경했다. `NATURAL LANGUAGE MODE`로 테스트했을 때 `store` 검색 결과가 0건으로 나와 기존 LIKE 검색과 차이가 컸다.

```sql
MATCH(title, addr1, addr2)
AGAINST('store' IN NATURAL LANGUAGE MODE)
```

따라서 prefix 검색이 가능한 `BOOLEAN MODE`와 wildcard를 사용했다.

```sql
MATCH(title, addr1, addr2)
AGAINST('store*' IN BOOLEAN MODE)
```

MyBatis Mapper에서는 keyword 뒤에 `*`를 붙여 전달하도록 변경했다.

```xml
AND MATCH(a.title, a.addr1, a.addr2)
    AGAINST(CONCAT(#{keyword}, '*') IN BOOLEAN MODE)
```

## 7. 개선 후 실행 계획

FULLTEXT 적용 후 `EXPLAIN`에서 `attractions` 테이블이 `ft_attractions_keyword` 인덱스를 사용하는 것을 확인했다.

| keyword | table | type | key | Extra |
| --- | --- | --- | --- | --- |
| store | a | fulltext | ft_attractions_keyword | Using where; Ft_hints: no_ranking; Using filesort |
| cafe | a | fulltext | ft_attractions_keyword | Using where; Ft_hints: no_ranking; Using filesort |

기존 `LIKE '%keyword%'` 검색에서는 `type=ALL`, `key=NULL`로 전체 스캔이 발생했지만, 개선 후에는 `type=fulltext`로 검색 전용 인덱스를 사용한다.

![fulltext explain store](./attraction-search-performance-images/fulltext-explain-store.png)

![fulltext explain cafe](./attraction-search-performance-images/fulltext-explain-cafe.png)

## 8. 개선 후 측정 기록

동일한 키워드로 API 로그를 다시 확인했다.

| keyword | 개선 전 total | 개선 후 total | 개선 전 search | 개선 후 search | 개선 전 count | 개선 후 count | totalCount 변화 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| store | 523ms | 4ms | 169ms | 3ms | 354ms | 1ms | 16 -> 16 |
| cafe | 940ms | 34ms | 552ms | 32ms | 388ms | 1ms | 416 -> 352 |

![fulltext after log store](./attraction-search-performance-images/fulltext-after-log-store.png)

![fulltext after log cafe](./attraction-search-performance-images/fulltext-after-log-cafe.png)

`store`는 기존 LIKE 검색과 동일하게 16건을 반환하면서 total 시간이 523ms에서 4ms로 감소했다. `cafe`는 total 시간이 940ms에서 34ms로 감소했고, count 시간은 388ms에서 1ms로 감소했다.

다만 FULLTEXT 검색은 LIKE 포함 검색과 동작 방식이 다르기 때문에 결과 수가 일부 달라질 수 있다. 실제로 `cafe`는 기존 LIKE 기준 416건에서 FULLTEXT BOOLEAN MODE 기준 352건으로 변경되었다.

## 9. 최종 정리

1차 최적화로 일반 목록/필터 조회의 count 병목은 해결했다. 그러나 키워드 검색은 `LIKE '%keyword%'` 기반 부분 일치 검색 때문에 별도의 병목이 남아 있었다.

이번 개선에서는 최소 검색어 길이 정책으로 불필요한 짧은 키워드 검색을 방어하고, FULLTEXT 인덱스와 `MATCH ... AGAINST(... IN BOOLEAN MODE)` 검색으로 문자열 검색 조건이 인덱스를 사용하도록 변경했다.

그 결과 `store`, `cafe` 기준으로 `search`와 `count` 시간이 모두 크게 감소했다. 특히 키워드 검색의 count 쿼리는 300ms대에서 1ms 수준으로 개선되었다.

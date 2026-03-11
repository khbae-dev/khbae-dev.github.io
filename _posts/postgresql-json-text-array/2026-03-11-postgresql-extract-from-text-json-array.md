---
title: PostgreSQL text 컬럼의 JSON 배열 문자열에서 값 추출하는 방법
date: 2026-03-11 17:10:00 +09:00
tags: [postgresql, json, jsonb, sql, database]
description: PostgreSQL에서 text 컬럼에 저장된 `[{}]` 형태의 JSON 문자열을 jsonb로 캐스팅한 뒤 measurement와 value를 추출하는 실무 패턴을 정리했다.
---

실무에서 꽤 자주 만나는 데이터 형태가 있습니다. 컬럼 타입은 `text`인데 실제 값은 JSON 문자열이고, 그 안에는 배열과 객체가 들어 있는 경우입니다.

예를 들면 아래처럼 저장돼 있습니다.

```text
[{"measurement":"metric-a","value":10}]
```

또는

```text
[{"measurement":"metric-b","value":0.43}]
```

이럴 때 중요한 포인트는 하나입니다.
`text` 컬럼에는 바로 JSON 연산자를 쓸 수 없기 때문에 먼저 `json` 또는 `jsonb`로 캐스팅해야 합니다.

이번 글에서는 `[{}]` 구조의 JSON 문자열에서 값을 꺼낼 때 가장 많이 쓰는 PostgreSQL 패턴만 실무 기준으로 정리합니다.

## 1. 먼저 데이터 구조부터 확인

예를 들어 컬럼이 아래처럼 생겼다고 가정하겠습니다.

```sql
CREATE TABLE table_name (
    id bigint,
    col_data text
);
```

`col_data` 값은 이런 형태입니다.

```text
[{"measurement":"metric-a","value":23.4}]
```

즉 구조는 아래와 같습니다.

- 바깥은 JSON 배열
- 배열 안에는 객체가 1개 이상 들어 있음
- 객체 안에 `measurement`, `value` 같은 키가 있음

그래서 접근 순서는 항상 같습니다.

1. `text`를 `jsonb`로 캐스팅한다
2. 배열의 몇 번째 요소인지 고른다
3. 객체 안의 키를 꺼낸다

## 2. 가장 기본적인 방법: 캐스팅 후 첫 번째 요소 접근

배열의 첫 번째 객체에서 `measurement`를 꺼내려면 아래처럼 작성합니다.

```sql
SELECT
    col_data::jsonb -> 0 ->> 'measurement' AS measurement
FROM table_name;
```

연산자 의미는 아래와 같습니다.

| 표현식 | 의미 |
| --- | --- |
| `::jsonb` | `text`를 `jsonb`로 변환 |
| `-> 0` | 배열의 첫 번째 요소 선택 |
| `->> 'measurement'` | 해당 키의 값을 text로 추출 |

예를 들어 값이 아래와 같다면

```text
[{"measurement":"metric-a","value":10}]
```

결과는 아래처럼 나옵니다.

```text
metric-a
```

## 3. `value`도 같이 꺼낼 수 있다

같은 방식으로 `value`도 바로 추출할 수 있습니다.

```sql
SELECT
    col_data::jsonb -> 0 ->> 'measurement' AS measurement,
    (col_data::jsonb -> 0 ->> 'value')::numeric AS value
FROM table_name;
```

여기서 `->>`는 text를 반환하므로 숫자로 쓰려면 `numeric`, `int`, `double precision` 같은 타입으로 한 번 더 캐스팅하는 편이 안전합니다.

## 4. `json`보다 `jsonb`를 더 많이 쓴다

아래처럼 `json`으로 캐스팅해도 동작은 같습니다.

```sql
SELECT
    col_data::json -> 0 ->> 'measurement' AS measurement
FROM table_name;
```

다만 실무에서는 대부분 `jsonb`를 더 선호합니다.

- 연산과 함수 지원이 더 풍부함
- 이후 조건 검색이나 인덱스 확장에 유리함
- 같은 컬럼을 반복해서 다룰 때 운영 측면에서 더 낫다

그래서 특별한 이유가 없으면 아래처럼 `jsonb` 기준으로 가져가는 편이 무난합니다.

```sql
SELECT
    col_data::jsonb -> 0 ->> 'measurement' AS measurement
FROM table_name;
```

## 5. 배열 안에 객체가 여러 개라면 펼쳐서 조회

값이 항상 객체 1개만 들어 있는 것은 아닙니다.

예를 들어 아래처럼 여러 측정값이 함께 들어 있을 수 있습니다.

```text
[
  {"measurement":"metric-a","value":10},
  {"measurement":"metric-b","value":22}
]
```

이럴 때는 `jsonb_array_elements`로 배열을 행 단위로 펼칩니다.

```sql
SELECT
    elem ->> 'measurement' AS measurement,
    (elem ->> 'value')::numeric AS value
FROM table_name,
jsonb_array_elements(col_data::jsonb) AS elem;
```

결과는 아래처럼 나옵니다.

```text
metric-a     10
metric-b     22
```

즉 `[{}]` 구조를 "여러 행"처럼 다루고 싶을 때 가장 자주 쓰는 패턴이 `jsonb_array_elements`입니다.

## 6. 특정 measurement의 value만 가져오기

실무에서는 보통 "배열 안에서 특정 측정값만 골라서 value를 가져오고 싶다"는 요구가 많습니다.

예를 들어 실제 데이터가 아래와 같다고 하겠습니다.

```text
[{"measurement":"metric-c","value":0.43}]
```

그러면 특정 measurement의 값을 추출하는 쿼리는 아래처럼 작성할 수 있습니다.

```sql
SELECT
    (
        SELECT (elem ->> 'value')::numeric
        FROM jsonb_array_elements(col_data::jsonb) AS elem
        WHERE elem ->> 'measurement' = 'metric-c'
        LIMIT 1
    ) AS metric_c_value
FROM table_name;
```

배열 안에 순서가 항상 같다고 보장되지 않는다면 `-> 0`으로 고정 접근하는 방식보다 이 패턴이 더 안전합니다.

## 7. 첫 번째 요소가 특정 measurement인지 바로 필터링

만약 첫 번째 요소만 보면 충분한 구조라면 `WHERE` 절에서도 바로 비교할 수 있습니다.

```sql
SELECT *
FROM table_name
WHERE col_data::jsonb -> 0 ->> 'measurement' = 'metric-a';
```

이 방식은 구조가 단순할 때 빠르게 확인하기 좋습니다.
다만 배열 안 순서가 바뀔 수 있다면 `jsonb_array_elements` 기반으로 조건을 거는 쪽이 더 안전합니다.

## 8. 키 이름이 `measure`와 `measurement`로 섞여 있다면

현장에서 의외로 자주 생기는 문제가 키 이름이 일관되지 않은 경우입니다.

예를 들어 어떤 데이터는 아래처럼 들어옵니다.

```text
[{"measure":"metric-a","value":23.4}]
```

이럴 때는 `COALESCE`로 두 키를 함께 처리할 수 있습니다.

```sql
SELECT
    COALESCE(
        col_data::jsonb -> 0 ->> 'measurement',
        col_data::jsonb -> 0 ->> 'measure'
    ) AS measurement_name
FROM table_name;
```

스키마를 통일하는 것이 가장 좋지만, 이미 적재된 데이터가 섞여 있다면 이런 방식으로 완충하는 경우가 많습니다.

## 9. 컬럼이 계속 JSON 문자열이라면 타입 변경을 검토

`text` 컬럼인데 실제로는 JSON만 저장하고 있다면 매 쿼리마다 캐스팅이 들어갑니다.
조회가 많아질수록 불편하고, 쿼리도 장황해집니다.

가능하다면 아예 `jsonb` 타입으로 바꾸는 편이 낫습니다.

```sql
ALTER TABLE table_name
ALTER COLUMN col_data TYPE jsonb
USING col_data::jsonb;
```

이후에는 캐스팅 없이 바로 조회할 수 있습니다.

```sql
SELECT
    col_data -> 0 ->> 'measurement' AS measurement
FROM table_name;
```

## 10. 실무에서 같이 보는 체크포인트

쿼리 자체보다 더 중요한 포인트도 있습니다.

- `text` 컬럼 안의 값이 모두 유효한 JSON인지 먼저 확인할 것
- 배열 안 객체 개수가 항상 1개인지 확인할 것
- `value`를 숫자로 계산할 예정이면 text가 아닌 숫자 타입으로 다시 캐스팅할 것
- JSON 구조를 계속 쓸 거라면 장기적으로는 `jsonb` 컬럼 전환을 검토할 것

특히 잘못된 JSON 문자열이 한 행이라도 섞여 있으면 캐스팅 시점에 쿼리 전체가 실패할 수 있으니, 운영 데이터에서는 정합성 확인이 먼저입니다.

## 11. 정리

핵심은 단순합니다.

- `text` 컬럼이면 먼저 `jsonb`로 캐스팅한다
- `[{}]` 구조면 `-> 0`으로 첫 객체에 접근한다
- 여러 객체가 있으면 `jsonb_array_elements`로 펼친다
- 특정 항목의 값이 필요하면 `measurement` 조건으로 골라낸다

가장 자주 쓰는 한 줄은 아래입니다.

```sql
SELECT col_data::jsonb -> 0 ->> 'measurement'
FROM table_name;
```

그리고 실제 업무에서 더 많이 쓰는 패턴은 아래입니다.

```sql
SELECT
    elem ->> 'measurement' AS measurement,
    (elem ->> 'value')::numeric AS value
FROM table_name,
jsonb_array_elements(col_data::jsonb) AS elem;
```

`text`에 JSON 문자열을 저장하는 구조는 당장은 편해 보여도, 조회가 늘어나면 결국 캐스팅 비용과 가독성 문제가 같이 옵니다.
그래서 자주 쓰는 데이터라면 처음부터 `jsonb`로 가져가는 것이 보통 더 낫습니다.

---
title: jsonb 검색이 풀스캔이 되는 이유와 GIN 인덱스
date: 2026-06-30 19:24:00 +09:00
categories: [Database]
tags: [postgresql, jsonb, gin, index]
description: B-tree 인덱스가 jsonb 내부 키를 탐색하지 못하는 이유를 이해하고, GIN 인덱스로 jsonb 검색을 해결하는 방법을 정리했다.
---

jsonb 컬럼에 조건을 걸면 인덱스가 있어도 풀스캔이 나가는 경우가 있습니다. 이유를 정리해 보니 인덱스 종류의 문제였습니다.

## B-tree는 jsonb 내부를 모른다

B-tree 인덱스는 컬럼 값 전체의 대소 비교로 동작합니다. jsonb 컬럼에 B-tree를 걸면 "jsonb 문서 전체"가 하나의 값으로 정렬될 뿐, 문서 안의 특정 키를 찾는 `@>` 같은 연산에는 쓰이지 못합니다.

```sql
-- 이 쿼리는 attrs에 B-tree가 있어도 풀스캔
SELECT * FROM products
WHERE attrs @> '{"brand": "acme"}';
```

## GIN 인덱스로 해결

GIN(Generalized Inverted Index)은 역색인 구조라서 jsonb 문서 안의 키·값 각각을 색인합니다. `@>`, `?`, `?|` 같은 jsonb 연산자가 인덱스를 탈 수 있게 됩니다.

```sql
CREATE INDEX idx_products_attrs ON products USING GIN (attrs);

-- 특정 키 경로만 자주 찾는다면 표현식 인덱스도 방법
CREATE INDEX idx_products_brand ON products ((attrs->>'brand'));
```

수 초씩 걸리던 jsonb 조건 검색이 수십 ms로 내려갑니다. 문서가 크고 행이 많을수록 차이가 커집니다.

## 선택 기준

- 문서 내 임의 키를 다양하게 검색: `USING GIN (attrs)`
- 특정 키 하나만 반복 검색: `(attrs->>'key')` 표현식 B-tree가 더 작고 빠름
- 쓰기 비중이 아주 높은 테이블은 GIN 갱신 비용(pending list)도 고려

## 정리

- 인덱스는 "있냐"가 아니라 "그 연산자를 지원하냐"가 핵심이다.
- jsonb 포함 검색은 GIN, 단일 키 추출 검색은 표현식 인덱스.
- 적용 전후를 EXPLAIN ANALYZE로 확인하는 습관까지가 한 세트다.

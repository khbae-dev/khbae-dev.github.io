---
title: 느린 쿼리 진단의 시작, EXPLAIN ANALYZE로 Seq Scan 찾기
date: 2026-06-04 02:34:00 +09:00
categories: [Database]
tags: [database, explain, index, postgresql]
description: 느린 쿼리를 감이 아니라 실행계획으로 진단하는 기본 절차를 정리했다. Seq Scan 확인부터 인덱스 생성, 개선 검증까지.
---

쿼리 튜닝의 첫 단추를 다시 정리했습니다. 어떤 고급 기법보다 먼저 할 일은 실행계획을 눈으로 확인하는 것입니다.

## 진단: 실제로 어떻게 실행됐는지 본다

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_email = 'a@b.com';
```

```
Seq Scan on orders  (cost=0.00..25890.00 rows=1 width=98)
                    (actual time=812.4..812.5 rows=3 loops=1)
  Filter: (customer_email = 'a@b.com')
  Rows Removed by Filter: 999997
```

읽는 법의 핵심만 짚으면 이렇습니다.

- `Seq Scan` = 테이블 전체를 처음부터 끝까지 읽었다는 뜻
- `Rows Removed by Filter`가 크다 = 읽은 것 대부분을 버렸다, 인덱스가 필요하다는 신호
- `actual time`이 진짜 소요 시간 (cost는 추정치)

## 처방: 조회 조건 컬럼에 인덱스

```sql
CREATE INDEX idx_orders_email ON orders (customer_email);
```

다시 EXPLAIN ANALYZE를 돌리면 `Index Scan using idx_orders_email`로 바뀌고, 실행 시간이 수백 ms에서 밀리초 단위로 내려갑니다. 백만 행 기준 100배 안팎의 차이가 흔합니다.

물론 인덱스가 만능은 아닙니다. 쓰기마다 인덱스 갱신 비용이 붙고, 카디널리티가 낮은 컬럼(성별 등)에는 효과가 작습니다. 그래서 "자주 조회되고, 선택도가 높은 컬럼"부터 거는 게 원칙입니다.

## 정리

- 튜닝은 추측이 아니라 EXPLAIN ANALYZE에서 시작한다.
- Seq Scan + 큰 Rows Removed by Filter = 인덱스 후보.
- 인덱스 추가 후 같은 방법으로 개선을 검증해야 한 사이클이 끝난다.

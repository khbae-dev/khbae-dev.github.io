---
title: 수십억 행 테이블에서 파티션 프루닝으로 스캔 범위 줄이기
date: 2026-07-03 19:22:00 +09:00
categories: [Database]
tags: [database, partitioning, postgresql, performance]
description: 조건을 걸어도 느린 초대형 테이블 쿼리를 RANGE 파티셔닝과 파티션 프루닝으로 해결하는 원리를 정리했다.
---

행이 수십억 개인 테이블은 인덱스가 있어도 쿼리가 무겁습니다. 인덱스 깊이가 깊어지고, 오래된 데이터까지 한 덩어리로 관리되기 때문입니다. 이럴 때 쓰는 정석이 RANGE 파티셔닝이라는 걸 정리해 봤습니다.

## 파티셔닝이 해결하는 것

파티셔닝은 논리적으로 한 테이블을 물리적으로 여러 조각으로 나눕니다. 핵심 효과는 파티션 프루닝(pruning)입니다. WHERE 조건에 파티션 키가 들어오면 옵티마이저가 관련 없는 파티션을 계획 단계에서 아예 제외합니다.

```sql
CREATE TABLE events (
  id bigint,
  created_at timestamptz NOT NULL,
  payload jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_06 PARTITION OF events
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
CREATE TABLE events_2026_07 PARTITION OF events
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
```

```sql
-- created_at 조건이 있으면 해당 월 파티션 1개만 스캔한다
EXPLAIN SELECT * FROM events
WHERE created_at >= '2026-07-01' AND created_at < '2026-07-02';
```

EXPLAIN에서 스캔 대상이 `events_2026_07` 하나로 줄어든 게 보이면 프루닝이 동작한 겁니다. 전체 스캔 대비 수십 배 빨라지는 이유가 여기 있습니다.

## 같이 알아두면 좋은 것

- 파티션 키가 WHERE에 없으면 프루닝이 안 된다. 쿼리 패턴에 맞는 키(대부분 시간)를 골라야 한다.
- 오래된 데이터 삭제가 `DROP PARTITION` 한 번으로 끝난다. DELETE로 수억 행을 지우는 것과 비교하면 운영 부담 차이가 크다.
- 파티션마다 인덱스가 작아져 인덱스 효율도 함께 좋아진다.

## 정리

- 초대형 테이블의 본질적 해법은 "덜 읽는 구조"를 만드는 것이고, 그게 파티셔닝이다.
- RANGE 파티셔닝 + 파티션 키 조건 = 필요한 조각만 스캔.
- 보관 주기 관리(retention)까지 공짜로 따라온다.

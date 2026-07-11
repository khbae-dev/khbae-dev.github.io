---
title: 어제 8ms였던 쿼리가 오늘 2초가 됐다면, autovacuum 지연
date: 2026-06-08 16:01:00 +09:00
categories: [Database]
tags: [postgresql, autovacuum, bloat, tuning]
description: 대형 테이블에서 autovacuum 기본 임계값이 너무 커서 dead tuple이 쌓이고 쿼리가 느려지는 문제와 scale_factor 튜닝을 정리했다.
---

쿼리도 인덱스도 그대로인데 성능이 며칠 새 뚝 떨어지는 현상을 공부하다가 autovacuum 임계값의 함정을 알게 됐습니다.

## dead tuple이 쌓이는 속도 vs 치우는 속도

PostgreSQL의 UPDATE/DELETE는 기존 행을 바로 지우지 않고 죽은 튜플(dead tuple)로 남깁니다. 이를 autovacuum이 주기적으로 청소하는데, 기본 트리거 조건이 문제입니다.

```
vacuum 실행 조건 ≈ dead tuples > threshold(50) + scale_factor(0.2) × 전체 행 수
```

`scale_factor` 기본값 0.2는 "전체의 20%가 죽어야 청소 시작"이라는 뜻입니다. 1억 행 테이블이라면 2천만 행이 쌓일 때까지 vacuum이 안 돕니다. 그동안 인덱스는 죽은 행을 헛읽고, 테이블은 부풀어 I/O가 늘어납니다. 8ms 쿼리가 2초가 되는 경로입니다.

## 큰 테이블만 따로 조인다

전역 설정을 낮추면 작은 테이블에 과한 vacuum이 돕니다. 테이블 단위로 조정하는 게 정석입니다.

```sql
ALTER TABLE events SET (
  autovacuum_vacuum_scale_factor = 0.01,  -- 1%면 청소 시작
  autovacuum_vacuum_cost_limit = 2000      -- 청소 속도도 상향
);
```

현재 상태 진단은 이 쿼리로 합니다.

```sql
SELECT relname, n_dead_tup, n_live_tup,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 10;
```

## 정리

- 기본 scale_factor 0.2는 대형 테이블에는 사실상 "청소 안 함"과 같다.
- 갱신이 잦은 큰 테이블은 0.01~0.02 수준으로 낮춰 자주, 조금씩 치우게 한다.
- n_dead_tup 모니터링을 대시보드에 두면 성능 저하를 며칠 전에 예見할 수 있다.

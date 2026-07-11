---
title: 컬럼 하나 추가했을 뿐인데 서비스가 멈추는 이유, 락 큐
date: 2026-06-10 16:03:00 +09:00
categories: [Database]
tags: [postgresql, ddl, lock, migration]
description: ALTER TABLE이 요구하는 ACCESS EXCLUSIVE 락이 락 큐를 만들어 테이블 전체를 얼리는 메커니즘과 lock_timeout, CREATE INDEX CONCURRENTLY 대응을 정리했다.
---

"컬럼 추가는 금방 끝나는 작업인데 왜 장애가 나지?"라는 의문을 풀어봤습니다. 답은 작업 시간이 아니라 락 대기 행렬에 있었습니다.

## 락 큐가 만들어지는 과정

`ALTER TABLE`은 ACCESS EXCLUSIVE 락을 요구합니다. 이 락은 SELECT를 포함한 모든 접근과 충돌합니다. 문제의 시나리오는 이렇습니다.

1. 어떤 세션이 긴 트랜잭션으로 테이블에 락(공유 락이라도)을 잡고 있다.
2. ALTER TABLE이 ACCESS EXCLUSIVE를 기다리며 큐에 선다.
3. 이후 도착하는 평범한 SELECT/UPDATE도 전부 ALTER 뒤에 줄을 선다. PostgreSQL 락 큐는 새치기가 없기 때문입니다.
4. 결과: ALTER가 대기하는 동안 테이블이 사실상 동결.

즉 ALTER 자체는 1초짜리여도, 앞의 트랜잭션이 10분짜리면 10분짜리 장애가 됩니다.

## 안전 장치

```sql
-- 락을 3초 안에 못 잡으면 실패시킨다 (큐를 안 만든다)
BEGIN;
SET LOCAL lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN memo text;
COMMIT;
```

실패하면 잠시 후 재시도하면 됩니다. 핵심은 "기다리며 큐를 만드는 것"을 금지하는 겁니다.

인덱스 생성은 아예 락 없이 하는 방법이 있습니다.

```sql
CREATE INDEX CONCURRENTLY idx_orders_user ON orders (user_id);
```

일반 CREATE INDEX와 달리 쓰기를 막지 않습니다. 대신 트랜잭션 블록 밖에서 실행해야 하고 시간이 더 걸립니다.

## 정리

- DDL 장애의 본질은 작업 시간이 아니라 락 대기 큐다.
- 마이그레이션 스크립트에는 lock_timeout을 기본으로 깔아둔다.
- 인덱스는 CONCURRENTLY, 긴 트랜잭션 정리(idle in transaction 감시)도 세트다.

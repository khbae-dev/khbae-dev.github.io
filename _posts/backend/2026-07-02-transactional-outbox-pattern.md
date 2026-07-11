---
title: DB 커밋과 이벤트 발행 사이의 유실, Transactional Outbox로 막기
date: 2026-07-02 19:26:00 +09:00
categories: [Backend]
tags: [kafka, outbox, transaction, event-driven, msa]
description: 주문은 커밋됐는데 이벤트는 발행되지 않는 이중 쓰기 문제의 구조를 이해하고, 같은 트랜잭션에 이벤트를 기록하는 Transactional Outbox 패턴을 정리했다.
---

이벤트 기반 아키텍처의 고전적 함정을 정리했습니다. "주문은 저장됐는데 배송 서비스는 그 사실을 영원히 모르는" 상황이 어떻게 생기는지입니다.

## 이중 쓰기(dual write) 문제

주문 처리 코드가 보통 이렇게 생겼습니다.

```
1. DB 트랜잭션 커밋 (주문 저장)
2. Kafka에 order.created 발행
```

1과 2는 서로 다른 시스템이라 원자적으로 묶이지 않습니다. 1 직후 프로세스가 죽으면 이벤트는 영구 유실됩니다. 순서를 바꾸면 반대로 "이벤트는 나갔는데 주문은 없는" 유령 이벤트가 생깁니다. 어느 쪽이든 재시도 로직으로는 근본 해결이 안 됩니다.

## Transactional Outbox 패턴

핵심은 이벤트 발행을 "DB 쓰기"로 바꾸는 겁니다.

```sql
BEGIN;
INSERT INTO orders (...) VALUES (...);
INSERT INTO outbox (aggregate_id, event_type, payload)
VALUES (:order_id, 'order.created', :json);
COMMIT;  -- 주문과 이벤트가 원자적으로 함께 저장된다
```

별도의 릴레이 프로세스가 outbox 테이블을 폴링(또는 CDC로 WAL 구독)해서 Kafka로 발행하고, 성공한 행을 지웁니다. 커밋됐다면 이벤트도 반드시 존재하고, 릴레이가 죽어도 outbox에 남아 있으니 재기동 후 이어서 발행됩니다. 유실이 구조적으로 사라집니다.

대신 전달 보장이 at-least-once가 되므로, 컨슈머는 이벤트 ID 기반 멱등 처리로 중복을 흡수해야 합니다. Debezium 같은 CDC 도구를 쓰면 폴링 없이 낮은 지연으로 릴레이를 구현할 수 있습니다.

## 정리

- DB와 브로커에 각각 쓰는 코드는 반드시 둘 사이 크래시 시나리오를 물어봐야 한다.
- Outbox는 "발행"을 트랜잭션 안의 "기록"으로 바꿔 원자성을 확보한다.
- at-least-once + 컨슈머 멱등성이 한 세트라는 점까지가 이 패턴의 전부다.

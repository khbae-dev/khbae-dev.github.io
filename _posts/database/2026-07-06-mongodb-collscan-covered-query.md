---
title: MongoDB COLLSCAN 잡기, 복합 인덱스와 covered query
date: 2026-07-06 19:20:00 +09:00
categories: [Database]
tags: [mongodb, index, performance, database]
description: 쿼리·정렬 패턴과 인덱스가 어긋나 전체 스캔(COLLSCAN)이 나가는 문제를 explain으로 확인하고, ESR 순서의 복합 인덱스와 covered query로 해결하는 과정을 정리했다.
---

MongoDB 쿼리가 매번 1초 넘게 걸리는 사례를 공부하면서 `explain()` 읽는 법을 다시 정리했습니다. 결론은 단순했습니다. 쿼리·정렬 패턴에 맞는 인덱스가 없으면 매번 컬렉션 전체를 읽는 COLLSCAN이 나갑니다.

## 진단: explain에서 볼 것

```javascript
db.orders.find({ userId: 42, status: "PAID" })
  .sort({ createdAt: -1 })
  .explain("executionStats")
```

- `stage: "COLLSCAN"` — 인덱스를 아예 못 탄 상태
- `totalDocsExamined`가 반환 문서 수보다 압도적으로 크면 인덱스가 어긋난 상태
- 정렬이 인덱스를 못 타면 `SORT` 스테이지(메모리 정렬)가 별도로 나타남

## 해결: ESR 순서의 복합 인덱스

복합 인덱스는 Equality → Sort → Range 순서로 구성하는 게 원칙입니다.

```javascript
// 동등 조건(userId, status) → 정렬(createdAt) 순
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 })
```

여기서 한 발 더 나가면 covered query가 됩니다. 조회하는 필드가 전부 인덱스에 포함되면 문서 자체를 읽지 않고 인덱스만으로 응답합니다.

```javascript
// projection이 인덱스 필드 안에서 끝나면 document fetch가 사라진다
db.orders.find(
  { userId: 42, status: "PAID" },
  { _id: 0, userId: 1, status: 1, createdAt: 1 }
).sort({ createdAt: -1 })
```

이 패턴으로 1.4초짜리 전체 스캔 쿼리가 한 자릿수 ms까지 내려가는 걸 확인할 수 있습니다. 두 자릿수 배 이상의 차이입니다.

## 정리

- 느린 쿼리는 감으로 잡지 말고 `explain("executionStats")`부터 본다.
- 복합 인덱스는 ESR(동등→정렬→범위) 순서가 기본기다.
- 자주 치는 조회는 covered query로 만들면 문서 접근 자체가 사라진다.

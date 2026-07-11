---
title: 인덱스를 만들었는데도 풀스캔이 뜨는 이유, 복합 인덱스 컬럼 순서
date: 2026-06-06 16:03:00 +09:00
categories: [Database]
tags: [database, index, mysql, postgresql]
description: 복합 인덱스는 왼쪽 선두 컬럼부터만 사용된다는 leftmost prefix 원칙과, 컬럼 순서를 쿼리 패턴에 맞추는 방법을 정리했다.
---

"인덱스 분명히 있는데 왜 안 타지?"의 대표 원인을 정리해 둡니다. 복합 인덱스의 컬럼 순서 문제입니다.

## 전화번호부 비유로 이해하기

`(user_id, created_at)` 복합 인덱스는 전화번호부처럼 user_id로 먼저 정렬되고, 같은 user_id 안에서 created_at으로 정렬된 구조입니다. 성(姓) 없이 이름만으로 전화번호부를 찾을 수 없듯이, 선두 컬럼(user_id) 조건 없이 created_at만으로는 이 인덱스를 쓸 수 없습니다.

```sql
-- 인덱스: (user_id, created_at)

-- 인덱스 사용 O
WHERE user_id = 42
WHERE user_id = 42 AND created_at > '2026-06-01'

-- 인덱스 사용 X (선두 컬럼 부재 → 풀스캔)
WHERE created_at > '2026-06-01'
```

## 해결: 쿼리 패턴에 맞춰 순서 재배치

created_at 단독 조회가 주력 쿼리라면 인덱스를 다시 설계합니다.

```sql
-- created_at 단독 + (user_id, created_at) 조합을 모두 커버하려면
CREATE INDEX idx_orders_created ON orders (created_at);
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at);
```

순서를 정하는 실무 기준은 이렇습니다.

- 단독으로도 자주 조회되는 컬럼을 선두로
- 동등 조건(=) 컬럼을 앞에, 범위 조건(>, <)을 뒤에
- 이미 있는 인덱스로 커버되는 조합은 중복 생성하지 않기

적용 전후를 `EXPLAIN`으로 확인하면 Seq Scan이 Index Scan으로 바뀌며 수백 배 차이가 나는 걸 볼 수 있습니다.

## 정리

- 복합 인덱스는 leftmost prefix, 즉 왼쪽부터 연속으로만 쓰인다.
- 인덱스 설계의 출발점은 테이블이 아니라 실제 쿼리 패턴이다.
- "인덱스가 있다"와 "이 쿼리가 그 인덱스를 탈 수 있다"는 다른 문제다.

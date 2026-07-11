---
title: 뒷페이지로 갈수록 느려지는 페이지네이션, OFFSET 대신 커서
date: 2026-06-04 02:40:00 +09:00
categories: [Database]
tags: [database, pagination, keyset, performance]
description: OFFSET이 앞의 행을 전부 읽고 버리는 구조적 낭비를 이해하고, 커서(키셋) 페이지네이션으로 페이지 깊이와 무관하게 일정한 응답을 만드는 방법을 정리했다.
---

목록 API가 1페이지는 빠른데 1,000페이지쯤 가면 수백 ms씩 걸리는 이유를 정리해 봤습니다.

## OFFSET은 읽고 버린다

```sql
SELECT * FROM posts ORDER BY id DESC LIMIT 20 OFFSET 20000;
```

이 쿼리에서 DB는 20,020행을 인덱스 순서로 읽은 뒤 앞의 20,000행을 버립니다. OFFSET 값만큼의 스캔이 매번 발생하므로 뒤로 갈수록 선형적으로 느려집니다. 스크롤이 깊은 피드, 관리자 페이지, 배치 순회에서 그대로 병목이 됩니다.

## 커서(키셋)는 직행한다

직전 페이지의 마지막 값(커서)을 조건으로 넘기면, 인덱스에서 그 지점으로 바로 점프합니다.

```sql
-- 첫 페이지
SELECT * FROM posts ORDER BY id DESC LIMIT 20;

-- 다음 페이지: 마지막으로 본 id를 커서로
SELECT * FROM posts
WHERE id < :last_seen_id
ORDER BY id DESC LIMIT 20;
```

읽는 양이 항상 20행이라 페이지 깊이와 무관하게 응답이 일정합니다. 정렬 키가 중복될 수 있으면 유니크한 tie-breaker를 붙입니다.

```sql
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC LIMIT 20;
```

트레이드오프도 있습니다. "37페이지로 점프" 같은 임의 페이지 접근이 안 되고, 커서를 클라이언트에 전달하는 API 설계가 필요합니다. 무한 스크롤·더보기 UI와는 정확히 맞아떨어집니다.

## 정리

- OFFSET의 비용은 페이지 번호에 비례한다. 깊은 페이지가 느린 건 구조적 문제다.
- 커서 방식은 인덱스 직행이라 어느 페이지든 일정하게 빠르다.
- 무한 스크롤 API의 기본값은 키셋 페이지네이션으로 잡는 게 맞다.

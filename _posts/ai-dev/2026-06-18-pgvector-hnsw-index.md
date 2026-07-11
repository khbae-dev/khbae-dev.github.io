---
title: 데모는 빠른데 운영에서 벡터 검색이 타임아웃날 때, HNSW 인덱스
date: 2026-06-18 19:20:00 +09:00
categories: [AI Dev]
tags: [pgvector, hnsw, vector-search, rag]
description: 인덱스 없는 벡터 검색이 전수 비교로 느려지는 문제와, pgvector의 HNSW 인덱스·부분 인덱스 푸시다운·ef_search 튜닝을 정리했다.
---

RAG 데모는 즉각 응답하는데 운영 데이터로 올리자 벡터 검색이 타임아웃나는 문제를 공부했습니다. 데이터 양이 임계를 넘으면 드러나는 인덱스 문제였습니다.

## 인덱스 없는 벡터 검색은 전수 비교다

pgvector에서 인덱스 없이 유사도 검색을 하면, 쿼리 벡터를 저장된 모든 벡터와 하나씩 비교합니다(exact search, O(N)). 수천 건 데모에서는 빠르지만, 수백만 건이 되면 매 쿼리가 전수 스캔이라 수 초씩 걸립니다. 정확도는 100%지만 운영 지연을 못 버팁니다.

## HNSW로 근사 최근접 탐색

HNSW(Hierarchical Navigable Small World)는 그래프 기반 근사 최근접 이웃(ANN) 인덱스입니다. 약간의 정확도를 내주는 대신 탐색을 로그 시간 수준으로 줄입니다.

```sql
CREATE INDEX ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

여기에 실무 튜닝을 얹습니다.

```sql
-- 조회 시 정확도/속도 균형. 클수록 정확·느림
SET hnsw.ef_search = 100;

-- 메타 필터가 자주 걸리면 부분 인덱스로 푸시다운
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WHERE lang = 'ko' AND published = true;
```

- `m`, `ef_construction`은 인덱스 빌드 품질 파라미터(크면 정확·느린 빌드).
- `ef_search`는 조회 시 탐색 폭으로, 정확도와 지연의 실시간 다이얼입니다. recall이 부족하면 올리고, 지연이 문제면 내립니다.
- 메타데이터 필터(언어, 공개 여부)가 고정적이면 부분 인덱스로 검색 대상을 미리 좁혀 속도와 정확도를 함께 얻습니다.

이 조합으로 수 초짜리 검색이 수십 ms로 내려갑니다. 근사 검색이라 recall을 실제 데이터로 측정해 임계 파라미터를 정하는 과정이 필요합니다.

## 정리

- 인덱스 없는 벡터 검색은 O(N) 전수 비교라 데이터가 늘면 반드시 느려진다.
- HNSW 인덱스로 근사 최근접 탐색을 쓰면 로그 시간 수준으로 빨라진다.
- ef_search로 정확도·속도를 조율하고, 고정 필터는 부분 인덱스로 푸시다운한다.

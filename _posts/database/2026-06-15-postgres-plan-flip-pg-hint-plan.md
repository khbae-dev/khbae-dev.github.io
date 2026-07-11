---
title: 코드도 데이터도 그대로인데 쿼리가 갑자기 느려졌다면, 플랜 플립
date: 2026-06-15 16:01:00 +09:00
categories: [Database]
tags: [postgresql, query-plan, statistics, auto-explain]
description: autoanalyze가 통계를 갱신하는 순간 실행계획이 뒤집혀 멀쩡하던 쿼리가 수천 배 느려지는 플랜 플립 현상과 감지·고정 방법을 정리했다.
---

배포도 없었고 데이터 분포도 크게 안 변했는데 특정 쿼리만 갑자기 느려지는 사례를 공부하다가 "플랜 플립"이라는 개념을 알게 됐습니다.

## 통계 갱신이 계획을 뒤집는다

PostgreSQL 옵티마이저는 테이블 통계(row 수, 값 분포)를 근거로 실행계획을 고릅니다. autoanalyze가 돌면서 통계가 갱신되는 순간, 비용 추정이 경계값을 넘어가면 잘 돌던 Index Scan이 Nested Loop나 Seq Scan 조합으로 뒤집힐 수 있습니다. 특히 컬럼 간 상관관계가 있는 조건(예: 도시와 우편번호)은 독립 가정 때문에 추정치가 실제와 크게 어긋나기 쉽습니다.

## 대응 3종 세트

**1. 추정 교정 — 확장 통계.** 상관관계 있는 컬럼 묶음에 확장 통계를 만들어 옵티마이저의 추정 자체를 정확하게 만듭니다.

```sql
CREATE STATISTICS stat_city_zip (dependencies)
ON city, zipcode FROM addresses;
ANALYZE addresses;
```

**2. 감지 — auto_explain.** 느려진 쿼리의 "그때 그 계획"을 로그로 남겨야 원인 분석이 됩니다.

```sql
-- postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze = on
```

**3. 고정 — pg_hint_plan.** 재발이 잦고 영향이 큰 핵심 쿼리는 힌트로 계획을 고정하는 선택지도 있습니다. 남용하면 통계 개선 효과를 못 받으므로 최후 수단으로 씁니다.

## 정리

- "아무것도 안 바꿨는데 느려졌다"의 유력 용의자는 통계 갱신에 의한 플랜 플립이다.
- 확장 통계로 추정을 교정하는 게 정공법, auto_explain은 필수 계측.
- 계획 고정(힌트)은 효과가 확실한 핵심 쿼리에만 제한적으로.

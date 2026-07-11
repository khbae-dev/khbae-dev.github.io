---
title: 같은 질문을 표현만 바꿔 물어도 매번 과금될 때, 시맨틱 캐시
date: 2026-06-20 19:20:00 +09:00
categories: [AI Dev]
tags: [llm, cache, embedding, cost]
description: 문자열이 조금만 달라도 캐시 미스가 나는 문제를, 질문을 임베딩해 의미 유사도로 캐시를 조회하는 시맨틱 캐시로 해결하는 방법을 정리했다.
---

LLM 비용을 줄이려고 캐시를 붙였는데 효과가 미미한 이유를 공부했습니다. 전통적 캐시가 LLM에는 잘 안 맞았습니다.

## 문자열 캐시의 한계

일반 캐시는 키가 정확히 같아야 히트합니다. 그런데 사용자들은 같은 의도를 제각각 다르게 표현합니다.

```
"환불 어떻게 해요?"
"환불하는 방법 알려주세요"
"제품 반품하고 돈 돌려받고 싶어요"
```

세 질문은 의미가 같지만 문자열은 전부 달라, 문자열 캐시로는 전부 미스입니다. 결국 매번 LLM을 호출해 과금됩니다. 자연어의 다양성 때문에 정확 일치 캐시의 히트율은 낮을 수밖에 없습니다.

## 의미로 캐시를 찾는다

시맨틱 캐시는 질문을 임베딩(벡터)으로 바꿔, 과거 질문 벡터들과의 유사도로 캐시를 조회합니다. 충분히 유사하면(임계값 이상) 저장된 답을 재사용합니다.

```python
def semantic_cache_get(question, threshold=0.92):
    q_vec = embed(question)
    hit = vector_store.search(q_vec, top_k=1)   # 가장 가까운 과거 질문
    if hit and hit.score >= threshold:
        return hit.cached_answer                # 캐시 히트: LLM 호출 없음
    return None

def answer(question):
    cached = semantic_cache_get(question)
    if cached:
        return cached
    ans = call_llm(question)
    vector_store.add(embed(question), question, ans)   # 캐시에 저장
    return ans
```

임계값 설정이 핵심 트레이드오프입니다. 너무 낮으면 다른 의미의 질문에 엉뚱한 캐시 답을 주고(위험), 너무 높으면 히트율이 떨어집니다. 0.9 초반대에서 시작해 도메인에 맞게 튜닝합니다. 개인화·시의성이 중요한 답변(계정 상태, 실시간 데이터)은 캐시 대상에서 제외하고, 일반 지식·FAQ성 질문에 적용하는 게 안전합니다.

## 정리

- LLM 입력은 자연어라 정확 일치 캐시의 히트율이 구조적으로 낮다.
- 질문을 임베딩해 의미 유사도로 조회하는 시맨틱 캐시가 히트율을 끌어올린다.
- 유사도 임계값 튜닝과 "캐시하면 안 되는 질문" 구분이 안전한 운영의 핵심이다.

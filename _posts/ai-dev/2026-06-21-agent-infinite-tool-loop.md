---
title: AI 에이전트가 같은 툴을 수백 번 호출하며 토큰을 태울 때
date: 2026-06-21 19:24:00 +09:00
categories: [AI Dev]
tags: [llm, agent, tool-loop, cost]
description: 종료 조건 없는 툴 호출 루프에 빠진 에이전트의 토큰 폭발을, 스텝 상한·동일 호출 캐시·루프 감지로 끊는 방법을 정리했다.
---

자율 에이전트가 같은 도구를 200번 넘게 호출하며 토큰만 소모한 사례를 공부했습니다. 무한 루프의 에이전트 버전입니다.

## 왜 멈추지 못하나

에이전트는 "생각 → 도구 호출 → 관찰 → 다시 생각"을 반복합니다. 문제는 종료 조건이 불명확할 때입니다. 도구가 매번 애매한 결과(빈 값, 에러 메시지, 같은 데이터)를 돌려주면, 모델은 "조금 다르게 다시 해보자"며 비슷한 호출을 반복합니다. 사람이라면 "이건 안 되는구나" 하고 포기할 상황에서, 에이전트는 관성적으로 루프를 돕니다. 목표 달성도 실패 선언도 못 한 채 토큰만 태웁니다.

## 세 겹으로 끊는다

```python
class AgentGuard:
    def __init__(self, max_steps=25):
        self.steps = 0
        self.max_steps = max_steps
        self.seen = {}          # 동일 호출 캐시
        self.recent = []        # 루프 감지용

    def check(self, tool, args):
        self.steps += 1
        # 1. 스텝 상한: 무조건 멈추는 안전판
        if self.steps > self.max_steps:
            raise StopAgent("step limit exceeded")

        key = (tool, canonical(args))
        # 2. 동일 호출 캐시: 같은 호출은 재실행 없이 캐시 반환
        if key in self.seen:
            return self.seen[key]

        # 3. 루프 감지: 최근 N개 중 같은 호출 반복이면 개입
        self.recent.append(key)
        if self.recent[-6:].count(key) >= 3:
            raise StopAgent("loop detected: " + tool)
        return None
```

- 스텝 상한: 어떤 경우에도 정해진 횟수 넘으면 종료합니다. 가장 단순하고 확실한 안전판입니다.
- 동일 호출 캐시: 같은 도구를 같은 인자로 부르면 결과를 재사용해, 무의미한 반복 실행과 과금을 막습니다.
- 루프 감지: 최근 호출 이력에서 같은 패턴이 반복되면 조기 개입합니다. 모델에게 "이 접근은 실패했으니 다른 방법을 쓰거나 포기하라"는 신호를 주거나 사람에게 에스컬레이션합니다.

여기에 예산(토큰·비용) 상한을 두어 어떤 실패 모드에서도 지출 천장을 고정하는 게 실무적으로 중요합니다.

## 정리

- 종료 조건이 불명확한 에이전트는 무한 툴 루프에 빠져 토큰을 태운다.
- 스텝 상한(안전판) + 동일 호출 캐시(중복 제거) + 루프 감지(조기 개입)로 끊는다.
- 토큰·비용 예산 상한을 둬서 최악의 경우에도 지출을 고정한다.

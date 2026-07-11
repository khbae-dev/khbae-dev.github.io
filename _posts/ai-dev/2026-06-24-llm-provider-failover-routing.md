---
title: LLM provider 한 곳에 의존하면 그곳이 멈출 때 전 기능이 멈춘다
date: 2026-06-24 19:24:00 +09:00
categories: [AI Dev]
tags: [llm, failover, circuit-breaker, resilience]
description: 단일 LLM provider 의존(SPOF)의 위험과, 멀티 provider 라우팅·서킷브레이커·fallback 체인으로 가용성을 확보하는 방법을 정리했다.
---

LLM 기능을 제품에 넣을 때 놓치기 쉬운 가용성 문제를 공부했습니다. 모델 provider가 곧 단일 장애점(SPOF)이 되는 구조입니다.

## 남의 인프라에 우리 가용성을 맡긴 상태

LLM API 하나에만 연결해 두면, 그 provider가 장애나 rate limit에 걸리는 순간 우리 제품의 AI 기능 전체가 멈춥니다. 우리가 통제할 수 없는 외부 인프라에 서비스 가용성을 전적으로 위임한 셈입니다. 대형 provider도 종종 장애를 겪고, 특정 모델의 폐기·정책 변경도 갑자기 옵니다.

## 멀티 provider + 서킷브레이커 + fallback

핵심은 provider를 교체 가능한 부품으로 추상화하고, 죽은 곳을 격리해 다른 곳으로 넘기는 겁니다.

```javascript
const providers = [
  { name: "primary",  call: callProviderA, breaker: new CircuitBreaker() },
  { name: "secondary",call: callProviderB, breaker: new CircuitBreaker() },
];

async function complete(prompt) {
  for (const p of providers) {
    if (p.breaker.isOpen()) continue;      // 최근 실패 많으면 건너뜀
    try {
      return await p.call(prompt);
    } catch (e) {
      p.breaker.recordFailure();           // 실패 누적 → 회로 개방
      continue;                            // 다음 provider로 fallback
    }
  }
  throw new Error("all providers unavailable");
}
```

- 멀티 provider 라우팅: 같은 요청을 여러 provider로 보낼 수 있게 인터페이스를 통일합니다(프롬프트·응답 형식 차이는 어댑터로 흡수).
- 서킷브레이커: 특정 provider의 실패가 임계를 넘으면 회로를 열어 한동안 그곳으로 안 보냅니다. 죽은 곳에 계속 요청하며 지연을 쌓는 걸 막습니다.
- fallback 체인: 우선순위대로 시도하고, 실패 시 다음으로 넘깁니다.

주의할 점은 provider마다 품질·비용·형식이 다르다는 겁니다. fallback 시 품질이 떨어질 수 있으니 이를 로깅하고, 프롬프트를 provider 중립적으로 설계하는 노력이 필요합니다.

## 정리

- 단일 LLM provider 의존은 우리 제품에 남의 SPOF를 이식하는 것이다.
- provider를 교체 가능한 부품으로 추상화하고 fallback 체인을 둔다.
- 서킷브레이커로 죽은 provider를 격리해야 fallback이 지연 없이 동작한다.

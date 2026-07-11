---
title: 재시도 한 번에 결제가 두 번 되는 문제, Idempotency-Key
date: 2026-06-04 02:37:00 +09:00
categories: [Backend]
tags: [api-design, idempotency, payment, reliability]
description: 네트워크 재시도가 중복 결제로 이어지는 구조와, 멱등 키로 같은 요청에 같은 결과를 보장하는 API 설계를 정리했다.
---

결제 API 설계에서 반드시 짚어야 하는 문제를 정리했습니다. 네트워크의 근본적인 불확실성에서 출발합니다.

## "응답을 못 받음"은 "처리 안 됨"이 아니다

클라이언트가 결제 요청을 보냈는데 타임아웃이 났다고 합시다. 이때 가능한 상태는 두 가지입니다. 요청이 서버에 도달하지 못했거나, 처리는 됐는데 응답만 유실됐거나. 클라이언트는 이 둘을 구분할 수 없습니다. 안전하게 재시도하자니 이미 처리된 경우 중복 결제가 되고, 재시도를 안 하자니 미처리된 경우 결제가 누락됩니다.

## 멱등 키: 요청에 고유한 이름 붙이기

해결책은 "같은 요청인지"를 서버가 식별할 수 있게 만드는 겁니다. 클라이언트가 작업 단위마다 고유 키를 생성해 보냅니다.

```
POST /payments
Idempotency-Key: 3f1c2a9e-...-b7d4   (작업 생성 시 1회 발급한 UUID)
```

서버는 키 단위로 이렇게 처리합니다.

```javascript
async function handlePayment(req) {
  const key = req.headers["idempotency-key"];
  const existing = await store.get(key);
  if (existing) return existing.response;   // 재요청 → 저장된 결과 그대로

  const lock = await store.lockNX(key);     // 동시 중복 요청 방어
  if (!lock) return retryLaterResponse();

  const result = await processPayment(req.body);
  await store.save(key, result, TTL_24H);   // 결과 저장
  return result;
}
```

핵심 규칙은 세 가지입니다. 키는 "재시도 묶음"당 하나(재시도 시 같은 키 재사용), 결과 저장과 처리는 원자적으로, 키에는 TTL을 둬서 저장소가 무한히 크지 않게. 이렇게 하면 재요청이 100번 와도 결제는 정확히 1건입니다.

## 정리

- 타임아웃은 "미처리"가 아니라 "결과 불명"이다. 이 구분이 설계의 출발점.
- 부작용 있는 API(결제, 주문, 발송)에는 Idempotency-Key를 기본 스펙으로 넣는다.
- 같은 키 = 같은 응답. 이 보장 하나가 클라이언트의 재시도를 안전하게 만든다.

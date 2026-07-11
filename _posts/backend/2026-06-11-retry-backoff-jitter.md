---
title: 429 받고 다 같이 1초 뒤 재시도하면 또 429, jitter가 필요한 이유
date: 2026-06-11 16:04:00 +09:00
categories: [Backend]
tags: [retry, backoff, jitter, rate-limit]
description: 고정 간격 재시도가 실패를 동기화시켜 폭주를 반복하는 문제와 Retry-After 존중, 지수 백오프, jitter 조합을 정리했다.
---

재시도 로직이 오히려 장애를 키우는 패턴을 정리했습니다. 핵심은 "실패가 클라이언트들을 동기화시킨다"는 관찰입니다.

## 재시도 동기화 문제

API가 과부하로 429를 반환하면, 수천 클라이언트가 같은 순간에 실패를 겪습니다. 모두가 "1초 뒤 재시도" 로직을 갖고 있다면, 정확히 1초 뒤 수천 요청이 다시 동시에 도착합니다. 서버는 또 429를 뱉고, 이 파도는 계속 반복됩니다. 고정 간격 재시도는 부하를 시간축 위의 뾰족한 스파이크들로 만들 뿐입니다.

## 세 가지를 조합한다

```javascript
async function requestWithRetry(fn, maxAttempts = 5) {
  for (let attempt = 0; ; attempt++) {
    const res = await fn();
    if (res.status !== 429 && res.status < 500) return res;
    if (attempt >= maxAttempts - 1) throw new Error("retry exhausted");

    // 1. 서버가 알려준 시점을 최우선으로 존중
    const retryAfter = Number(res.headers.get("retry-after")) * 1000;

    // 2. 지수 백오프 + 3. full jitter
    const base = Math.min(1000 * 2 ** attempt, 30000);
    const delay = retryAfter || Math.random() * base;

    await new Promise(r => setTimeout(r, delay));
  }
}
```

- **Retry-After 존중**: 서버가 명시한 대기 시간이 있으면 그게 정답입니다.
- **지수 백오프**: 연속 실패 시 간격을 1s→2s→4s로 늘려 서버 회복 시간을 법니다.
- **jitter**: 간격에 무작위성을 섞어 클라이언트들을 시간축에 흩뿌립니다. 동기화 파도를 부수는 게 jitter의 역할이라, 셋 중 하나만 골라야 한다면 jitter입니다.

재시도 총량 자체를 제한하는 것(멱등 요청만, 시도 횟수 상한, 재시도 예산)도 함께 챙깁니다.

## 정리

- 고정 간격 재시도는 실패를 동기화시켜 스파이크를 재생산한다.
- Retry-After > 지수 백오프 + full jitter 순서로 대기 시간을 정한다.
- 재시도는 서버를 돕는 방식이어야지, 공격이 되면 안 된다.

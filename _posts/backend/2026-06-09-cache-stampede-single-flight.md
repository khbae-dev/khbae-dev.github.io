---
title: 캐시 만료 0.1초에 DB가 눕는 cache stampede, single-flight로 막기
date: 2026-06-09 16:01:00 +09:00
categories: [Backend]
tags: [cache, stampede, redis, single-flight]
description: 인기 키의 TTL이 끝나는 순간 수천 요청이 동시에 캐시 미스를 내며 DB를 때리는 문제와 single-flight, TTL jitter 대응을 정리했다.
---

트래픽이 평소와 같은데 특정 시각마다 DB가 휘청이는 문제의 원인으로 cache stampede(thundering herd)를 공부했습니다.

## 만료의 순간이 위험하다

초당 수천 번 조회되는 인기 키가 있다고 하면, TTL이 살아있는 동안 DB에는 요청이 가지 않습니다. 문제는 만료되는 바로 그 순간입니다. 수천 개의 동시 요청이 일제히 캐시 미스를 맞고, 모두가 같은 무거운 쿼리를 DB에 던집니다. DB가 밀리면 재계산도 늦어지고, 그동안 미스는 계속 쌓입니다. 캐시가 있는데도 캐시가 없는 것보다 더 뾰족한 부하가 생기는 겁니다.

## single-flight: 재계산은 한 명만

같은 키의 동시 미스를 하나의 재계산으로 합칩니다. 첫 요청만 DB에 가고, 나머지는 그 결과를 기다렸다 나눠 받습니다.

```javascript
const inflight = new Map();

async function getWithSingleFlight(key, loader, ttl) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  if (inflight.has(key)) return inflight.get(key); // 진행 중이면 합류

  const p = (async () => {
    try {
      const val = await loader();
      await redis.set(key, JSON.stringify(val), "EX", ttl);
      return val;
    } finally {
      inflight.delete(key);
    }
  })();
  inflight.set(key, p);
  return p;
}
```

멀티 인스턴스 환경이라면 Redis `SET NX`로 분산 락을 잡은 인스턴스만 재계산하고 나머지는 짧게 대기 후 캐시를 다시 읽게 합니다.

보조 수단으로 TTL jitter도 씁니다. TTL에 ±10% 무작위를 섞으면 여러 키가 같은 시각에 일제히 만료되는 것도 막을 수 있습니다. 더 나아가면 만료 전에 백그라운드로 미리 갱신하는 soft TTL 방식도 있습니다.

## 정리

- 캐시의 위험 지점은 히트율이 아니라 만료의 순간이다.
- single-flight로 "동시 미스 = 재계산 1건"을 보장하는 게 핵심.
- TTL jitter와 사전 갱신은 만료 자체를 분산·제거하는 보완재다.

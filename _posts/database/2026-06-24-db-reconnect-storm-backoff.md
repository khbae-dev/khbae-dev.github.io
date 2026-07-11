---
title: 재시작한 DB가 켜자마자 다시 죽는 이유, 커넥션 스톰
date: 2026-06-24 19:21:00 +09:00
categories: [Database]
tags: [database, connection-pool, backoff, resilience]
description: 장애 복구 직후 모든 인스턴스가 동시에 커넥션 풀을 채우려다 DB를 다시 쓰러뜨리는 thundering herd 문제와 지수 백오프+jitter 대응을 정리했다.
---

장애 복구 시나리오를 공부하다가 흥미로운 패턴을 봤습니다. 죽었던 DB를 재시작하면 켜지자마자 다시 죽는 현상입니다. 원인은 재연결 폭주, 이른바 커넥션 스톰이었습니다.

## 왜 복구 직후가 가장 위험한가

DB가 내려간 동안 모든 애플리케이션 인스턴스의 커넥션 풀은 비어 있습니다. DB가 살아나는 순간, 수십 개 인스턴스가 동시에 풀 최대치만큼 커넥션을 새로 맺으려 달려듭니다(thundering herd). 커넥션 수립은 인증·메모리 할당이 필요한 비싼 작업이라, 갓 부팅해서 캐시도 비어 있는 DB에는 최악의 부하입니다. 결과적으로 복구 → 폭주 → 재다운이 반복됩니다.

## 해결 조합

**1. 지수 백오프 + jitter.** 재연결 시도 간격을 지수적으로 늘리되, 무작위 지연을 섞어 인스턴스들이 서로 다른 시각에 시도하게 만듭니다.

```javascript
async function reconnect(attempt = 0) {
  const base = Math.min(1000 * 2 ** attempt, 30000);
  const jitter = Math.random() * base; // full jitter
  await sleep(jitter);
  try { return await pool.connect(); }
  catch { return reconnect(attempt + 1); }
}
```

**2. 풀 상한과 점진 워밍업.** 복구 직후부터 풀 최대치를 채우지 말고, 최소 커넥션부터 시작해 헬스체크가 안정된 뒤 점진적으로 늘립니다.

**3. 헬스체크 기반 재개.** LB나 애플리케이션이 DB 헬스체크 통과를 확인한 뒤에야 트래픽을 단계적으로 돌립니다.

## 정리

- 복구 직후의 DB는 평소보다 약하다. 전 인스턴스 동시 재연결은 재장애 레시피다.
- jitter 없는 백오프는 폭주 시각만 뒤로 미룰 뿐이다. 무작위성이 핵심.
- 풀 워밍업과 점진 트래픽 재개까지 갖추면 복구가 한 번에 끝난다.

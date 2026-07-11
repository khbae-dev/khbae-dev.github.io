---
title: 헬스체크는 초록불인데 전부 타임아웃, bulkhead로 자원 격리하기
date: 2026-06-12 16:03:00 +09:00
categories: [Backend]
tags: [resilience, bulkhead, circuit-breaker, thread-pool]
description: 느린 의존성 하나가 공용 스레드풀을 독점해 무관한 요청까지 마비시키는 문제와, bulkhead 격리·서킷브레이커 조합을 정리했다.
---

모니터링은 전부 정상인데 사용자만 타임아웃을 겪는 미스터리한 장애 패턴을 공부했습니다. 헬스체크는 가벼워서 통과하지만, 실제 요청은 자원이 없어 대기하는 상황입니다.

## 공용 풀의 독점

전형적인 시나리오는 이렇습니다. 서버의 워커 스레드풀이 200개인데, 결제 PG사 API가 느려집니다. 결제 요청들이 응답을 기다리며 스레드를 하나씩 점유하고, 몇 분 안에 200개 전부가 결제 대기에 묶입니다. 이제 결제와 무관한 상품 조회, 로그인까지 처리할 스레드가 없습니다. 장애 반경이 "결제 기능"에서 "서비스 전체"로 번진 겁니다.

## bulkhead: 침수 구획 나누기

배의 격벽(bulkhead)처럼 의존성별로 자원 풀을 분리합니다.

```java
// Resilience4j 예시
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(20)      // 결제 호출은 동시 20개까지만
    .maxWaitDuration(Duration.ofMillis(100))
    .build();
Bulkhead paymentBulkhead = Bulkhead.of("payment", config);

Supplier<PaymentResult> guarded =
    Bulkhead.decorateSupplier(paymentBulkhead, () -> payClient.charge(req));
```

결제가 아무리 느려져도 점유 상한이 20이라 나머지 180 스레드는 다른 기능을 계속 처리합니다. 여기에 서킷브레이커를 겹치면, 실패율이 임계를 넘는 순간 결제 호출을 즉시 실패(fast-fail)시켜 대기 자체를 없앱니다. 장애가 결제 기능 안에 갇히고, 사용자에게는 "결제 일시 불가" 같은 부분 실패로 격하됩니다.

## 정리

- 공용 풀은 장애 전파의 고속도로다. 외부 의존성마다 전용 풀/상한을 둔다.
- bulkhead가 반경을 좁히고, 서킷브레이커가 대기 시간을 없앤다. 둘은 세트다.
- 헬스체크가 정상인데 타임아웃이 나면 "자원 독점"부터 의심한다.

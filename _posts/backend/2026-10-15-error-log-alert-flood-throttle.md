---
title: 같은 오류로 알림 메일이 수백 통 오는 문제, 지문별 쿨다운
date: 2026-10-15 19:22:00 +09:00
categories: [Backend]
tags: [spring, logging, alerting, exception-handling, java]
description: 반복되는 서버 오류마다 ERROR 로그와 알림 메일이 나가 운영 알림이 폭주하던 문제를, 오류 지문별 쿨다운과 억제 건수 집계로 줄인 방법을 정리했다.
---

ERROR 로그가 남으면 운영 메일이 나가도록 설정된 서비스가 있었습니다. 외부 DB가 잠깐 흔들리거나 특정 요청이 반복해서 실패하면, 같은 오류로 메일이 몇 분 만에 수백 통 쌓였습니다. 정작 새로운 오류는 그 사이에 묻혔습니다.

## 목표

- 같은 오류는 일정 시간에 한 번만 ERROR로 남긴다.
- 그 사이 몇 번 더 났는지는 잃지 않는다.
- HTTP 응답은 전혀 바뀌지 않는다. 로그 정책만 바꾼다.

## 오류 지문

"같은 오류"를 정해야 합니다. 메시지 전체를 쓰면 ID나 시각이 섞여 매번 다른 오류가 되므로, 예외 타입과 발생 위치, 요청 경로 정도로 지문을 만들었습니다.

```java
String fingerprint = ex.getClass().getName() + "|" + topFrame(ex) + "|" + request.getRequestURI();
```

## 쿨다운 구현

```java
public class ErrorLogThrottle {
    private final long cooldownMillis;
    private final Map<String, Window> windows;

    public ErrorLogThrottle(long cooldownMillis, int maxFingerprints) {
        this.cooldownMillis = cooldownMillis;
        // 접근 순서 LinkedHashMap: 지문이 너무 많아지면 가장 오래 안 쓴 것부터 버린다
        this.windows = new LinkedHashMap<>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, Window> eldest) {
                return size() > maxFingerprints;
            }
        };
    }

    public synchronized Decision evaluate(String fingerprint, long now) {
        Window w = windows.get(fingerprint);
        if (w == null) {
            windows.put(fingerprint, new Window(now));
            return Decision.allow(0);
        }
        if (now - w.lastLoggedAt >= cooldownMillis) {
            long suppressed = w.suppressed;
            windows.put(fingerprint, new Window(now));
            return Decision.allow(suppressed);       // 다시 남기면서 그동안 억제한 건수를 함께 알린다
        }
        w.suppressed++;
        return Decision.suppress(w.suppressed);
    }
}
```

몇 가지 선택의 이유는 이렇습니다.

- **지문 수에 상한을 둔다.** 지문에 요청 경로가 들어가므로, 경로가 무한히 다양하면 맵이 끝없이 커집니다. 접근 순서 `LinkedHashMap`의 `removeEldestEntry`로 간단한 LRU를 만들었습니다.
- **억제된 건 DEBUG나 WARN 한 줄로.** ERROR가 아니면 메일은 나가지 않지만, 로그 검색으로는 여전히 찾을 수 있습니다.
- **다시 허용될 때 억제 건수를 붙인다.** "10분 동안 342건 더 발생"이 붙어 있으면 장애 규모를 한눈에 압니다.
- **시계를 주입받는다.** 생성자에 `LongSupplier`를 받으면 테스트에서 시간을 마음대로 돌릴 수 있습니다.

## 예외 처리기에서 쓰기

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorBody> handle(Exception ex, HttpServletRequest req) {
    Decision d = throttle.evaluate(fingerprint(ex, req), clock.millis());
    if (d.allowed()) {
        log.error("Unhandled error (suppressed since last: {})", d.suppressedCount(), ex);
    } else {
        log.debug("Suppressed repeated error #{}: {}", d.suppressedCount(), ex.toString());
    }
    return ResponseEntity.status(500).body(ErrorBody.generic());
}
```

## 한계

상태는 인스턴스 메모리에 있으므로 서버가 3대면 같은 오류가 최대 3번 나갑니다. 이 정도면 충분해서 공유 저장소는 두지 않았습니다. 더 줄이려면 알림 쪽(예: 알림 시스템의 그룹핑·억제 기능)에서 한 번 더 묶는 편이 맞습니다.

## 정리

- 반복 오류 알림은 "새 오류를 묻어 버린다"는 점에서 알림이 없는 것보다 나쁠 수 있다.
- 오류 지문별 쿨다운으로 ERROR는 주기당 한 번만, 억제 건수는 다음 ERROR에 붙여 남긴다.
- 지문 맵에는 상한을 두고, 시계는 주입받아 테스트한다.

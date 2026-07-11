---
title: 뒤로가기가 매번 풀 로딩되는 이유, bfcache를 깨뜨리는 코드
date: 2026-06-29 19:21:00 +09:00
categories: [Frontend]
tags: [bfcache, performance, navigation, web]
description: unload 핸들러와 Cache-Control no-store가 bfcache를 무효화해 뒤로가기 즉시 복원을 막는 문제와 pagehide 전환을 정리했다.
---

같은 사이트인데 어떤 페이지는 뒤로가기가 즉시 복원되고 어떤 페이지는 매번 새로 로딩되는 이유를 공부했습니다. 브라우저의 bfcache(back/forward cache)를 깨뜨리는 코드가 원인이었습니다.

## bfcache는 페이지를 통째로 얼려둔다

브라우저는 페이지를 떠날 때 JS 힙과 DOM 상태를 통째로 메모리에 보존했다가, 뒤로가기 시 그 스냅샷을 그대로 되살립니다. 네트워크 요청도, 스크립트 재실행도 없어서 복원이 사실상 0ms이고 스크롤 위치·입력값까지 유지됩니다. 문제는 특정 조건에서 브라우저가 이 캐시를 포기한다는 점입니다. 대표 원인이 두 가지입니다.

- `unload` 이벤트 핸들러: bfcache와 의미가 충돌해 등록만으로 캐시 부적격이 된다(특히 구형 동작 기준).
- `Cache-Control: no-store`: 민감하지 않은 페이지에까지 습관적으로 붙이면 bfcache도 함께 꺼진다.

그 밖에 열린 IndexedDB 트랜잭션, 진행 중인 fetch를 정리하지 않는 것도 사유가 됩니다. 진단은 DevTools → Application → Back/forward cache에서 부적격 사유를 직접 확인할 수 있습니다.

## 고치는 법

```javascript
// 나쁨: unload는 bfcache를 죽인다
window.addEventListener("unload", saveAnalytics);

// 좋음: pagehide로 교체 (bfcache 진입 시에도 호출됨)
window.addEventListener("pagehide", (e) => {
  navigator.sendBeacon("/analytics", payload);
  // e.persisted === true면 bfcache로 들어간다는 뜻
});
```

복원 시점 처리가 필요하면 `pageshow`에서 `e.persisted`를 확인합니다. 그리고 no-store는 정말 민감한 페이지(계좌 정보 등)에만 남기고 걷어냅니다.

## 정리

- 뒤로가기 성능은 최적화가 아니라 "브라우저를 방해하지 않기"의 문제다.
- unload는 pagehide로, 습관적 no-store는 제거. 이 둘이 대부분을 해결한다.
- DevTools의 bfcache 패널로 부적격 사유를 확인하는 것부터 시작한다.

---
title: 클릭 후 0.2초 멍해지는 버튼, 긴 핸들러 쪼개서 양보하기
date: 2026-06-15 16:02:00 +09:00
categories: [Frontend]
tags: [inp, scheduler, react, performance]
description: 클릭 핸들러 안의 무거운 동기 작업이 다음 페인트를 막아 INP를 망치는 문제를 작업 분할과 scheduler.yield, startTransition으로 개선하는 방법을 정리했다.
---

버튼을 눌렀는데 시각적 반응이 한 박자 늦는 문제를 INP 관점에서 공부했습니다. 핸들러가 "다 끝내고 나서" 화면을 그리게 두는 게 원인이었습니다.

## INP는 다음 페인트까지의 시간이다

INP(Interaction to Next Paint)는 입력 처리 완료가 아니라 "입력 후 다음 화면 갱신"까지를 잽니다. 클릭 핸들러가 200ms짜리 동기 작업을 돌리면, 그 200ms 동안 브라우저는 페인트를 못 하므로 눌림 표시조차 늦게 나타납니다. 사용자에게는 "버튼이 씹혔다"로 느껴집니다.

## 작업을 쪼개고 중간에 양보한다

핵심 아이디어는 무거운 작업을 청크로 나누고, 청크 사이에 메인 스레드를 브라우저에 양보해 페인트할 틈을 주는 겁니다.

```javascript
async function handleClick() {
  setPressed(true);              // 1. 즉각적인 시각 피드백 먼저

  for (const chunk of chunks(bigWork, 50)) {
    processChunk(chunk);
    // 2. 청크 사이마다 양보 → 브라우저가 페인트·입력 처리
    if ("scheduler" in window && scheduler.yield) {
      await scheduler.yield();   // 양보 후 우선 복귀 보장
    } else {
      await new Promise(r => setTimeout(r, 0));
    }
  }
}
```

`scheduler.yield()`는 setTimeout(0)과 달리 양보 후 이어서 실행될 때 큐의 앞쪽 우선순위를 보장받는, 이 용도로 설계된 API입니다.

React라면 상태 갱신 단위로도 나눌 수 있습니다. 급한 갱신(눌림 표시)과 급하지 않은 갱신(결과 목록 재계산)을 분리합니다.

```jsx
setPressed(true);                        // 긴급: 즉시
startTransition(() => setResults(compute())); // 비긴급: 중단 가능
```

## 정리

- INP 개선의 첫 수는 "시각 피드백을 가장 먼저, 무거운 일은 그 뒤에"다.
- 50ms 넘는 동기 블록은 쪼개고 scheduler.yield로 페인트 틈을 준다.
- React에서는 startTransition으로 갱신의 긴급도를 나누는 게 같은 원리다.

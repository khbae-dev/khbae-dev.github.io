---
title: 스크롤이 뚝뚝 끊기는 jank, Web Worker로 메인 스레드 비우기
date: 2026-07-01 19:21:00 +09:00
categories: [Frontend]
tags: [web-worker, performance, long-task, javascript]
description: 대용량 JSON 파싱·정렬이 메인 스레드에서 Long Task를 만들어 프레임을 떨어뜨리는 문제와 Web Worker·Transferable 오프로드를 정리했다.
---

화면 스크롤이 순간순간 얼어붙는 jank 현상의 원인을 공부했습니다. Performance 탭에서 빨간 딱지가 붙는 Long Task가 범인이었습니다.

## 16ms의 예산

60fps를 유지하려면 한 프레임의 예산은 약 16.7ms입니다. 메인 스레드는 이 안에서 JS 실행, 스타일 계산, 레이아웃, 페인트를 전부 처리해야 합니다. 대용량 JSON 파싱이나 수만 건 정렬 같은 작업이 200ms를 점유하면 그동안 프레임 여러 장이 통째로 사라지고, 스크롤·클릭 이벤트 처리도 밀립니다. 50ms를 넘는 작업이 Long Task로 분류되는 이유입니다.

## Worker로 옮기고, 복사 대신 이전

무거운 연산을 Web Worker로 보내면 메인 스레드는 렌더링에 전념할 수 있습니다.

```javascript
// main.js
const worker = new Worker("./parse-worker.js");
worker.postMessage(rawBuffer, [rawBuffer]); // Transferable: 복사 없이 소유권 이전
worker.onmessage = (e) => render(e.data);
```

```javascript
// parse-worker.js
self.onmessage = (e) => {
  const text = new TextDecoder().decode(e.data);
  const rows = JSON.parse(text);
  rows.sort((a, b) => b.score - a.score);
  self.postMessage(rows);
};
```

두 번째 인자로 넘기는 Transferable이 포인트입니다. postMessage는 기본적으로 구조화 복제(복사)를 하는데, ArrayBuffer 같은 객체는 소유권만 이전(zero-copy)할 수 있어 수십 MB 데이터도 전달 비용이 사실상 0입니다. 큰 데이터를 주고받는 오프로드에서는 이게 없으면 복사 비용이 새로운 병목이 됩니다.

worker 생성 비용이 있으니 매번 만들지 말고 재사용하고, comlink 같은 래퍼를 쓰면 RPC처럼 다룰 수 있어 코드도 깔끔해집니다.

## 정리

- jank 디버깅은 Performance 탭의 Long Task(>50ms) 찾기에서 시작한다.
- 파싱·정렬·집계처럼 DOM이 필요 없는 연산은 전부 Worker 후보다.
- 큰 데이터는 Transferable로 zero-copy 전달까지 해야 오프로드가 완성된다.

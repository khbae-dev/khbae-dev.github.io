---
title: CPU는 노는데 모든 요청이 멈추는 Node 이벤트 루프 블로킹
date: 2026-06-11 16:01:00 +09:00
categories: [Backend]
tags: [nodejs, event-loop, worker-threads, performance]
description: 단일 스레드에서 무거운 동기 연산이 이벤트 루프를 점유하면 전체 요청이 대기하는 구조와 worker_threads 오프로드를 정리했다.
---

서버 CPU 사용률은 한 코어만 100%이고 나머지는 놀고 있는데, 모든 요청이 응답하지 않는 상황. Node의 실행 모델을 이해하면 자연스러운 결과라는 걸 정리했습니다.

## 단일 스레드의 의미

Node에서 JavaScript 코드는 하나의 스레드(이벤트 루프)에서 실행됩니다. I/O는 비동기로 잘 흩어지지만, JS 코드 자체가 오래 돌면 얘기가 다릅니다. 대용량 JSON.parse, 동기 암호화, 복잡한 정렬, 거대한 루프 같은 CPU 작업이 300ms 돌면, 그 300ms 동안 다른 요청의 콜백은 하나도 실행되지 못합니다. 코어가 32개여도 소용없습니다. JS는 그 한 스레드에서만 돌기 때문입니다.

이벤트 루프 지연은 이렇게 측정합니다.

```javascript
import { monitorEventLoopDelay } from "node:perf_hooks";
const h = monitorEventLoopDelay(); h.enable();
setInterval(() => console.log("p99 delay(ms):", h.percentile(99) / 1e6), 5000);
```

## worker_threads로 오프로드

CPU 작업을 워커 스레드로 보내면 이벤트 루프는 계속 요청을 받습니다.

```javascript
// main.js
import { Worker } from "node:worker_threads";

function heavyTask(data) {
  return new Promise((resolve, reject) => {
    const w = new Worker("./heavy-worker.js", { workerData: data });
    w.on("message", resolve);
    w.on("error", reject);
  });
}
```

```javascript
// heavy-worker.js
import { parentPort, workerData } from "node:worker_threads";
parentPort.postMessage(expensiveCompute(workerData));
```

실무에서는 요청마다 워커를 만들지 말고 piscina 같은 워커 풀을 씁니다. 워커 생성 비용이 있고, 워커 수도 코어 수 기준 상한이 필요하기 때문입니다.

## 정리

- Node에서 "CPU 여유 + 전체 무응답" 조합은 이벤트 루프 블로킹의 시그니처다.
- 이벤트 루프 지연(p99)을 상시 계측하는 것부터 시작한다.
- 수십 ms 이상 도는 동기 연산은 워커 풀로 보내는 게 원칙이다.

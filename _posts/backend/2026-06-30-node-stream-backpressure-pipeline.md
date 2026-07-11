---
title: Node 스트림에서 힙이 계속 자라다 OOM으로 죽는 이유, backpressure
date: 2026-06-30 19:21:00 +09:00
categories: [Backend]
tags: [nodejs, stream, backpressure, memory]
description: write()의 false 반환을 무시하면 내부 버퍼가 힙을 잠식하는 backpressure 문제를 이해하고, stream.pipeline으로 자동 처리하는 방법을 정리했다.
---

대용량 파일 변환 작업에서 힙이 45MB에서 수백 MB까지 자라다 OOM으로 죽는 패턴을 공부했습니다. 원인은 스트림 API의 반환값 하나를 무시한 데 있었습니다.

## write()의 false는 "그만 보내라"는 신호

읽기가 쓰기보다 빠르면(빠른 디스크 읽기 → 느린 네트워크 쓰기) 어딘가에 데이터가 쌓입니다. Writable 스트림의 `write()`는 내부 버퍼가 `highWaterMark`를 넘으면 `false`를 반환하는데, 이게 "버퍼 찼으니 `drain` 이벤트까지 기다려라"는 backpressure 신호입니다.

```javascript
// 나쁨: 반환값 무시 — 버퍼가 무한히 쌓인다
readable.on("data", (chunk) => {
  writable.write(chunk); // false여도 계속 밀어넣는다
});
```

이 코드는 동작은 하지만, 쓰기가 밀리는 만큼 청크가 전부 힙에 대기합니다. 파일이 크면 그대로 OOM입니다.

## pipeline이 전부 처리해 준다

수동으로 `drain`을 다루는 대신 `stream.pipeline`을 쓰면 backpressure 전파, 에러 전파, 리소스 정리가 전부 자동입니다.

```javascript
import { pipeline } from "node:stream/promises";
import { createReadStream, createWriteStream } from "node:fs";
import { createGzip } from "node:zlib";

await pipeline(
  createReadStream("huge.log"),
  createGzip(),
  createWriteStream("huge.log.gz")
);
```

쓰기가 밀리면 읽기가 자동으로 일시정지되므로 힙 사용량이 highWaterMark 수준에서 평평하게 유지됩니다. 구식 `readable.pipe(writable)`도 backpressure는 처리하지만 에러 처리·정리가 허술해서, 요즘은 pipeline이 표준입니다.

## 정리

- 스트림을 이벤트로 직접 이어붙이는 코드는 backpressure 누락을 의심한다.
- write()의 false는 경고가 아니라 프로토콜이다. 무시하면 힙으로 갚는다.
- 변환·복사 파이프는 stream.pipeline 하나로 통일하는 게 안전하다.

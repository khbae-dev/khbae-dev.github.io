---
title: 중단 버튼을 눌러도 LLM 과금이 안 멈추는 이유, 취소 신호 전파
date: 2026-07-06 19:23:00 +09:00
categories: [AI Dev]
tags: [llm, streaming, abortcontroller, cost]
description: 스트리밍 응답을 릴레이만 하는 서버가 클라이언트 disconnect를 감지하지 못해 업스트림 생성이 끝까지 도는 문제와 AbortController 전파를 정리했다.
---

LLM 챗 UI에서 "중단"을 눌러도 토큰 과금이 계속되는 문제를 공부했습니다. 취소 신호가 중간에서 끊긴 게 원인이었습니다.

## 릴레이 서버가 취소를 흘리지 않는다

전형적인 구조는 브라우저 → 우리 서버 → LLM API입니다. 우리 서버는 LLM의 스트리밍 응답을 받아 브라우저로 중계(relay)합니다. 사용자가 중단을 누르면 브라우저는 우리 서버로의 연결을 끊습니다. 그런데 서버가 이 disconnect를 감지해 업스트림(LLM API) 요청까지 취소하지 않으면, LLM은 아무도 안 보는 응답을 끝까지 생성합니다. 토큰은 계속 소모되고 과금됩니다. 긴 응답일수록 낭비가 큽니다.

## AbortController로 신호를 끝까지 잇는다

핵심은 클라이언트 연결의 종료를 감지해, 같은 취소 신호를 업스트림 fetch에 전파하는 겁니다.

```javascript
app.post("/chat", async (req, res) => {
  const ac = new AbortController();

  // 1. 클라이언트가 끊으면 업스트림도 취소
  req.on("close", () => ac.abort());

  // 2. 같은 signal을 LLM 호출에 전달
  const upstream = await fetch(LLM_URL, {
    method: "POST",
    body: JSON.stringify({ ...payload, stream: true }),
    signal: ac.signal,          // 이 signal이 취소를 전파한다
  });

  for await (const chunk of upstream.body) {
    res.write(chunk);
  }
  res.end();
});
```

`AbortController` 하나를 만들어 그 `signal`을 업스트림 요청에 넘기고, 클라이언트의 `close` 이벤트에서 `abort()`를 호출하면 됩니다. 그러면 중단하는 순간 LLM 연결이 끊기고 생성이 멈춰 토큰 소비가 0이 됩니다. 프레임워크에 따라 disconnect 감지 방법(`req.on('close')`, `request.signal` 등)은 다르지만 원리는 같습니다. 신호가 클라이언트에서 업스트림까지 끊김 없이 이어지게 하는 것.

## 정리

- 스트리밍 릴레이에서 클라이언트 취소가 업스트림까지 전파되지 않으면 과금이 새어나간다.
- AbortController의 signal을 업스트림 호출에 넘기고 close에서 abort()한다.
- 취소는 "연결의 사슬 전체"에서 이어져야 실제로 멈춘다. 한 곳만 끊으면 소용없다.

---
title: AI 응답이 몇 초씩 멈춰 있다면, SSE 스트리밍으로 체감 속도 바꾸기
date: 2026-06-07 16:00:00 +09:00
categories: [AI Dev]
tags: [llm, streaming, sse, ttft]
description: 응답을 한 번에 받아 생기는 긴 공백을, 토큰이 생성되는 즉시 흘려보내는 SSE 스트리밍으로 바꿔 체감 지연(TTFT)을 개선하는 방법을 정리했다.
---

LLM 응답이 다 나올 때까지 화면이 몇 초간 비어 있는 UX를 개선하는 방법을 공부했습니다. 총 생성 시간은 그대로인데 체감이 극적으로 달라지는 지점입니다.

## 기다림의 정체

기본(non-streaming) 호출은 모델이 응답을 전부 생성한 뒤 한 번에 반환합니다. 300 토큰짜리 답이면 그게 다 만들어질 때까지 사용자는 빈 화면을 봅니다. 첫 글자가 뜨는 시간(TTFT, Time To First Token)이 곧 전체 생성 시간이 되는 겁니다. 사람은 진행이 보이지 않는 대기를 실제보다 길게 느끼므로, 같은 8초라도 "멈춘 8초"는 훨씬 답답합니다.

## 토큰을 생기는 대로 흘려보낸다

스트리밍을 켜면 모델이 토큰을 만드는 즉시 조각조각 전송합니다. 서버-클라이언트는 SSE(Server-Sent Events)로 잇는 게 표준입니다.

```javascript
// 서버: 업스트림 스트림을 SSE로 중계
const stream = await llm.chat({ ...payload, stream: true });
res.setHeader("Content-Type", "text/event-stream");
for await (const chunk of stream) {
  res.write(`data: ${JSON.stringify(chunk.delta)}\n\n`);
}
res.write("data: [DONE]\n\n");
res.end();
```

```javascript
// 클라이언트: 도착하는 대로 화면에 붙인다
const res = await fetch("/chat", { method: "POST", body });
for await (const chunk of res.body) {   // 스트림 순회
  appendToUI(decode(chunk));            // 첫 토큰이 ~0.3초에 뜬다
}
```

총 생성 시간은 동일합니다. 바뀌는 건 TTFT입니다. 첫 글자가 8초 뒤가 아니라 0.3초 뒤에 나타나고, 이후 글자가 흐르듯 채워지니 사용자는 "빠르게 반응한다"고 느낍니다. 취소도 자연스러워집니다(연결을 끊으면 즉시 멈춤). 다만 스트리밍 중에는 응답 전체를 검증한 뒤 보여주는 후처리(가드레일, JSON 파싱)가 어려워지므로, 그런 경우는 부분 검증 전략을 함께 설계합니다.

## 정리

- 체감 지연을 지배하는 건 총 생성 시간이 아니라 첫 토큰까지의 시간(TTFT)이다.
- SSE 스트리밍으로 토큰을 즉시 흘리면 같은 시간에도 훨씬 빠르게 느껴진다.
- 스트리밍은 취소도 쉬워지지만, 전체 검증이 필요한 응답은 별도 전략이 필요하다.

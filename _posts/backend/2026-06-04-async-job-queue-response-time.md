---
title: API 응답 2초의 범인, 요청 안에서 돌리는 무거운 작업
date: 2026-06-04 02:43:00 +09:00
categories: [Backend]
tags: [queue, async, worker, api-design]
description: 메일 발송·PDF 생성 같은 무거운 작업을 요청-응답 사이클 안에서 처리해 느려지는 API를 작업 큐와 워커로 분리하는 패턴을 정리했다.
---

회원가입 API가 2초씩 걸리는 사례를 공부했습니다. DB insert는 수십 ms인데, 나머지 시간이 전부 환영 메일 발송, 프로필 썸네일 생성 같은 부가 작업이었습니다.

## 요청-응답 사이클에서 빼야 할 것들

사용자가 기다리는 시간에 꼭 포함돼야 하는 건 "응답에 필요한 작업"뿐입니다. 메일 발송, PDF 생성, 썸네일 리사이즈, 외부 시스템 동기화, 푸시 알림은 성공 여부가 응답 내용을 바꾸지 않습니다. 이런 작업이 요청 안에 있으면 응답 시간이 부가 작업의 합이 되고, 외부 SMTP가 느려지는 순간 회원가입 전체가 느려지는 결합도 문제까지 생깁니다.

## 큐에 넣고 즉시 응답

```javascript
// API: 필수 작업만 하고 나머지는 큐로
app.post("/signup", async (req, res) => {
  const user = await db.users.create(req.body);      // 필수
  await queue.add("welcome-email", { userId: user.id });   // 위임
  await queue.add("thumbnail", { userId: user.id });
  res.status(201).json(user);   // 즉시 응답 (~100ms)
});

// worker: 뒤에서 소비
queue.process("welcome-email", async (job) => {
  await mailer.sendWelcome(job.data.userId);
});
```

BullMQ(Redis), SQS, RabbitMQ 무엇이든 구조는 같습니다. 응답이 0.1초대로 내려가는 것과 별개로 얻는 게 많습니다. 워커는 실패 시 재시도할 수 있고, 트래픽 급증 시 큐가 버퍼가 되고, 워커 수로 처리량을 독립적으로 조절할 수 있습니다.

주의할 점은 작업의 멱등성입니다. 재시도가 있는 이상 같은 작업이 두 번 실행될 수 있으므로, "이미 보낸 메일인지" 확인하는 방어가 워커 쪽에 필요합니다.

## 정리

- 응답에 필요 없는 작업이 응답 시간을 결정하고 있다면 구조 문제다.
- 큐 + 워커 분리는 속도, 재시도, 부하 흡수를 한 번에 가져온다.
- 워커 작업은 멱등하게. 재시도는 언젠가 반드시 일어난다.

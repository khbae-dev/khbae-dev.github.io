---
title: 배포할 때마다 5xx가 튀는 이유, graceful shutdown 제대로 하기
date: 2026-06-15 16:04:00 +09:00
categories: [Backend]
tags: [deployment, sigterm, kubernetes, zero-downtime]
description: 종료 신호를 받자마자 프로세스가 죽으면서 진행 중 요청이 끊기는 문제를 SIGTERM 드레이닝, preStop 대기, keepalive 정리로 해결하는 과정을 정리했다.
---

배포 자동화가 잘 돼 있어도 배포 시각마다 5xx가 살짝 튀는 서비스가 많습니다. 원인을 공부해 보니 "종료 방식"의 문제였습니다.

## 종료 순간에 벌어지는 일

롤링 배포에서 구 파드가 내려갈 때 두 가지 레이스가 있습니다.

- 처리 중이던 요청: 프로세스가 즉사하면 연결이 리셋돼 클라이언트는 502/504를 받는다.
- 로드밸런서 전파 지연: 쿠버네티스가 파드를 endpoints에서 빼는 것과 SIGTERM을 보내는 건 동시에 일어난다. LB가 목록 갱신을 반영하기 전 몇 초 동안은 "죽어가는 파드"로 새 요청이 계속 들어온다.

## 3단 방어

**1. SIGTERM을 잡아 드레이닝.**

```javascript
const server = app.listen(8080);

process.on("SIGTERM", () => {
  server.close(() => process.exit(0)); // 새 연결 거부, 진행 중 요청은 완료
  setTimeout(() => process.exit(1), 25000); // 안전 상한
});
```

**2. preStop으로 전파 시간 벌기.** SIGTERM 전에 몇 초를 벌어 LB 목록 갱신을 기다립니다.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]
terminationGracePeriodSeconds: 40   # preStop + 드레이닝보다 크게
```

**3. keepalive 연결 정리.** 유휴 keepalive 소켓은 server.close가 기다려주지 않으므로, 종료 시 idle 커넥션을 닫는 처리(Node 18+ `server.closeIdleConnections()`)까지 해야 잔여 연결 리셋이 없습니다.

## 정리

- 무중단 배포의 마지막 조각은 인프라가 아니라 애플리케이션의 종료 코드다.
- preStop sleep은 꼼수가 아니라 LB 전파 지연에 대한 정석 대응이다.
- SIGTERM 핸들러, preStop, grace period 세 값의 합이 맞아야 5xx 0이 된다.

---
title: 접속자는 줄었는데 서버가 죽는 이유, 좀비 WebSocket 연결
date: 2026-06-23 19:23:00 +09:00
categories: [Backend]
tags: [websocket, heartbeat, fd, nodejs]
description: 비정상 종료된 WebSocket이 닫히지 않은 채 FD를 점유해 ulimit에 도달하는 문제와 heartbeat ping/pong으로 죽은 연결을 회수하는 방법을 정리했다.
---

WebSocket 서버 운영에서 직관과 어긋나는 장애를 공부했습니다. 동시 접속자가 줄고 있는데 서버 FD 사용량은 계속 늘다가 `EMFILE: too many open files`로 죽는 경우입니다.

## TCP는 상대의 실종을 모른다

클라이언트가 정상적으로 close 프레임을 보내면 서버도 소켓을 정리합니다. 문제는 비정상 종료입니다. 모바일 네트워크 전환, 노트북 덮개 닫기, 중간 NAT 장비의 매핑 만료 같은 경우 상대는 사라졌는데 서버 소켓은 ESTABLISHED로 남습니다. TCP는 데이터를 보내보기 전까지 상대의 실종을 감지하지 못하기 때문입니다. 이런 좀비 연결이 FD와 세션 메모리를 영구 점유하며 쌓입니다.

## heartbeat로 능동 감지

애플리케이션 레벨에서 주기적으로 생존 확인을 합니다. ws 라이브러리 기준 표준 패턴입니다.

```javascript
wss.on("connection", (ws) => {
  ws.isAlive = true;
  ws.on("pong", () => { ws.isAlive = true; });
});

const interval = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) return ws.terminate(); // 응답 없던 연결은 강제 종료
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);

wss.on("close", () => clearInterval(interval)); // 서버 종료 시 타이머 정리
```

30초마다 ping을 보내고, 다음 주기까지 pong이 없으면 죽은 연결로 판단해 `terminate()`로 즉시 회수합니다. 놓치기 쉬운 마무리가 두 가지 있는데, 연결별로 만든 타이머·리스너를 close 핸들러에서 정리하는 것과, 클라이언트 쪽에도 재연결 로직을 두는 것입니다.

## 정리

- WebSocket 서버에서 heartbeat는 옵션이 아니라 자원 회수 장치다.
- "접속자 수"와 "열린 소켓 수" 지표가 벌어지면 좀비 연결을 의심한다.
- terminate 후 타이머·리스너 정리까지 해야 누수가 0이 된다.

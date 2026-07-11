---
title: 내 코드가 아니라 서드파티 JS가 버튼을 멈추게 할 때, Partytown
date: 2026-06-07 16:01:00 +09:00
categories: [Frontend]
tags: [partytown, third-party, web-worker, inp]
description: GTM·광고·애널리틱스 스크립트가 메인 스레드를 점유해 INP를 망치는 문제와, 서드파티를 Web Worker로 옮기는 Partytown 접근을 정리했다.
---

성능 프로파일을 떠 보면 정작 무거운 건 우리 코드가 아니라 GTM, 광고, 애널리틱스인 경우가 많다는 걸 공부했습니다. 문제는 이걸 지울 권한이 개발자에게 없다는 점입니다.

## 지울 수 없다면 옮긴다

서드파티 태그들은 메인 스레드에서 실행되며 초기화 시점과 이벤트 시점마다 수십~수백 ms를 점유합니다. 마케팅 요구로 태그는 계속 늘어나므로, "빼자"는 협상은 대개 실패합니다. Partytown의 접근은 다릅니다. 스크립트를 Web Worker로 옮겨 실행하고, DOM 접근은 프록시로 중계합니다.

```html
<script>
  /* partytown 설정 */
  window.partytown = { forward: ["dataLayer.push", "gtag"] };
</script>
<script src="/~partytown/partytown.js"></script>

<!-- type만 바꾸면 워커에서 실행된다 -->
<script type="text/partytown" src="https://www.googletagmanager.com/gtag/js?id=G-XXX"></script>
<script type="text/partytown">
  window.dataLayer = window.dataLayer || [];
  function gtag(){ dataLayer.push(arguments); }
  gtag("js", new Date()); gtag("config", "G-XXX");
</script>
```

동작 원리가 흥미로운데, 워커에서의 DOM 접근을 동기처럼 보이게 하려고 동기 XHR과 Service Worker(또는 Atomics)를 조합해 메인 스레드와 통신합니다. 그래서 워커 안의 스크립트 실행이 아무리 무거워도 메인 스레드의 프레임 예산을 침범하지 않습니다.

주의점도 있습니다. DOM을 많이 만지는 스크립트(A/B 테스트 도구, 채팅 위젯)는 프록시 왕복 비용 때문에 오히려 부적합하고, 계측·픽셀류처럼 "받아서 보내는" 스크립트가 최적입니다. `forward` 설정으로 메인에서 호출하는 전역 함수를 워커로 연결하는 것도 잊기 쉬운 부분입니다.

## 정리

- 서드파티 성능 문제의 현실적 해법은 제거가 아니라 이주다.
- 계측·광고 픽셀류는 Partytown으로 워커로 보내면 메인 스레드가 비워진다.
- DOM 집약적인 위젯은 예외다. 스크립트 성격을 보고 골라 적용한다.

---
title: 탭을 오래 켜두면 느려지다 죽는 앱, useEffect cleanup 점검
date: 2026-06-20 19:22:00 +09:00
categories: [Frontend]
tags: [react, memory-leak, useeffect, abortcontroller]
description: 리스너·타이머·구독·fetch를 정리하지 않아 생기는 SPA 메모리 누수의 패턴과 cleanup 함수, AbortController 정리 습관을 정리했다.
---

"우리 서비스는 탭을 며칠 켜두면 느려져요"라는 증상의 원인을 공부했습니다. SPA는 페이지 전환에도 프로세스가 살아 있으므로, 안 치운 쓰레기가 계속 쌓입니다.

## 누수의 4대 단골

컴포넌트는 언마운트돼도, 컴포넌트가 바깥 세상에 등록해 둔 것들은 자동으로 사라지지 않습니다. 등록된 콜백이 컴포넌트의 클로저(그리고 그 안의 state, DOM 참조)를 붙잡고 있어서 GC도 못 합니다.

- window/document 이벤트 리스너
- setInterval / setTimeout
- WebSocket·스토어 구독
- 완료되지 않은 fetch의 setState 호출

## cleanup을 짝으로 쓰는 습관

useEffect에서 등록한 것은 반드시 반환 함수에서 해제합니다.

```jsx
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener("resize", onResize);
  const timer = setInterval(refresh, 5000);

  const ac = new AbortController();
  fetch(`/api/items`, { signal: ac.signal })
    .then(r => r.json())
    .then(setItems)
    .catch(e => { if (e.name !== "AbortError") throw e; });

  return () => {                       // cleanup: 등록의 역순 해제
    window.removeEventListener("resize", onResize);
    clearInterval(timer);
    ac.abort();                        // 진행 중 요청 취소
  };
}, []);
```

AbortController는 두 가지를 동시에 해결합니다. 언마운트 후 setState 시도(경고와 상태 꼬임)를 막고, 불필요한 네트워크도 끊습니다.

누수 확인은 DevTools Memory 탭에서 힙 스냅샷을 시간차로 두 번 찍어 비교하면 됩니다. 특정 컴포넌트 인스턴스가 페이지 전환을 반복해도 계속 늘어난다면 그 컴포넌트의 effect부터 봅니다.

## 정리

- SPA에서 등록과 해제는 항상 짝이다. cleanup 없는 구독성 effect는 리뷰에서 잡는다.
- fetch에는 AbortController를 기본으로 붙인다.
- 힙 스냅샷 비교는 누수를 심증에서 물증으로 바꿔준다.

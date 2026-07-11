---
title: 페이지 전환마다 로딩바가 길어지는 이유, 요청 워터폴
date: 2026-07-05 19:24:00 +09:00
categories: [Frontend]
tags: [frontend, react, router, prefetch, performance]
description: 컴포넌트 마운트 순서를 따라 fetch가 직렬로 발생하는 요청 워터폴 문제와 route loader 병렬 fetch, 호버 프리페치를 정리했다.
---

SPA에서 페이지 전환이 느린 원인을 추적하는 방법을 공부했습니다. 네트워크 탭을 폭포수(워터폴) 모양으로 읽는 게 시작이었습니다.

## 마운트 순서 = 요청 순서가 되는 문제

컴포넌트 안에서 데이터를 가져오는 구조(useEffect fetch)는 자연스럽게 워터폴을 만듭니다.

```
Page 마운트 → 유저 fetch (300ms)
  └ 응답 후 Dashboard 마운트 → 주문 fetch (300ms)
      └ 응답 후 OrderList 마운트 → 상세 fetch (300ms)
```

각 fetch는 부모의 응답이 와야 시작됩니다. 서로 의존성이 없는 데이터인데도 순차 실행돼 총 지연이 합산됩니다. 3단이면 900ms, 여기에 lazy 로드된 JS 청크까지 끼면 더 길어집니다.

## 라우트 진입 시점에 병렬로

라우터의 loader로 데이터 로딩을 라우트 레벨로 끌어올리면, 렌더와 무관하게 진입 즉시 병렬 실행됩니다.

```javascript
// React Router loader
{
  path: "/dashboard",
  loader: async () => {
    const [user, orders] = await Promise.all([
      fetchUser(),
      fetchOrders(),
    ]);
    return { user, orders };
  },
  element: <Dashboard />,
}
```

한 발 더 가면 전환 전에 미리 시작할 수 있습니다. 링크에 마우스를 올리는 순간(호버)과 실제 클릭 사이엔 보통 100~300ms가 있는데, 이 시간에 프리페치를 걸어두는 겁니다. 프레임워크별로 `<Link prefetch>` 계열 옵션이 있고, 직접 구현해도 onMouseEnter에서 loader를 미리 호출하는 정도입니다. 체감 전환 시간이 "클릭 후 1초"에서 "거의 즉시"로 바뀝니다.

## 정리

- 컴포넌트 내부 fetch는 편하지만 구조적으로 워터폴을 만든다.
- 의존성 없는 데이터는 라우트 loader에서 Promise.all로 병렬화한다.
- 호버 프리페치는 사용자의 의도를 앞질러 요청을 시작하는 공짜 최적화다.

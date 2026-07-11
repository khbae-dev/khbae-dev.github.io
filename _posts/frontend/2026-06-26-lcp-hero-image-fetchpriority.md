---
title: 히어로 이미지에 lazy를 걸면 LCP가 망가진다, 우선순위의 문제
date: 2026-06-26 19:24:00 +09:00
categories: [Frontend]
tags: [lcp, fetchpriority, preload, web-vitals]
description: 첫 화면 이미지까지 lazy loading을 걸어 LCP가 늦어지는 실수와 fetchpriority·preload로 로딩 우선순위를 바로잡는 방법을 정리했다.
---

이미지 최적화한다고 전부 `loading="lazy"`를 걸었다가 LCP가 오히려 나빠지는 사례를 공부했습니다. 용량이 아니라 우선순위의 문제였습니다.

## lazy는 "나중에"라는 선언이다

`loading="lazy"`는 뷰포트에 가까워질 때까지 로딩을 미루겠다는 뜻입니다. 폴드 아래 이미지에는 절약이지만, 첫 화면의 히어로 이미지에 걸면 브라우저가 "이건 급하지 않다"고 판단해 다른 리소스를 먼저 받습니다. LCP(최대 콘텐츠풀 페인트) 요소가 가장 늦게 도착하는 역효과가 나는 겁니다. 게다가 lazy 이미지는 레이아웃 계산 후에야 뷰포트 판정이 되므로 시작 자체가 늦습니다.

## 우선순위를 명시적으로 뒤집기

```html
<!-- 히어로(LCP 후보): 최우선으로 -->
<img src="/hero.avif" fetchpriority="high" decoding="async"
     width="1200" height="630" alt="...">

<!-- CSS 배경이거나 발견이 늦는 경우: preload로 앞당기기 -->
<link rel="preload" as="image" href="/hero.avif" fetchpriority="high">

<!-- 폴드 아래: 여기만 lazy -->
<img src="/section2.avif" loading="lazy" width="800" height="450" alt="...">
```

- `fetchpriority="high"`: 같은 이미지 큐 안에서 히어로를 앞으로 당깁니다.
- `preload`: HTML 파싱 초기에 다운로드를 시작시킵니다. JS로 렌더되거나 CSS 배경이라 발견이 늦는 이미지에 특히 효과적입니다.
- 반대로 히어로에 lazy·`decoding="sync"` 없이 위 조합만 해도 LCP가 초 단위로 줄어드는 경우가 흔합니다.

포맷(AVIF/WebP)과 반응형 srcset 같은 용량 최적화는 그 다음 단계입니다. 순서가 잘못되면 90KB짜리 이미지도 늦게 뜹니다.

## 정리

- LCP는 용량보다 "언제 다운로드가 시작되느냐"에 먼저 좌우된다.
- 히어로는 fetchpriority=high(+preload), lazy는 폴드 아래 전용.
- lazy 일괄 적용은 최적화가 아니라 우선순위 파괴일 수 있다.

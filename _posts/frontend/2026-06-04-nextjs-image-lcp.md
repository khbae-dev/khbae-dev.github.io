---
title: 페이지가 느린 진짜 이유가 이미지였다, next/image로 LCP 잡기
date: 2026-06-04 02:33:00 +09:00
categories: [Frontend]
tags: [nextjs, image, lcp, avif]
description: 원본 크기 이미지가 LCP를 블로킹하는 문제를 next/image의 자동 최적화, priority, 모던 포맷으로 해결하는 방법을 정리했다.
---

Lighthouse 점수를 깎아먹는 1순위 원인이 대체로 이미지라는 걸 정리했습니다. 특히 원본을 그대로 올린 히어로 이미지입니다.

## 1.8MB 원본이 하는 일

디자이너에게 받은 원본(4000px, 1.8MB)을 그대로 `<img>`로 쓰면 세 가지 낭비가 겹칩니다. 모바일 화면(400px)에 4000px을 보내는 해상도 낭비, JPEG/PNG의 포맷 비효율, 그리고 그 큰 파일이 LCP 요소라서 다운로드가 끝날 때까지 "다 뜬 것 같지 않은" 화면이 유지되는 겁니다.

## next/image가 자동화해 주는 것들

```jsx
import Image from "next/image";

<Image
  src="/hero.jpg"          // 원본을 넣어도
  width={1200} height={630}
  priority                  // LCP 이미지: lazy 대신 우선 로드 + preload
  sizes="(max-width: 768px) 100vw, 1200px"
  alt="메인 배너"
/>
```

이 한 컴포넌트가 처리하는 게 많습니다.

- 요청 기기 폭에 맞는 크기로 리사이즈해 srcset 제공
- 브라우저가 지원하면 AVIF/WebP로 자동 변환 (1.8MB JPEG가 수십 KB대로)
- width/height 기반 공간 예약으로 CLS 방지
- 기본 lazy loading, 단 `priority`를 준 이미지는 즉시+preload

핵심 규칙은 하나입니다. 첫 화면의 LCP 이미지에는 반드시 `priority`를 준다. 기본값(lazy)이 히어로에는 독이 되기 때문입니다. Next가 아니어도 원리는 같습니다. 리사이즈 + AVIF/WebP + srcset + 히어로 우선순위, 이 네 가지를 수동으로 조합하면 됩니다.

## 정리

- 이미지 최적화의 본질은 "그 화면에 필요한 만큼만 보내기"다.
- next/image는 리사이즈·포맷·공간 예약·우선순위를 한 번에 처리한다.
- LCP 이미지에 priority, 나머지는 lazy — 이 구분만 지켜도 절반은 성공이다.

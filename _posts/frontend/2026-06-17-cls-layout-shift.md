---
title: 버튼 누르려다 광고를 누르는 이유, CLS 잡기
date: 2026-06-17 19:21:00 +09:00
categories: [Frontend]
tags: [cls, web-vitals, layout, font]
description: 이미지·광고·웹폰트가 늦게 로드되며 레이아웃을 밀어내는 CLS의 원인별 처방을 정리했다. 공간 예약이 핵심이다.
---

클릭하려는 순간 화면이 밀려 엉뚱한 걸 누르게 되는 경험의 정체를 정리했습니다. Web Vitals 지표로는 CLS(Cumulative Layout Shift)입니다.

## 원인은 전부 "늦게 도착하는 것들"

레이아웃 이동은 나중에 도착한 리소스가 자리를 비집고 들어올 때 생깁니다. 주범이 정해져 있습니다.

- 크기 명시 없는 이미지: 로드 완료 순간 본문을 아래로 밀어낸다.
- 광고·임베드 슬롯: 응답이 와야 크기를 아는 iframe이 갑자기 커진다.
- 웹폰트: 폴백 폰트로 그려진 텍스트가 웹폰트 도착 후 재배치된다(FOIT/FOUT).
- 동적 배너: "앱으로 보기" 같은 요소가 상단에 끼어든다.

## 처방: 도착 전에 공간을 예약한다

```html
<!-- 이미지: 크기를 미리 알려 비율만큼 공간 확보 -->
<img src="/chart.webp" width="800" height="450" alt="...">
```

```css
/* 반응형이면 aspect-ratio로 예약 */
.thumbnail { width: 100%; aspect-ratio: 16 / 9; }

/* 광고 슬롯: 최소 높이를 먼저 잡아둔다 */
.ad-slot { min-height: 280px; }

/* 웹폰트: 교체는 허용하되 크기 차이를 줄인다 */
@font-face {
  font-family: "Pretendard";
  src: url(/fonts/Pretendard.woff2) format("woff2");
  font-display: swap;
  size-adjust: 98%;   /* 폴백과의 크기 차이 보정 */
}
```

동적으로 끼어드는 배너는 기존 콘텐츠를 밀지 않는 위치(overlay, 하단 고정)로 설계합니다. 측정은 Chrome DevTools Performance 탭에서 Layout Shift 항목을 보면 어떤 요소가 언제 밀었는지 프레임 단위로 확인됩니다.

## 정리

- CLS의 해법은 한 문장이다: 늦게 올 것들의 자리를 미리 잡아둔다.
- 이미지 width/height(또는 aspect-ratio)는 협상 불가한 기본기.
- 광고 min-height, font-display 조정까지 하면 0.1 이하(양호)는 충분히 나온다.

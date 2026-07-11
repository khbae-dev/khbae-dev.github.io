---
title: 로컬은 멀쩡한데 운영만 화면이 두 번 그려지는 hydration mismatch
date: 2026-06-09 16:04:00 +09:00
categories: [Frontend]
tags: [react, ssr, hydration, nextjs]
description: 서버 HTML과 클라이언트 첫 렌더가 달라 전체 재렌더가 일어나는 hydration mismatch의 원인 값들과 placeholder 패턴을 정리했다.
---

SSR 앱에서 운영 환경만 화면이 번쩍이며 두 번 그려지는 문제를 공부했습니다. 콘솔의 "Hydration failed..." 경고와 같은 뿌리입니다.

## 리액트는 불일치를 용서하지 않는다

hydration은 서버가 보낸 HTML 위에 리액트가 이벤트와 상태를 "접붙이는" 과정입니다. 이때 클라이언트에서 계산한 첫 렌더 결과가 서버 HTML과 일치해야 한다는 전제가 있습니다. 한 글자라도 다르면 리액트는 서버 HTML을 신뢰할 수 없다고 보고 통째로 버린 뒤 처음부터 다시 그립니다. SSR의 성능 이점이 사라지고 화면이 깜빡이는 겁니다.

불일치를 만드는 값들은 정해져 있습니다. 실행 시점·환경에 따라 달라지는 것들입니다.

- `new Date()`, 상대 시간("3분 전") — 서버와 클라이언트의 실행 시각이 다르다
- `Math.random()`, 랜덤 ID
- `window`, `localStorage` 접근 — 서버엔 없어서 분기 결과가 달라진다
- 타임존·로케일 의존 포매팅

## 마운트 후에만 진짜 값을 채운다

원칙은 "첫 렌더는 서버와 같게, 클라이언트 전용 값은 마운트 후에"입니다.

```jsx
function LastSeen({ timestamp }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []); // 마운트 후에만 true

  // 서버 렌더와 클라 첫 렌더가 동일한 placeholder
  if (!mounted) return <span>-</span>;

  // 마운트 이후 = hydration 완료 후 → 클라 전용 값 안전
  return <span>{formatRelative(timestamp)}</span>;
}
```

시간처럼 살짝 다른 게 불가피한 경우엔 `suppressHydrationWarning`을 그 요소에만 제한적으로 쓸 수 있지만, 남용하면 진짜 버그를 덮습니다. 근본적으로는 "서버가 아는 값(props로 내려온 데이터)"과 "클라이언트만 아는 값"을 구분하는 습관이 해결책입니다.

## 정리

- hydration mismatch는 랜덤·시간·window 의존 값이 첫 렌더에 섞였다는 신호다.
- 첫 렌더는 서버와 동일하게, 클라 전용 값은 useEffect 이후에 채운다.
- suppressHydrationWarning은 국소 마취지 치료제가 아니다.

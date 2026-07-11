---
title: 타이핑할 때마다 검색창이 버벅일 때, useDeferredValue
date: 2026-06-24 19:22:00 +09:00
categories: [Frontend]
tags: [react, usedeferredvalue, inp, concurrent]
description: 입력값과 무거운 리스트 렌더가 한 덩어리로 묶여 키 입력이 밀리는 문제를 useDeferredValue로 분리해 INP를 개선하는 방법을 정리했다.
---

검색어를 입력할 때마다 화면이 0.3초씩 멈추는 검색 UI의 개선 방법을 공부했습니다. React 18 동시성 기능이 정확히 이 문제를 위해 있었습니다.

## 입력과 결과 렌더가 한 몸인 게 문제

```jsx
const [query, setQuery] = useState("");
// query가 바뀔 때마다 무거운 필터링 + 수천 행 렌더가 같은 렌더에서 실행
const results = filterHeavy(items, query);
```

키 입력 → setState → 리렌더 한 번에 "입력창 갱신"과 "무거운 결과 리스트 렌더"가 함께 들어 있습니다. 리스트 렌더가 300ms 걸리면 다음 키 입력 반응도 300ms 밀립니다. 이게 INP(Interaction to Next Paint)를 망치는 전형적인 패턴입니다.

## 급한 것과 급하지 않은 것 분리

`useDeferredValue`는 값의 "한 박자 늦은 복제본"을 만듭니다. 입력창은 원본으로 즉시 갱신되고, 무거운 리스트는 지연된 값으로 백그라운드에서 렌더됩니다.

```jsx
const [query, setQuery] = useState("");
const deferredQuery = useDeferredValue(query);

const results = useMemo(
  () => filterHeavy(items, deferredQuery),
  [items, deferredQuery]
);

const isStale = query !== deferredQuery; // 갱신 중 표시용

return (
  <>
    <input value={query} onChange={(e) => setQuery(e.target.value)} />
    <div style={{ opacity: isStale ? 0.6 : 1 }}>
      <ResultList results={results} />
    </div>
  </>
);
```

핵심은 백그라운드 렌더가 "중단 가능"하다는 점입니다. 렌더 도중 새 키 입력이 오면 리액트가 진행 중이던 리스트 렌더를 버리고 입력부터 처리합니다. debounce처럼 고정 지연을 넣는 게 아니라, 기기 성능에 맞춰 알아서 양보한다는 게 차이입니다. 결과 계산이 정말 무겁다면 리스트 컴포넌트를 memo로 감싸야 지연 값이 같을 때 재렌더를 건너뜁니다.

## 정리

- 입력 지연의 원인은 대부분 "입력과 무거운 렌더가 같은 업데이트에 묶여서"다.
- useDeferredValue는 긴급(입력)과 비긴급(결과)을 분리하고 비긴급을 중단 가능하게 만든다.
- debounce 대체가 아니라 렌더 우선순위 도구라는 관점으로 쓰는 게 맞다.

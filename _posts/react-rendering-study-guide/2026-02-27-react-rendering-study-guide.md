---
title: 리액트 렌더링 완전 정리
date: 2026-02-27 23:20:00 +09:00
tags: [react, rendering, javascript, frontend, performance]
description: 리렌더링 트리거, 렌더/커밋 단계, Reconciliation, 메모이제이션, 동시성 렌더링까지 리액트 렌더링 핵심을 실무 중심으로 정리했다.
---

리액트 성능 이슈의 대부분은 "왜 렌더가 다시 일어났는지"를 이해하면 절반 이상 해결됩니다.
이 글은 렌더링 동작을 추상적인 설명이 아니라, 디버깅과 최적화에 바로 쓰는 관점으로 정리합니다.

## 1. 렌더링을 한 줄로 정의하면

리액트 렌더링은 다음 두 단계로 나뉩니다.

- Render phase: 컴포넌트 함수를 다시 실행해 다음 UI 트리를 계산
- Commit phase: 계산 결과를 실제 DOM에 반영

중요한 점은 "컴포넌트 함수가 실행되는 것"과 "실제 DOM 변경"이 같은 의미가 아니라는 것입니다.
함수가 다시 실행돼도 이전 결과와 같으면 DOM 변경은 최소화됩니다.

## 2. 리렌더링이 발생하는 대표 트리거

실무에서 가장 자주 만나는 트리거는 아래 4가지입니다.

- state 변경 (`setState`, `useState`, `useReducer`)
- 부모 컴포넌트 리렌더링
- props 변경
- context 값 변경

```jsx
function Parent({ theme }) {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>+1</button>
      <Child theme={theme} />
    </>
  );
}

function Child({ theme }) {
  console.log("Child render");
  return <div>{theme}</div>;
}
```

위 코드에서 `count`만 바뀌어도 `Parent`가 다시 렌더되고, 기본적으로 `Child`도 같이 다시 실행됩니다.
즉 "props가 안 바뀌었는데 왜 렌더되지?"라는 상황은 부모 리렌더링 때문인 경우가 많습니다.

## 3. Reconciliation: 무엇을 바꿀지 결정하는 과정

리액트는 이전 트리와 다음 트리를 비교해 필요한 변경만 적용합니다.
이 과정에서 핵심은 `type`과 `key`입니다.

- 같은 type + 같은 key: 기존 노드를 재사용
- type이 달라짐: 기존 트리를 버리고 새로 마운트
- 리스트에서 key가 불안정함(index key 등): 상태가 엉뚱한 아이템으로 이동할 수 있음

```jsx
{
  items.map((item) => (
    <TodoRow key={item.id} item={item} />
  ));
}
```

리스트 렌더링의 기본 원칙은 "UI 순서가 바뀔 수 있으면 index를 key로 쓰지 않는다"입니다.

## 4. React.memo, useMemo, useCallback은 언제 쓰나

세 도구 모두 "불필요한 재계산/재렌더링"을 줄이는 용도입니다.
다만 무조건 쓰면 오히려 비교 비용과 코드 복잡도만 늘어날 수 있습니다.

### `React.memo`

props가 같으면 자식 컴포넌트 재렌더링을 건너뜁니다.

```jsx
const ExpensiveChart = React.memo(function ExpensiveChart({ data }) {
  return <Chart data={data} />;
});
```

전제 조건:

- 자식 렌더 비용이 실제로 비싸야 함
- props 참조 안정성이 어느 정도 보장돼야 함

### `useMemo`

비싼 계산 결과를 캐시합니다.

```jsx
const visibleItems = useMemo(() => {
  return filterAndSortItems(items, query);
}, [items, query]);
```

### `useCallback`

함수 참조를 안정화해 memoized 자식의 불필요한 갱신을 줄입니다.

```jsx
const handleSelect = useCallback((id) => {
  setSelectedId(id);
}, []);
```

## 5. 동시성 렌더링: 사용자 입력을 먼저 살리는 전략

입력은 즉시 반응하고, 무거운 화면 업데이트는 조금 늦춰도 되는 경우가 많습니다.
이때 `startTransition`, `useDeferredValue`가 유용합니다.

```jsx
import { startTransition, useMemo, useState } from "react";

function SearchPage({ allUsers }) {
  const [keyword, setKeyword] = useState("");
  const [query, setQuery] = useState("");

  const onChange = (e) => {
    const next = e.target.value;
    setKeyword(next); // 입력 반응은 즉시

    startTransition(() => {
      setQuery(next); // 무거운 리스트 갱신은 낮은 우선순위
    });
  };

  const filtered = useMemo(() => {
    return allUsers.filter((u) =>
      u.name.toLowerCase().includes(query.toLowerCase()),
    );
  }, [allUsers, query]);

  return (
    <>
      <input value={keyword} onChange={onChange} />
      <p>count: {filtered.length}</p>
    </>
  );
}
```

핵심은 "모든 상태 업데이트를 빠르게"가 아니라 "사용자 체감 우선순위를 나누는 것"입니다.

## 6. StrictMode에서 두 번 렌더되는 이유

개발 모드에서 StrictMode를 켜면 일부 로직이 두 번 실행되는 것처럼 보일 수 있습니다.
이는 부작용(side effect) 안전성을 조기에 발견하기 위한 동작입니다.

- 개발 환경에서만 주로 관찰됨
- 프로덕션 동작과 구분해서 분석해야 함
- `useEffect` 정리(cleanup) 누락 여부를 점검하는 좋은 신호

렌더링 문제를 분석할 때는 "개발 모드 특성인지, 실제 프로덕션 문제인지"를 먼저 분리하세요.

## 7. 렌더링 성능 디버깅 순서

감으로 최적화하지 말고, 아래 순서로 확인하면 시행착오를 줄일 수 있습니다.

1. React DevTools Profiler로 느린 커밋 구간 확인
2. 어떤 컴포넌트가 자주 리렌더되는지 확인
3. 리렌더링 원인이 state/props/context/부모 중 무엇인지 분류
4. 필요한 지점에만 memoization 적용
5. 적용 전/후 커밋 시간 비교

최적화의 목표는 "렌더링 횟수 0"이 아니라 "사용자 체감 지연 최소화"입니다.

## 8. 실무 체크리스트

- 상태는 최소 단위로 나누고, 변경 범위를 작게 유지했는가
- 리스트 key는 안정적인 식별자인가
- 무거운 계산은 `useMemo` 등으로 관리했는가
- `React.memo`는 실제 병목 구간에만 적용했는가
- 입력 반응성과 대량 렌더링 우선순위를 분리했는가
- Profiler 수치로 최적화 효과를 검증했는가

리액트 렌더링을 제대로 이해하면 성능뿐 아니라 코드 구조도 좋아집니다.
특히 "왜 다시 렌더됐는지 설명 가능한 코드"를 목표로 잡으면 팀 생산성이 크게 올라갑니다.

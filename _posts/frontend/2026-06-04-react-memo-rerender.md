---
title: 글자 하나 입력에 리렌더 1,000번, memo 3형제 바로 쓰기
date: 2026-06-04 02:39:00 +09:00
categories: [Frontend]
tags: [react, memo, usecallback, usememo]
description: 부모 리렌더가 자식 전체로 전파되는 구조를 이해하고 React.memo, useCallback, useMemo를 원인 진단 후에 조합하는 방법을 정리했다.
---

입력창에 한 글자 칠 때마다 리스트 전체가 다시 렌더되는 문제를 정리했습니다. 도구(React.memo)보다 원인 이해가 먼저였습니다.

## 리렌더는 아래로 흐른다

리액트의 기본 규칙: 부모가 리렌더되면 자식도 리렌더됩니다. props가 안 바뀌었어도요. 검색창 state가 리스트와 같은 컴포넌트에 있으면, 타이핑마다 부모가 리렌더되고 1,000개 행이 전부 다시 실행됩니다.

첫 번째 처방은 memo가 아니라 상태 내리기입니다. 입력 state를 작은 컴포넌트로 격리하면 리렌더 범위가 그 안에 갇힙니다. 그래도 부모 리렌더가 불가피한 구조라면 memo를 씁니다.

## memo가 무력화되는 이유와 3형제 조합

`React.memo`는 props를 얕은 비교해서 같으면 렌더를 건너뜁니다. 함정은 "렌더마다 새로 만들어지는 참조"입니다.

```jsx
// memo가 무력화되는 코드: onSelect와 options가 매 렌더 새 참조
<MemoRow onSelect={(id) => select(id)} options={{ dense: true }} />
```

```jsx
const Row = React.memo(function Row({ item, onSelect }) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});

function List({ items }) {
  const [q, setQ] = useState("");

  const onSelect = useCallback((id) => select(id), []);        // 함수 참조 고정
  const filtered = useMemo(
    () => items.filter(i => i.name.includes(q)),               // 계산 결과 캐시
    [items, q]
  );

  return filtered.map(item =>
    <Row key={item.id} item={item} onSelect={onSelect} />);
}
```

React.memo(컴포넌트 스킵) + useCallback(함수 참조 유지) + useMemo(객체·계산 결과 유지)는 이렇게 한 세트로 맞물려야 동작합니다. 적용 전후는 DevTools Profiler의 "왜 렌더됐는지" 표시로 검증합니다. 안 바뀐 행이 회색(렌더 스킵)으로 나오면 성공입니다.

## 정리

- 첫 수는 memo가 아니라 상태를 필요한 곳으로 내리는 구조 개선이다.
- memo는 참조 동일성이 생명이라 useCallback/useMemo와 세트로만 완성된다.
- Profiler로 원인을 확인하고 넣는다. 예방적 memo 도배는 코드만 무겁게 한다.

---
title: 1만 행 리스트가 8fps로 버벅일 때, 가상화로 보이는 것만 그리기
date: 2026-06-05 16:00:00 +09:00
categories: [Frontend]
tags: [react, virtualization, react-window, performance]
description: 대량 리스트를 전부 DOM으로 그려서 생기는 성능 문제를, 뷰포트에 보이는 행만 렌더하는 windowing 기법으로 해결하는 방법을 정리했다.
---

대량 데이터 테이블 성능 문제의 표준 해법을 정리했습니다. 핵심 관찰은 단순합니다. 화면에는 어차피 15행 정도만 보인다는 겁니다.

## DOM 1만 개의 비용

1만 행을 전부 렌더하면 DOM 노드가 (행당 자식 포함) 수만~수십만 개가 됩니다. 초기 렌더가 느린 건 물론이고, 스크롤할 때마다 스타일 계산·레이아웃·페인트 범위가 커져 프레임이 떨어집니다. 메모리도 수백 MB씩 먹습니다. React.memo를 아무리 발라도 "존재하는 DOM"의 비용은 사라지지 않습니다.

## windowing: 뷰포트 밖은 존재하지 않는다

가상화(virtualization/windowing)는 스크롤 위치를 기준으로 보이는 행 + 버퍼 몇 개만 실제로 렌더하고, 전체 높이는 빈 컨테이너로 흉내 냅니다.

```jsx
import { FixedSizeList } from "react-window";

<FixedSizeList
  height={600}        // 뷰포트 높이
  itemCount={10000}
  itemSize={40}       // 행 높이(px)
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>  {/* style 필수: 절대 위치 배치 */}
      <Row item={items[index]} />
    </div>
  )}
</FixedSizeList>
```

스크롤하면 벗어난 행의 DOM이 재활용되고, 어떤 순간에도 실제 DOM은 ~20개 수준으로 유지됩니다. 행 높이가 제각각이면 `VariableSizeList`, 더 유연한 측정이 필요하면 TanStack Virtual 같은 대안도 있습니다.

주의점은 두 가지입니다. 행마다 내려주는 style을 빼먹으면 배치가 깨지고, 브라우저 Ctrl+F 검색이 "그려진 행"에만 동작하므로 검색 UX는 자체 필터로 보완해야 합니다.

## 정리

- 리스트 성능의 상한은 최적화 기법이 아니라 DOM 개수가 정한다.
- 수백 행이 넘는 리스트는 가상화가 기본값이다.
- 고정 높이면 FixedSizeList로 10분 안에 적용된다. 60fps는 덤이다.

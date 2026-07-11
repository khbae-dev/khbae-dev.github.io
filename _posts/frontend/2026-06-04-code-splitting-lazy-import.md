---
title: 첫 화면이 2MB 번들을 기다리는 문제, 코드 스플리팅
date: 2026-06-04 02:36:00 +09:00
categories: [Frontend]
tags: [code-splitting, lazy, bundle, webpack]
description: 안 쓰는 코드까지 첫 로딩에 포함되는 단일 번들 문제를 route 단위 lazy import로 나누고, 번들 분석으로 검증하는 과정을 정리했다.
---

첫 화면 로딩이 느린 SPA의 단골 원인을 정리했습니다. 사용자는 로그인 페이지를 보러 왔는데, 관리자 대시보드·차트 라이브러리·PDF 뷰어까지 전부 다운로드하고 있었던 겁니다.

## 단일 번들의 문제

번들러는 기본적으로 import 그래프 전체를 하나의 파일로 묶습니다. 앱이 커질수록 번들이 수 MB로 불어나고, 첫 화면에 필요한 코드가 5%여도 100%를 받아야 렌더가 시작됩니다. 다운로드 시간뿐 아니라 파싱·컴파일 시간도 번들 크기에 비례해 늘어납니다.

## 경계에서 자른다: route가 첫 번째 칼자국

`import()` 동적 구문을 만나면 번들러가 그 지점을 청크 경계로 잘라 별도 파일을 만들고, 실제 사용 시점에 네트워크로 가져옵니다. React에서는 lazy + Suspense로 감쌉니다.

```jsx
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const AdminPanel = lazy(() => import("./pages/AdminPanel"));

<Suspense fallback={<PageSkeleton />}>
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/dashboard" element={<Dashboard />} />
    <Route path="/admin" element={<AdminPanel />} />
  </Routes>
</Suspense>
```

자르는 우선순위는 효율 순입니다. 라우트(페이지) 단위가 가장 크고 자연스러운 경계, 그다음이 무거운 라이브러리를 쓰는 컴포넌트(차트, 에디터, PDF), 마지막이 모달처럼 조건부로만 열리는 UI입니다.

전환 시 로딩 공백이 신경 쓰이면 링크 호버 시점에 `import()`를 미리 호출하는 프리페치로 메꿉니다. 효과 검증은 번들 분석기(rollup-plugin-visualizer 등)로 청크 구성을 눈으로 확인하는 것까지가 한 사이클입니다.

## 정리

- 첫 로딩의 적은 "지금 안 쓰는 코드"다. 초기 번들에서 몰아낸다.
- 라우트 단위 lazy가 가성비 1위, 무거운 라이브러리 컴포넌트가 2위.
- 번들 분석기로 before/after를 확인해야 진짜 줄었는지 안다.

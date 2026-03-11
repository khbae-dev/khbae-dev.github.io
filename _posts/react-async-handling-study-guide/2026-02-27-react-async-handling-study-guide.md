---
title: 리액트 비동기 처리 학습 가이드
date: 2026-02-27 22:00:00 +09:00
tags: [react, javascript, async, frontend]
description: 로딩/에러 상태 관리, 요청 취소, 경쟁 상태 방지, 커스텀 훅 재사용까지 리액트 비동기 처리를 실전 중심으로 정리한 학습 노트.
---

리액트 앱에서는 비동기 작업이 기본입니다. API 요청, 지연 로딩, 사용자 액션 기반 업데이트, 백그라운드 새로고침까지 모두 비동기 흐름을 가집니다.
이 글은 실무와 학습에 바로 적용할 수 있는 패턴만 간결하게 정리한 스터디 노트입니다.

## 이번에 다시 보면서 메모한 점

- `loading/error/data`는 렌더 상태 문제이고, 요청 취소는 네트워크 수명 주기 문제라서 따로 봐야 했다.
- `mounted` 플래그만으로는 늦게 끝난 응답을 막을 수 있어도 HTTP 요청 자체는 취소되지 않는다.
- 검색 UI처럼 입력이 빠른 화면은 성공/실패 처리보다 "최신 요청만 반영" 규칙이 먼저 필요했다.

## 1. 기본기: loading, error, data 세 가지 상태

초보 단계에서 가장 흔한 실수는 성공 케이스만 처리하는 것입니다.
비동기 UI는 최소 다음 세 상태로 모델링해야 합니다.

- loading
- error
- success (data)

```jsx
import { useEffect, useState } from "react";

export default function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let mounted = true;

    async function fetchUsers() {
      try {
        setLoading(true);
        setError(null);
        const res = await fetch("/api/users");
        if (!res.ok) throw new Error("Failed to fetch users");
        const json = await res.json();
        if (mounted) setUsers(json);
      } catch (err) {
        if (mounted) setError(err.message);
      } finally {
        if (mounted) setLoading(false);
      }
    }

    fetchUsers();
    return () => {
      mounted = false;
    };
  }, []);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

이 패턴으로 얻는 이점:

- 렌더 분기가 예측 가능해진다
- 언마운트 이후 상태 업데이트를 막을 수 있다

## 2. AbortController로 요청 취소하기

`mounted` 플래그는 상태 업데이트만 막고 요청 자체는 계속 실행됩니다.
실제 네트워크 요청을 취소하려면 `AbortController`를 사용하세요.

```jsx
useEffect(() => {
  const controller = new AbortController();

  async function fetchUsers() {
    try {
      const res = await fetch("/api/users", { signal: controller.signal });
      const json = await res.json();
      setUsers(json);
    } catch (err) {
      if (err.name !== "AbortError") {
        setError(err.message);
      }
    }
  }

  fetchUsers();
  return () => controller.abort();
}, []);
```

이게 중요한 이유:

- 불필요한 네트워크와 메모리 사용을 줄인다
- 페이지 이동 후 늦게 도착한 응답이 상태를 덮어쓰는 문제를 줄인다

## 3. 경쟁 상태: 최신 요청만 반영하기

검색어처럼 입력이 빠르게 바뀌는 경우, 응답 도착 순서가 요청 순서와 달라질 수 있습니다.
보호 로직이 없으면 오래된 응답이 최신 데이터를 덮어쓸 수 있습니다.

```jsx
import { useEffect, useRef, useState } from "react";

function SearchUsers({ keyword }) {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const requestIdRef = useRef(0);

  useEffect(() => {
    if (!keyword) {
      setUsers([]);
      return;
    }

    const currentRequestId = ++requestIdRef.current;
    const controller = new AbortController();

    async function run() {
      try {
        setLoading(true);
        setError(null);
        const res = await fetch(`/api/users?keyword=${encodeURIComponent(keyword)}`, {
          signal: controller.signal,
        });
        if (!res.ok) throw new Error("Search failed");
        const json = await res.json();

        // 오래된 응답은 무시한다.
        if (currentRequestId === requestIdRef.current) {
          setUsers(json);
        }
      } catch (err) {
        if (err.name !== "AbortError") setError(err.message);
      } finally {
        if (currentRequestId === requestIdRef.current) {
          setLoading(false);
        }
      }
    }

    run();
    return () => controller.abort();
  }, [keyword]);

  if (loading) return <p>Searching...</p>;
  if (error) return <p>{error}</p>;
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

## 4. 병렬 요청 처리

서로 독립적인 데이터 소스라면 `Promise.all`로 병렬 실행하세요.

```jsx
useEffect(() => {
  async function loadDashboard() {
    try {
      setLoading(true);
      const [userRes, notifRes] = await Promise.all([
        fetch("/api/me"),
        fetch("/api/notifications"),
      ]);
      if (!userRes.ok || !notifRes.ok) throw new Error("Dashboard fetch failed");

      const [user, notifications] = await Promise.all([
        userRes.json(),
        notifRes.json(),
      ]);
      setUser(user);
      setNotifications(notifications);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }

  loadDashboard();
}, []);
```

실전 기준:

- 뒤 요청이 앞 요청 결과에 의존하면 순차 `await`
- 서로 독립이면 병렬 처리로 전체 대기 시간을 줄인다

## 5. 재사용 가능한 async 훅 만들기

같은 패턴이 반복되면 커스텀 훅으로 추출해 재사용하세요.

```jsx
import { useCallback, useState } from "react";

export function useAsync(asyncFn) {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [data, setData] = useState(null);

  const run = useCallback(async (...args) => {
    try {
      setLoading(true);
      setError(null);
      const result = await asyncFn(...args);
      setData(result);
      return result;
    } catch (err) {
      setError(err);
      throw err;
    } finally {
      setLoading(false);
    }
  }, [asyncFn]);

  return { loading, error, data, run };
}
```

사용 예시:

```jsx
const { loading, error, data: user, run: loadUser } = useAsync(async (id) => {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error("User fetch failed");
  return res.json();
});

useEffect(() => {
  loadUser(userId);
}, [loadUser, userId]);
```

## 6. React Query 또는 SWR를 써야 하는 시점

캐싱, stale-while-revalidate, 재시도, 쿼리 무효화가 본격적으로 필요해지면
직접 모든 로직을 구현하기보다 서버 상태 라이브러리를 도입하는 편이 효율적입니다.

- 단순한 화면: `useEffect` + 로컬 상태만으로 충분할 수 있다
- 중대형 앱: React Query/SWR 도입 효과가 빠르게 나타난다

## 7. 실전 체크리스트

리액트 비동기 기능을 배포하기 전, 아래 항목을 확인하세요.

- loading, error, success 상태가 모두 렌더링된다
- 언마운트/의존성 변경 시 요청이 취소된다
- 오래된 응답이 최신 상태를 덮어쓰지 못한다
- 독립 요청은 병렬 처리한다
- 반복 로직은 커스텀 훅 또는 데이터 라이브러리로 추출한다

이 체크리스트가 통과되면 비동기 UI 품질이 크게 안정됩니다.
이번에 다시 정리하면서 느낀 점은 비동기 코드를 예쁘게 쓰는 것보다 상태 전이를 설명 가능하게 만드는 쪽이 훨씬 중요하다는 점이었습니다.

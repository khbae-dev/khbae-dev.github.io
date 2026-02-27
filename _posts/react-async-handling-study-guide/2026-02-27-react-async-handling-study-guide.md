---
title: React Async Handling Study Guide
date: 2026-02-27 22:00:00 +09:00
tags: [react, javascript, async, frontend]
description: A practical study note on async handling in React, including loading and error states, request cancellation, race condition prevention, and a reusable custom hook.
---

React apps are full of async work: API requests, lazy loading, user-triggered updates, and background refreshes.
This post is a practical study note focused on patterns you can apply immediately.

## 1. Baseline: loading, error, data

The most common beginner mistake is handling only the successful case.
Always model async UI as at least three states:

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

What this solves:

- predictable render branches
- no state update after unmount

## 2. Cancel requests with AbortController

Using a `mounted` flag prevents a state update, but the request still runs.
For real cancellation, use `AbortController`.

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

Why this matters:

- avoids unnecessary network and memory work
- prevents stale responses from updating state after navigation

## 3. Race conditions: latest request wins

When query input changes quickly, responses may arrive out of order.
Without protection, old data can overwrite new data.

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

        // Ignore stale result.
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

## 4. Parallel requests

When data sources are independent, run them in parallel with `Promise.all`.

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

Rule of thumb:

- use sequential `await` only when later request depends on earlier result
- otherwise use parallel flow to reduce total waiting time

## 5. Reusable async hook

As the same pattern repeats, extract it into a custom hook.

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

Usage:

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

## 6. When to use React Query or SWR

If your app needs caching, stale-while-revalidate, retries, and query invalidation at scale,
use a server-state library instead of hand-rolling everything.

- light and simple screen: native `useEffect` + local state can be enough
- medium to large app: React Query or SWR usually pays off quickly

## 7. Practical checklist

Before you ship an async feature in React, verify:

- loading, error, and success are all rendered
- requests are canceled on unmount or dependency change
- stale responses cannot overwrite latest state
- independent requests are parallelized
- duplicated logic is extracted into a hook or a data library

If this checklist is green, your async UI is usually in a solid state.

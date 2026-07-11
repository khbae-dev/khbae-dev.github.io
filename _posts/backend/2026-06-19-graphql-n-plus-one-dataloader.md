---
title: GraphQL 화면 하나에 쿼리 수백 번, 구조적 N+1과 DataLoader
date: 2026-06-19 19:21:00 +09:00
categories: [Backend]
tags: [graphql, dataloader, n-plus-one, performance]
description: 리졸버가 필드 단위로 독립 실행되는 GraphQL의 구조적 N+1 문제를 이해하고, DataLoader 배칭으로 해결하는 방법을 정리했다.
---

ORM의 N+1은 eager loading으로 잡는다고 알고 있었는데, GraphQL에서는 그걸로 안 되는 이유를 공부했습니다.

## 리졸버는 서로를 모른다

GraphQL 서버는 쿼리 트리를 따라 리졸버를 필드마다 독립적으로 실행합니다.

```graphql
{
  posts(first: 100) {      # 쿼리 1번
    author { name }        # 게시글마다 author 리졸버 → 100번
    comments { body }      # 게시글마다 comments 리졸버 → 100번
  }
}
```

posts 리졸버가 아무리 JOIN을 잘해도, author 리졸버는 "게시글 하나"의 맥락만 알고 실행됩니다. 화면 하나에 DB 쿼리 수백 번이 나가는 구조적 N+1입니다. REST처럼 엔드포인트 단위로 eager loading을 걸 지점 자체가 없습니다.

## DataLoader: 같은 틱의 요청을 모아서 한 방에

DataLoader는 한 이벤트 루프 틱 동안 호출된 `load(key)`들을 모아 배치 함수 한 번으로 처리하고, 결과를 각 호출자에게 분배합니다. 중복 key 캐싱도 됩니다.

```javascript
const authorLoader = new DataLoader(async (userIds) => {
  const rows = await db.query(
    "SELECT * FROM users WHERE id = ANY($1)", [userIds]
  );
  const byId = new Map(rows.map(r => [r.id, r]));
  return userIds.map(id => byId.get(id)); // 순서 보장 필수
});

// 리졸버
Post: {
  author: (post, _, ctx) => ctx.authorLoader.load(post.authorId),
}
```

author 조회 100번이 `WHERE id = ANY(...)` 1번으로 합쳐집니다. 주의점은 두 가지입니다. 배치 함수는 요청받은 key 순서대로 결과를 반환해야 하고, 로더 인스턴스는 요청(request) 스코프로 만들어 요청 간 캐시 오염을 막아야 합니다.

## 정리

- GraphQL의 N+1은 코드 실수가 아니라 리졸버 모델의 구조적 결과다.
- 연관 엔티티 리졸버에는 DataLoader를 기본으로 깐다.
- 배치 함수의 순서 보장과 request 스코프 생성, 이 둘이 흔한 함정이다.

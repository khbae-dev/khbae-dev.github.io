---
title: 글 50개 조회에 쿼리 51번, ORM N+1과 eager loading
date: 2026-06-04 02:16:00 +09:00
categories: [Backend]
tags: [orm, n-plus-one, database, performance]
description: 반복문 안의 연관 데이터 접근이 쿼리를 폭발시키는 N+1 문제를 이해하고, eager loading으로 JOIN 한 번에 해결하는 방법을 정리했다.
---

ORM을 쓸 때 가장 먼저 밟는 지뢰, N+1 문제를 기본부터 정리했습니다.

## 코드는 한 줄인데 쿼리는 51번

```javascript
const posts = await Post.findAll({ limit: 50 });   // 쿼리 1번
for (const post of posts) {
  console.log(post.author.name);  // lazy loading → 각각 쿼리 1번, 총 50번
}
```

ORM의 lazy loading은 연관 데이터를 "접근하는 순간" 조회합니다. 편리하지만 반복문과 만나면 목록 1번 + 항목 수 N번, 합계 N+1번의 쿼리가 나갑니다. 항목이 50개면 51번, 페이지가 무거워질수록 정비례로 늘어납니다. 쿼리 하나하나는 빠르니 APM 없이는 눈치채기도 어렵습니다.

## eager loading: 처음부터 함께 가져오기

필요한 연관 관계를 조회 시점에 선언하면 ORM이 JOIN(또는 IN 배치 조회)으로 한 번에 가져옵니다.

```javascript
// Sequelize
const posts = await Post.findAll({
  limit: 50,
  include: [{ model: User, as: "author" }],  // JOIN 1번으로 끝
});
```

```python
# Django
posts = Post.objects.select_related("author")[:50]        # FK → JOIN
posts = Post.objects.prefetch_related("comments")[:50]    # 역방향/N:M → IN 조회
```

패턴을 하나 기억해 두면 좋습니다. FK로 단일 객체를 따라가는 관계는 JOIN 계열(select_related/include), 컬렉션을 따라가는 관계는 별도 IN 쿼리 계열(prefetch_related)이 적합합니다. 그리고 개발 중 쿼리 로그를 켜두면 N+1은 로그에서 같은 모양 쿼리가 도배되는 걸로 바로 드러납니다.

## 정리

- 반복문 안에서 연관 속성에 접근하는 코드는 N+1의 시그니처다.
- 조회 시점에 무엇이 필요한지 선언(eager loading)하면 쿼리 1~2번으로 끝난다.
- 개발 환경에서 쿼리 로그(또는 쿼리 카운터)를 켜두는 게 최고의 예방책이다.

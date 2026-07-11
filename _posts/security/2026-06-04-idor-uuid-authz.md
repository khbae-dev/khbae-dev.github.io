---
title: URL의 숫자를 하나 바꾸면 남의 데이터가 열리는 IDOR
date: 2026-06-04 19:33:00 +09:00
categories: [Security]
tags: [security, idor, authorization, api]
description: 객체 접근 시 소유자 검증을 빠뜨려 순번 ID만 바꾸면 타인 데이터가 노출되는 IDOR 취약점과 권한 검사·UUID 방어를 정리했다.
---

가장 흔하면서도 피해가 큰 웹 취약점 IDOR(Insecure Direct Object Reference)를 공부했습니다. 대규모 개인정보 유출 사고의 단골 원인입니다.

## 번호만 바꾸면 열리는 문

리소스를 순번 ID로 조회하는 API에서, 서버가 "그 리소스가 요청자의 것인지"를 확인하지 않으면 문제가 생깁니다.

```javascript
// 취약: 로그인만 확인하고 소유권은 확인 안 함
app.get("/api/applications/:id", auth, async (req, res) => {
  const app = await Application.findById(req.params.id);
  res.json(app);   // 남의 지원서도 그대로 반환
});
```

`/api/applications/1002`를 `/api/applications/1001`로 바꾸면 남의 지원서가 열립니다. 순번이라 다음 번호를 추측하기도 쉽습니다. 자동화하면 전체 데이터를 순회 수집할 수 있어, 실제로 수천만 건 규모의 노출로 이어진 사고들이 있었습니다.

## 방어: 소유권 검증 + 추측 불가 ID

```javascript
// 1. 객체 단위 권한 검사 — 주인이 아니면 403
app.get("/api/applications/:id", auth, async (req, res) => {
  const app = await Application.findOne({
    _id: req.params.id,
    ownerId: req.user.id,       // 소유자 조건을 쿼리에 포함
  });
  if (!app) return res.sendStatus(404);   // 존재 여부도 숨긴다
  res.json(app);
});
```

핵심은 인증(누구인지)과 인가(이걸 볼 권한이 있는지)를 구분하는 겁니다. 로그인 확인만으로는 부족하고, 객체마다 소유자·권한을 검사해야 합니다. 여기에 순번 대신 UUID를 쓰면 다음 값을 추측할 수 없어 방어가 한 겹 더해집니다.

```
/api/applications/1002                              ← 추측 쉬움
/api/applications/9f1c2a3e-7b4d-4c8a-a1f2-...        ← 추측 불가
```

다만 UUID는 심층 방어일 뿐, 권한 검사를 대체하지 못합니다. UUID가 로그·리퍼러로 유출되면 여전히 접근되므로, 권한 검사가 언제나 1차 방어선입니다. 없는 리소스와 권한 없는 리소스를 모두 404로 응답해 존재 여부 자체를 숨기는 것도 좋은 습관입니다.

## 정리

- IDOR의 본질은 "인증은 했는데 인가를 빠뜨린" 것이다.
- 모든 객체 접근에 소유자·권한 검사를 넣는 게 1차이자 핵심 방어다.
- UUID는 추측을 막는 보조 수단이지 권한 검사의 대체재가 아니다.

---

*보안 취약점은 방어 목적으로만 다룹니다.*

---
title: 회원가입 요청에 isAdmin:true 한 줄로 관리자가 되는 Mass Assignment
date: 2026-06-21 19:20:00 +09:00
categories: [Security]
tags: [security, mass-assignment, authorization, api]
description: 요청 바디를 객체에 통째로 바인딩하면 숨은 권한 필드까지 덮어써지는 Mass Assignment 취약점과 화이트리스트 바인딩 방어를 정리했다.
---

요청 데이터를 편하게 처리하려는 코드가 권한 상승 취약점이 되는 Mass Assignment를 공부했습니다.

## 통째 바인딩의 위험

ORM·프레임워크는 요청 바디를 모델 객체에 한 번에 매핑하는 편의 기능을 제공합니다.

```javascript
// 위험: 요청 바디를 통째로 모델에 바인딩
const user = await User.create(req.body);
```

클라이언트가 폼에 없는 필드를 요청에 끼워 넣으면 그대로 반영됩니다.

```
POST /signup
{ "email": "a@b.com", "password": "...", "isAdmin": true, "credits": 999999 }
```

`isAdmin`, `role`, `credits`처럼 서버가 통제해야 할 필드까지 사용자가 덮어쓸 수 있습니다. 회원가입 한 번에 관리자가 되거나 잔액을 조작하는 겁니다. 개발자가 UI에 그 입력을 안 만들었다는 건 방어가 아닙니다. 요청은 UI를 거치지 않고 직접 보낼 수 있으니까요.

## 방어: 받을 필드를 명시적으로 고른다

```javascript
// 화이트리스트: 허용된 필드만 추출
const { email, password } = req.body;
const user = await User.create({
  email,
  password: await hash(password),
  role: "user",          // 권한은 서버가 고정
});
```

원칙은 "블랙리스트가 아니라 화이트리스트"입니다. 위험한 필드를 빼는 방식(블랙리스트)은 새 필드가 추가될 때마다 구멍이 생깁니다. 허용할 필드만 명시적으로 받는 방식이라야 안전합니다. 프레임워크별로 이를 지원하는 장치가 있습니다. Rails의 strong parameters(`permit`), NestJS의 DTO + `whitelist: true` ValidationPipe, Django의 `fields` 명시 등입니다. 그리고 권한·소유자·잔액 같은 민감 필드는 절대 요청에서 받지 말고 서버 로직으로만 설정합니다.

## 정리

- 요청 바디 통째 바인딩은 숨은 필드까지 열어주는 Mass Assignment 통로다.
- 허용 필드 화이트리스트로 받고, 권한류는 서버가 고정한다.
- "UI에 없으니 안전"은 착각이다. 요청은 UI를 우회할 수 있다.

---

*보안 취약점은 방어 목적으로만 다룹니다.*

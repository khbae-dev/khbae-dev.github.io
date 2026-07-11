---
title: CORS 설정 한 줄이 세션을 통째로 넘기는 경우, Origin 반사
date: 2026-07-06 19:22:00 +09:00
categories: [Security]
tags: [security, cors, csrf, session]
description: 요청 Origin을 검증 없이 그대로 반사하면서 credentials까지 허용하면 임의 사이트가 세션을 탈취할 수 있는 취약점과 화이트리스트 검증을 정리했다.
---

CORS를 "귀찮아서 다 열어둔" 설정이 세션 탈취 통로가 되는 경우를 공부했습니다. 편의 설정 하나가 인증을 무력화하는 사례입니다.

## 반사 + credentials의 위험한 조합

CORS 에러를 없애려고 요청의 Origin을 그대로 응답에 되돌려주는(reflect) 코드를 종종 봅니다.

```javascript
// 위험: 어떤 Origin이 와도 그대로 허용
res.setHeader("Access-Control-Allow-Origin", req.headers.origin);
res.setHeader("Access-Control-Allow-Credentials", "true");
```

두 헤더가 함께 있으면 문제가 심각해집니다. `Allow-Credentials: true`는 브라우저가 쿠키를 실은 교차 출처 요청을 허용하고 그 응답을 읽게 해줍니다. Origin을 무검증 반사하면 공격자 사이트(`evil.com`)에서 보낸 요청도 허용됩니다. 피해자가 로그인된 상태로 악성 페이지를 방문하면, 그 페이지의 스크립트가 피해자 쿠키로 API를 호출하고 응답(개인정보, 토큰)을 읽어 유출할 수 있습니다.

## 화이트리스트 검증 후에만 반사

```javascript
const ALLOWED = new Set([
  "https://app.example.com",
  "https://admin.example.com",
]);

const origin = req.headers.origin;
if (ALLOWED.has(origin)) {
  res.setHeader("Access-Control-Allow-Origin", origin);  // 검증된 것만 반사
  res.setHeader("Access-Control-Allow-Credentials", "true");
  res.setHeader("Vary", "Origin");   // 캐시 오염 방지
}
```

주의점 몇 가지가 있습니다. 와일드카드 `Access-Control-Allow-Origin: *`는 credentials와 함께 쓸 수 없습니다(브라우저가 거부). 그리고 `startsWith("https://example.com")` 같은 부분 일치 검증은 `https://example.com.evil.com`에 뚫리므로, 반드시 정확한 문자열 집합 비교여야 합니다. `Vary: Origin`은 CDN이 한 Origin의 응답을 다른 Origin에 캐시하는 오염을 막습니다.

## 정리

- Origin 무검증 반사 + Allow-Credentials는 교차 출처 세션 탈취의 문을 연다.
- 허용 목록은 정확 일치(Set) 비교로, 부분 일치·정규식 매칭은 우회당한다.
- Vary: Origin까지 넣어야 캐시 계층의 오염도 막힌다.

---

*보안 취약점은 방어 목적으로만 다룹니다.*

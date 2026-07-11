---
title: API 한 번 호출에 요청이 두 번 나가는 이유, CORS 프리플라이트 캐시
date: 2026-06-13 16:02:00 +09:00
categories: [Backend]
tags: [cors, http, preflight, performance]
description: 네트워크 탭에 OPTIONS 요청이 매번 함께 찍히는 원인과 Access-Control-Max-Age 한 줄로 프리플라이트를 캐시해 왕복을 절반으로 줄이는 방법을 정리했다.
---

네트워크 탭을 보다가 API 호출 하나에 요청이 두 개씩 찍히는 걸 파고들었습니다. 앞에 붙는 건 브라우저가 보내는 CORS 프리플라이트(OPTIONS)였습니다.

## 프리플라이트가 매번 나가는 조건

교차 출처 요청 중 "단순 요청"이 아닌 경우 — `Content-Type: application/json`, `Authorization` 헤더, PUT/DELETE 메서드 등 — 브라우저는 본 요청 전에 OPTIONS로 서버 허락을 확인합니다. 문제는 서버 응답에 `Access-Control-Max-Age`가 없으면 이 확인을 캐시하지 않고 매 호출마다 반복한다는 점입니다. 모든 API 호출의 왕복(latency)이 2배가 되는 셈입니다.

## 해결은 응답 헤더 한 줄

```
Access-Control-Max-Age: 7200
```

Express 예시로는 이렇게 씁니다.

```javascript
app.use(cors({
  origin: "https://app.example.com",
  maxAge: 7200,   // 프리플라이트 결과를 2시간 캐시
}));
```

이후 같은 조건의 요청은 프리플라이트를 건너뛰고 본 요청만 나갑니다. 브라우저별 상한이 있어서 Chrome은 최대 7200초(2시간), Firefox는 최대 24시간까지 캐시합니다. 그 이상을 지정해도 상한으로 잘립니다.

주의할 점 하나. 캐시는 "URL + 요청 헤더 조합" 단위라서, 허용 헤더나 메서드 구성이 바뀌면 캐시가 무력화됩니다. 배포 직후 프리플라이트가 다시 몰리는 건 자연스러운 현상입니다.

## 정리

- OPTIONS가 매번 보인다면 Access-Control-Max-Age 부재를 의심한다.
- 헤더 한 줄로 API 체감 지연이 눈에 띄게 줄어든다. 특히 고지연 환경(모바일)에서 효과가 크다.
- 상한은 브라우저마다 다르며 Chrome 기준 7200초가 실질 최대치다.

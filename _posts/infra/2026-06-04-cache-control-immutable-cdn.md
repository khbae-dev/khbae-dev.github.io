---
title: origin 트래픽 대부분이 낭비일 때, Cache-Control 한 줄
date: 2026-06-04 02:35:00 +09:00
categories: [Infra]
tags: [cdn, cache-control, http, performance]
description: 정적 자산에 캐시 헤더가 없어 CDN이 매번 origin으로 되묻는 낭비를, max-age와 immutable로 없애는 방법을 정리했다.
---

CDN을 붙였는데도 origin 서버 트래픽이 여전히 높은 경우를 공부했습니다. 자산 자체가 "캐시해도 된다"고 말하지 않은 게 문제였습니다.

## 캐시 헤더가 없으면 CDN도 조심스럽다

Cache-Control 헤더가 없거나 짧으면, CDN 엣지는 자산의 신선도를 확신하지 못해 자주 origin에 되묻습니다(revalidation). 정적 자산(JS, CSS, 폰트, 이미지)은 사실상 안 바뀌는데도 매번 origin 왕복이 생기는 겁니다. 엣지 캐시의 존재 이유가 반감됩니다.

## 지문 붙은 자산은 영구 캐시

현대 빌드 도구는 파일명에 콘텐츠 해시를 박습니다(`app.9f3a1c.js`). 내용이 바뀌면 파일명이 바뀌므로, 같은 파일명은 영원히 같은 내용이라고 단언할 수 있습니다. 그러니 최대로 캐시하면 됩니다.

```nginx
# 해시가 박힌 정적 자산: 1년 + immutable
location ~* \.[0-9a-f]{8}\.(js|css|woff2|png|avif)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}

# HTML: 항상 최신 확인 (여기가 새 자산 파일명을 가리킨다)
location ~* \.html$ {
  add_header Cache-Control "no-cache";
}
```

`immutable`이 핵심입니다. 이게 없으면 브라우저는 사용자가 새로고침할 때 "혹시 바뀌었나" 조건부 요청(If-None-Match)을 보내는데, immutable은 그마저도 생략시킵니다. max-age가 남아 있는 동안은 네트워크를 아예 건드리지 않습니다.

전략의 뼈대는 이렇습니다. HTML은 캐시하지 않아 항상 최신 자산 목록을 받고, 그 HTML이 가리키는 해시 자산들은 영구 캐시. 배포로 내용이 바뀌면 파일명(해시)이 바뀌니 캐시 무효화를 따로 할 필요도 없습니다.

## 정리

- 정적 자산의 origin 왕복은 대부분 캐시 헤더 부재/부실 때문이다.
- 콘텐츠 해시가 박힌 자산은 max-age=1년 + immutable로 못 박는다.
- HTML은 no-cache, 자산은 영구 캐시 — 이 조합이 무효화 없이도 안전한 이유다.

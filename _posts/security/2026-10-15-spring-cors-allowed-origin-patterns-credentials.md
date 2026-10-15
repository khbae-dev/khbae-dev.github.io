---
title: Spring의 CORS 안전장치를 비껴가는 설정, allowedOriginPatterns("*")
date: 2026-10-15 19:16:00 +09:00
categories: [Security]
tags: [security, cors, spring, csrf, cookie]
description: Spring이 allowCredentials와 allowedOrigins("*") 조합은 막지만 allowedOriginPatterns("*")는 검사하지 않아 Origin 반사가 그대로 열리는 문제와, CORS 정책을 한 곳으로 모아 기동 단계에서 막는 방법을 정리했다.
---

[지난 글](/posts/cors-origin-reflection/)에서 Origin 반사와 credentials 허용의 조합이 위험하다는 이야기를 했습니다. 이번에는 Spring이 그 조합을 막아 주는 줄 알았는데 설정 방식 하나 때문에 그대로 열려 있던 사례입니다.

## Spring은 이 조합을 막는다, 단 한쪽만

Spring은 아래 설정을 거부합니다. 요청을 처리할 때 `IllegalArgumentException`이 납니다.

```java
config.setAllowCredentials(true);
config.addAllowedOrigin("*");   // 거부된다
```

그런데 같은 의도를 패턴으로 쓰면 통과합니다.

```java
config.setAllowCredentials(true);
config.addAllowedOriginPattern("*");   // 통과한다
```

`CorsConfiguration.validateAllowCredentials()`는 `allowedOrigins`만 검사하고 `allowedOriginPatterns`는 검사하지 않습니다. 그리고 패턴 방식은 응답의 `Access-Control-Allow-Origin`에 리터럴 `*`가 아니라 **요청의 Origin을 그대로 반사**합니다. 결과적으로 모든 사이트에 자격증명 포함 응답을 허용하게 됩니다.

## 실제로 악용 가능한 경로였다

설정 실수 하나로 끝나지 않고 세 가지가 맞물려 있었습니다.

1. 인증 토큰 추출기가 `Authorization` 헤더가 없으면 **쿠키로 폴백**했고, 프런트엔드는 쿠키 인증을 쓰고 있었다.
2. 위의 패턴 설정으로 모든 Origin이 반사되고 credentials가 허용됐다.
3. CSRF 보호가 꺼져 있어 쓰기 요청도 막히지 않았다.

공격자 페이지에서 아래 한 줄이면 피해자 권한으로 읽기와 쓰기가 가능했습니다.

```js
fetch('https://api.example.com/api/me', { credentials: 'include' })
```

## 고친 방법: 정책을 한 곳에 모으고 기동에서 막기

CORS 설정이 두 군데(인증 서버 설정, 리소스 서버 설정)로 갈라져 있었기 때문에, 한쪽만 고치면 다른 쪽이 남습니다. 그래서 CORS 정책을 결정하는 객체를 하나로 두고 양쪽이 그것을 쓰게 했습니다.

```java
public CorsConfiguration build(List<String> allowedOriginPatterns) {
    CorsConfiguration c = new CorsConfiguration();
    if (allowedOriginPatterns.contains("*")) {
        // 위험한 조합이 설정으로 되살아나지 못하게 기동을 실패시킨다
        throw new IllegalStateException("자격증명을 쓰는 CORS에 '*' 출처는 허용하지 않습니다.");
    }
    if (allowedOriginPatterns.isEmpty()) {
        c.addAllowedOriginPattern("*");
        c.setAllowCredentials(false);   // 출처는 열되 자격증명은 막는다
    } else {
        allowedOriginPatterns.forEach(c::addAllowedOriginPattern);
        c.setAllowCredentials(true);    // 명시한 출처에만 자격증명 허용
    }
    return c;
}
```

기본값을 "출처는 열고 자격증명만 차단"으로 둔 데는 이유가 있습니다. 리버스 프록시가 HTTPS를 종료하는 환경에서는 같은 사이트 요청도 scheme이 달라 교차 출처로 판정될 수 있습니다. 출처를 한 번에 좁히면 정상 요청이 403으로 깨질 수 있으므로, 출처 목록은 배포 환경에서 명시적으로 고르게 했습니다.

## 테스트는 공격자 입장에서

운영 코드가 등록하는 필터를 그대로 꺼내, 공격자 Origin으로 요청을 흘려보내고 응답 헤더를 확인하는 테스트를 만들었습니다. 그리고 자격증명 허용을 일부러 되살려 테스트가 실제로 실패하는지도 확인했습니다. 실패하지 않는 보안 테스트는 아무것도 지키지 않습니다.

## 정리

- `allowedOriginPatterns("*")`는 Origin을 반사하고, Spring의 credentials 검사도 비껴간다.
- 쿠키 폴백 인증, Origin 반사, CSRF 비활성이 겹치면 다른 사이트에서 피해자 권한으로 API를 호출할 수 있다.
- CORS 정책은 한 곳에서 결정하고, 위험한 조합은 런타임이 아니라 기동 단계에서 막는다.
- CORS는 CSRF 방어가 아니다. 쿠키 인증을 쓴다면 CSRF 대책은 따로 둔다.

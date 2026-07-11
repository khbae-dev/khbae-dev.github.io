---
title: CDN을 깔았는데 적중률 0%, 캐시 키에 뭘 넣었는지 보기
date: 2026-06-25 19:21:00 +09:00
categories: [Infra]
tags: [cdn, cloudfront, cache, performance]
description: 쿠키·쿼리스트링·헤더가 캐시 키에 전부 포함돼 요청마다 키가 달라지는 문제와 CloudFront 캐시 정책 최소화를 정리했다.
---

CloudFront를 붙였는데 origin 부하가 그대로인 사례를 공부했습니다. `X-Cache: Miss from cloudfront`만 가득한 상황, 원인은 캐시 키의 과잉 포함이었습니다.

## 캐시 키에 들어간 것 = 캐시를 쪼개는 축

CDN은 캐시 키가 완전히 같은 요청에만 캐시를 돌려줍니다. 키에 쿠키를 포함하면 세션 쿠키가 다른 사용자마다 다른 캐시가 되고, 전체 쿼리스트링을 포함하면 utm_source 하나가 달라도 미스, User-Agent를 포함하면 브라우저·버전 조합마다 별도 캐시가 됩니다. 정적 자산에 이 축들이 걸려 있으면 사실상 캐시가 무력화됩니다.

## 캐시 정책: 필요한 것만 키에 남긴다

CloudFront 캐시 정책 기준으로, 정적 자산은 이렇게 최소화합니다.

```yaml
# 정적 자산용 캐시 정책
CachePolicy:
  Name: static-assets
  ParametersInCacheKeyAndForwardedToOrigin:
    CookiesConfig:      { CookieBehavior: none }       # 쿠키 제외
    HeadersConfig:      { HeaderBehavior: none }        # UA 등 제외
    QueryStringsConfig: { QueryStringBehavior: whitelist,
                          QueryStrings: [v] }           # 버전 파라미터만
    EnableAcceptEncodingBrotli: true                    # 압축 변형은 별도 처리
    EnableAcceptEncodingGzip: true
```

판단 기준은 한 문장입니다. "이 값이 응답 내용을 실제로 바꾸는가?" 바꾸지 않으면 키에서 뺍니다. 응답에 영향은 없지만 origin에는 전달해야 하는 값(분석용 쿼리 등)은 origin request policy로 분리해 "전달은 하되 키에는 안 넣는" 구성이 가능합니다.

동적 API와 정적 자산의 behavior를 분리하는 것도 기본입니다. `/static/*`, `/assets/*`는 공격적으로 캐시하고, API 경로는 별도 정책으로 관리합니다. 적중률은 CloudFront 지표의 Cache Hit Rate로 검증합니다.

## 정리

- 캐시 미스의 주범은 TTL이 아니라 과잉 포함된 캐시 키인 경우가 많다.
- "응답을 바꾸는 값만 키에" — 쿠키·UA·전체 쿼리는 정적 자산 키에서 제거한다.
- 전달 필요와 키 포함을 분리(origin request policy)하면 둘 다 만족할 수 있다.

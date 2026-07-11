---
title: 코드 한 줄 안 고치고 응답 크기 70% 줄이기, brotli 압축
date: 2026-06-04 02:41:00 +09:00
categories: [Backend]
tags: [http, brotli, compression, nginx]
description: 텍스트 응답을 무압축으로 보내는 낭비를 서버 brotli 설정만으로 줄이는 방법과 gzip 대비 특성, 주의점을 정리했다.
---

성능 개선 중 가성비가 가장 좋은 축에 드는 설정을 정리했습니다. 애플리케이션 코드를 전혀 건드리지 않는 응답 압축입니다.

## 텍스트는 놀라울 만큼 잘 눌린다

JS 번들, JSON API 응답, HTML, CSS, SVG는 반복 패턴이 많은 텍스트라 압축 효율이 매우 높습니다. brotli는 gzip보다 압축률이 한 단계 더 좋아서(특히 텍스트에서 15~20% 추가 절감), 수백 KB짜리 JS 번들이 수십 KB로 내려가는 게 보통입니다. 브라우저는 요청에 `Accept-Encoding: gzip, deflate, br`을 보내고 있으므로, 서버가 `Content-Encoding: br`로 응답만 하면 됩니다.

## nginx 설정 예시

```nginx
# 정적 자산: 빌드 타임에 미리 압축(.br 파일)해 두고 서빙하는 게 최선
brotli_static on;

# 동적 응답: 실시간 압축
brotli on;
brotli_comp_level 5;          # 동적은 4~6 권장 (11은 정적 전용)
brotli_types text/plain text/css application/json
             application/javascript image/svg+xml;
```

Node 단독이라면 `compression` 계열 미들웨어나 리버스 프록시에 맡깁니다. 포인트 몇 가지를 기억해 둡니다.

- 압축 레벨 트레이드오프: 최고 레벨(11)은 CPU를 많이 먹어 동적 응답에 부적합. 정적 자산만 빌드 때 11로 미리 압축한다.
- 이미 압축된 포맷(jpg, png, webp, mp4)은 대상에서 제외. 효과가 없고 CPU만 쓴다.
- 아주 작은 응답(수백 B)은 압축 이득이 헤더 오버헤드보다 작을 수 있어 최소 크기 조건을 둔다.

## 정리

- 텍스트 응답 무압축은 순수한 낭비다. 확인부터 해보자 (응답 헤더에 Content-Encoding이 있는지).
- 정적 자산은 빌드 타임 사전 압축 + brotli_static, 동적 응답은 중간 레벨 실시간 압축.
- 전송량 감소는 로딩 속도와 트래픽 비용을 동시에 줄인다.

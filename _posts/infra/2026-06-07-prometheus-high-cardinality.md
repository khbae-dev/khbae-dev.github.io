---
title: 장애 때 모니터링이 먼저 죽는 이유, 고카디널리티 라벨
date: 2026-06-07 16:03:00 +09:00
categories: [Infra]
tags: [prometheus, cardinality, metrics, observability]
description: user_id 같은 무한 값 라벨이 시계열 수를 폭발시켜 Prometheus 메모리를 잠식하는 구조와 라벨 설계 원칙을 정리했다.
---

모니터링 시스템이 정작 장애 순간에 OOM으로 함께 죽는 아이러니를 공부했습니다. 원인은 대부분 라벨 설계에 있습니다.

## 라벨 조합 수 = 시계열 수

Prometheus에서 메트릭 하나는 "이름 + 라벨 값 조합"마다 독립된 시계열이 됩니다. 그리고 활성 시계열은 각각 메모리를 점유합니다.

```
http_requests_total{route="/api/orders", status="200"}        # 시계열 1개
http_requests_total{route="/api/orders", status="200",
                    user_id="8231942"}                          # 사용자마다 1개
```

`user_id`, `session_id`, `request_id`, 정규화 안 된 URL(경로에 ID 포함), 이메일 같은 무한 값 라벨을 하나 넣는 순간 시계열이 사용자 수만큼 생깁니다. 라벨 두 개면 곱으로 늘어납니다(cardinality explosion). 수백만 시계열이 되면 스크레이프·쿼리·컴팩션이 전부 느려지고, 트래픽이 몰리는 장애 시점에 시계열 생성이 가속되며 모니터링이 먼저 넘어집니다.

## 설계 원칙과 진단

라벨의 자격은 "유한하고 낮은 카디널리티"입니다. route(정규화된 패턴), method, status, region 같은 것들. 개별 사용자·요청 추적은 메트릭이 아니라 로그와 트레이스의 일입니다.

```promql
# 무엇이 폭발 중인지 진단
topk(10, count by (__name__)({__name__=~".+"}))
```

이미 폭발한 라벨은 수집 단계에서 잘라냅니다.

```yaml
metric_relabel_configs:
  - regex: "user_id"          # 라벨 자체를 드랍
    action: labeldrop
```

URL 경로는 계측 코드에서 `/users/8231942` → `/users/:id`로 정규화해서 뽑는 게 근본 해결입니다.

## 정리

- 라벨 하나 추가의 비용은 "그 라벨이 가질 수 있는 값의 수"만큼 곱해진다.
- 무한 값(ID류)은 메트릭 라벨 금지, 로그·트레이스로 보낸다.
- topk 진단 → relabel 응급처치 → 계측 코드 정규화 순으로 잡는다.

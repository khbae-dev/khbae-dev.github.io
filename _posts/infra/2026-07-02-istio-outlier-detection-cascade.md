---
title: 멀쩡한 파드가 전부 쫓겨나는 Istio 서킷브레이커 설정 실수
date: 2026-07-02 19:22:00 +09:00
categories: [Infra]
tags: [istio, outlier-detection, service-mesh, kubernetes]
description: outlier detection의 공격적인 임계값이 연쇄 축출을 일으켜 503을 폭증시키는 메커니즘과 안전한 설정 기준을 정리했다.
---

장애 격리를 위해 넣은 Istio outlier detection이 오히려 장애를 만드는 역설적인 사례를 공부했습니다.

## 보호 장치가 가해자가 되는 설정

outlier detection은 에러를 뱉는 호스트를 로드밸런싱 풀에서 일시 제외(ejection)하는 수동적 헬스체크입니다. 문제의 설정은 이런 조합입니다.

```yaml
outlierDetection:
  consecutive5xxErrors: 1        # 5xx 한 번이면 축출
  maxEjectionPercent: 100        # 전부 쫓아내도 됨
```

일시적인 이유(재배포 직후 워밍업, GC 순간)로 파드 하나가 5xx를 한 번 내면 즉시 축출됩니다. 그 트래픽이 남은 파드로 몰리며 부하가 올라가고, 다음 파드도 5xx를 내고 축출됩니다. `maxEjectionPercent: 100`이라 마지막 파드까지 쫓겨날 수 있고, 풀이 비면 모든 요청이 503 `no healthy upstream`이 됩니다. 연쇄 축출(cascading ejection)입니다.

## 안전한 기준선

```yaml
outlierDetection:
  consecutive5xxErrors: 5        # 노이즈에 관대하게
  interval: 10s
  baseEjectionTime: 30s
  maxEjectionPercent: 30         # 최소 70%는 항상 남긴다
  minHealthPercent: 50           # 이 밑으로는 축출 중단(패닉 모드)
```

- 임계값은 일시 오류를 흡수할 만큼 높게. 1은 사실상 금지 값입니다.
- maxEjectionPercent는 "최악의 경우에도 남길 용량"에서 역산합니다.
- 축출은 원인 치료가 아니라 증상 격리입니다. 5xx의 원인(느린 시작, 리소스 부족)은 startupProbe·리소스 조정으로 따로 잡아야 합니다.

배포 직후 5xx가 원인이라면 readinessProbe와 워밍업이 먼저입니다. 서킷브레이커는 그 다음 방어선입니다.

## 정리

- 축출 임계 1 + 축출 상한 100%는 연쇄 축출 레시피다.
- maxEjectionPercent로 "항상 남는 최소 용량"을 보장하는 게 핵심이다.
- outlier detection은 격리 장치일 뿐, 5xx의 원인은 따로 고쳐야 한다.

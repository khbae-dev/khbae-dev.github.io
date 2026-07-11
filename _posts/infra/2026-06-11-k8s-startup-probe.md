---
title: 배포마다 CrashLoopBackOff, liveness가 부팅을 죽일 때
date: 2026-06-11 16:02:00 +09:00
categories: [Infra]
tags: [kubernetes, probe, liveness, startup]
description: 부팅이 느린 앱을 liveness probe가 멈췄다고 오판해 재시작 루프에 빠뜨리는 문제와 startupProbe 분리를 정리했다.
---

새로 배포한 파드가 Running까지 못 가고 CrashLoopBackOff를 반복하는 문제의 단골 원인을 공부했습니다. 앱 버그가 아니라 프로브 설정이었습니다.

## liveness의 오판

liveness probe는 "멈춘 컨테이너를 재시작"하는 장치입니다. 문제는 부팅 중인 앱과 멈춘 앱을 구분하지 못한다는 점입니다. JVM 웜업, 마이그레이션, 캐시 프리로드로 기동에 90초가 걸리는 앱에 이런 설정이 있다면:

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
```

기동 40초 시점(10 + 10×3)에 kubelet이 "죽었다"고 판단해 컨테이너를 재시작합니다. 재시작해도 부팅은 또 90초가 필요하니 또 죽습니다. 영원히 못 뜨는 루프입니다. initialDelaySeconds를 크게 잡는 임시방편은 반대로 "진짜 행업"의 감지를 그만큼 늦춥니다.

## startupProbe: 부팅 전용 유예

쿠버네티스는 이 문제를 위해 프로브를 하나 더 뒀습니다. startupProbe가 성공할 때까지 liveness/readiness는 비활성입니다.

```yaml
startupProbe:
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 5
  failureThreshold: 60       # 최대 5초×60 = 300초까지 부팅 허용
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3        # 부팅 후에는 빡빡하게 감시
```

부팅 동안은 최대 300초까지 관대하게 기다리고, 일단 뜬 뒤에는 liveness가 짧은 주기로 행업을 감시합니다. "부팅의 관대함"과 "운영 중 민감함"을 분리하는 게 이 프로브의 존재 이유입니다. readiness는 별개로 "트래픽을 받을 준비"를 판단하며, liveness와 같은 엔드포인트를 쓰더라도 실패의 의미(재시작 vs 트래픽 제외)가 다르다는 것도 함께 기억해 둡니다.

## 정리

- 배포 직후의 CrashLoopBackOff는 앱보다 프로브 설정을 먼저 본다.
- 부팅이 느린 앱은 startupProbe로 유예를 주고, liveness는 운영 중 감시에 집중시킨다.
- initialDelaySeconds 늘리기는 미봉책, 프로브 역할 분리가 정석이다.

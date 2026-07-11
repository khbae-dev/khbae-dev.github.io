---
title: 파드 로그 하나가 노드 전체를 쓰러뜨리는 경로, 로그 회전과 DiskPressure
date: 2026-06-21 19:22:00 +09:00
categories: [Infra]
tags: [kubernetes, logging, diskpressure, ephemeral-storage]
description: 무한정 쌓이는 컨테이너 로그가 노드 디스크를 채워 DiskPressure 축출을 일으키는 문제와 로그 회전·ephemeral-storage limit 격리를 정리했다.
---

파드 하나의 폭주 로그가 그 노드의 무고한 파드까지 전부 쫓아내는 장애 경로를 공부했습니다.

## 한 파드의 로그가 공유 자원을 먹는다

컨테이너의 stdout/stderr 로그는 노드의 디스크(/var/log/containers 경유)에 쌓입니다. 애플리케이션이 에러 루프에 빠져 초당 수천 줄을 찍으면 로그 파일이 시간당 수 GB씩 자랍니다. 노드 디스크 사용률이 임계(기본 85%)를 넘으면 kubelet이 DiskPressure 상태를 선언하고, 디스크를 확보하려고 그 노드의 파드들을 축출(evict)하기 시작합니다. 문제 파드만이 아니라 같은 노드의 정상 파드까지 피해를 봅니다.

## 이중 방어: 회전 + 격리

**1. 로그 회전 상한.** kubelet 설정으로 컨테이너당 로그 보관량에 상한을 겁니다.

```yaml
# kubelet 설정
containerLogMaxSize: 10Mi
containerLogMaxFiles: 5      # 컨테이너당 최대 50Mi
```

**2. 파드별 ephemeral-storage limit.** 파드가 쓰는 로컬 디스크(로그, emptyDir, 쓰기 레이어)에 개별 상한을 둡니다.

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "4Gi"   # 초과 시 이 파드만 축출된다
```

limit을 넘으면 "그 파드만" 축출됩니다. 노드 전체가 DiskPressure에 빠지기 전에 원인 파드가 격리되는 구조라, 폭주의 피해 반경이 파드 하나로 좁혀집니다.

운영적으로는 로그를 수집기(Fluent Bit 등)로 중앙화하고, 노드 디스크 사용률과 파드별 ephemeral-storage 사용량에 알람을 걸어두는 것까지가 세트입니다. 애플리케이션 쪽 로그 레벨 폭주(DEBUG로 배포 등)도 같은 사고의 단골 원인이라 배포 체크리스트에 넣어둘 만합니다.

## 정리

- 컨테이너 로그는 무료가 아니다. 노드 디스크라는 공유 자원을 쓴다.
- 로그 회전 상한은 노드를 지키고, ephemeral-storage limit은 피해를 파드 하나로 격리한다.
- DiskPressure 축출은 무차별적이다. 사전 상한 없이 운영하는 건 도박이다.

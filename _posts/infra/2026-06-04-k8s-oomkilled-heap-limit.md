---
title: 메모리 limit을 올려도 exit 137이 반복될 때, 힙 상한 정렬
date: 2026-06-04 19:35:00 +09:00
categories: [Infra]
tags: [kubernetes, oomkilled, jvm, nodejs]
description: 런타임 힙이 컨테이너 limit을 인지하지 못해 OOMKilled가 반복되는 구조와 requests/limits, 힙 상한을 함께 맞추는 방법을 정리했다.
---

파드가 exit code 137(OOMKilled)로 죽는데 limit을 올려도 시간만 벌 뿐 또 죽는 문제를 공부했습니다. 원인은 컨테이너 limit과 런타임 힙 설정의 불일치였습니다.

## 두 세계의 단절

컨테이너 limit은 cgroup이 강제하는 물리적 상한입니다. 프로세스 전체(힙 + 스택 + 네이티브 메모리 + 버퍼)가 이 선을 넘는 순간 커널 OOM killer가 SIGKILL(137)을 날립니다. 경고도 GC 기회도 없습니다.

한편 런타임의 힙은 자기 설정대로 자랍니다. 구형 JVM이나 기본 설정의 Node는 "호스트 전체 메모리"를 기준으로 힙 상한을 잡기도 합니다. limit이 512Mi인 컨테이너에서 힙이 1GB까지 자라려 들면, GC가 일어나기 전에 cgroup 상한에 먼저 부딪혀 죽습니다. limit을 올려도 힙 상한이 여전히 더 크면 같은 일이 반복됩니다.

## 힙 상한을 limit 아래로

원칙은 "힙 상한 + 비힙 오버헤드 < limit"입니다. 비힙(스택, 네이티브, 코드 캐시)로 20~30%는 남겨야 합니다.

```yaml
resources:
  requests: { memory: "512Mi" }   # 스케줄링 기준
  limits:   { memory: "512Mi" }   # OOM 기준 (메모리는 requests=limits 권장)
```

```dockerfile
# Node: 힙 상한을 limit의 ~75%로
CMD ["node", "--max-old-space-size=384", "server.js"]
```

```bash
# JVM: 컨테이너 인지 + 비율 기반
JAVA_OPTS="-XX:MaxRAMPercentage=75.0"   # 최신 JVM은 cgroup을 읽는다
```

이렇게 정렬하면 메모리가 부족할 때 "커널의 즉살" 대신 "런타임의 GC(그리고 정 안 되면 런타임 레벨 OOM 에러)"가 먼저 옵니다. 원인 분석이 가능한 죽음으로 바뀌는 겁니다.

## 정리

- exit 137의 반복은 limit 크기 문제가 아니라 힙-limit 불일치 문제일 때가 많다.
- 힙 상한을 limit의 70~80%로 명시하고, JVM은 컨테이너 인지 옵션을 확인한다.
- 메모리는 requests=limits로 잡아 노드 압박에 의한 변수까지 줄인다.

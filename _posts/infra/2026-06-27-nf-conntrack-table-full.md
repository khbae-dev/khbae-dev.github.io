---
title: 트래픽은 평소인데 패킷이 랜덤 드롭될 때, conntrack 테이블 포화
date: 2026-06-27 19:24:00 +09:00
categories: [Infra]
tags: [linux, conntrack, network, kernel]
description: nf_conntrack 테이블이 죽은 연결로 가득 차 새 연결이 조용히 버려지는 문제와 테이블 크기·타임아웃 튜닝을 정리했다.
---

로드밸런서 뒤 노드에서 간헐적으로 연결이 실패하는데 애플리케이션 로그엔 아무것도 없는 장애 유형을 공부했습니다. 커널 레벨의 conntrack 테이블이 원인이었습니다.

## 커널이 연결마다 장부를 쓴다

iptables/NAT를 쓰는 리눅스는 지나가는 연결마다 nf_conntrack 테이블에 항목을 기록합니다. 이 테이블(`nf_conntrack_max`)이 가득 차면 새 연결의 패킷을 커널이 그냥 버립니다. 애플리케이션은 알 수도 없습니다. dmesg에 `nf_conntrack: table full, dropping packet`이 찍히는 게 유일한 단서입니다.

트래픽이 평소 수준인데 가득 차는 이유는 타임아웃입니다. established 상태 항목의 기본 타임아웃이 5일(432,000초)이라, 비정상 종료된 연결(클라이언트 실종, FIN 유실)의 장부가 닷새 동안 자리를 차지합니다. 연결 수명이 짧고 회전이 빠른 서비스일수록 좀비 항목이 빨리 쌓입니다.

## 진단과 튜닝

```bash
# 현재 사용량 / 최대치
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# dmesg에서 드롭 확인
dmesg | grep conntrack
```

```bash
# /etc/sysctl.d/99-conntrack.conf
net.netfilter.nf_conntrack_max = 1048576              # 메모리 여유에 맞게 상향
net.netfilter.nf_conntrack_tcp_timeout_established = 86400   # 5일 → 1일
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
```

max를 올릴 땐 해시 버킷(`nf_conntrack_buckets`)도 비례해서 올려야 조회 성능이 유지됩니다. 그리고 사용률(count/max)을 모니터링에 넣어 80% 알람을 걸어두면 재발을 사전에 잡습니다. NAT가 필요 없는 고트래픽 경로는 NOTRACK 룰로 장부 자체를 면제하는 방법도 있습니다.

## 정리

- "로그 없는 랜덤 연결 실패"는 커널 conntrack 포화를 의심하고 dmesg부터 본다.
- established 5일 기본값이 좀비 항목의 주범이다. 워크로드에 맞게 줄인다.
- max 상향 + 타임아웃 튜닝 + 사용률 알람이 한 세트다.

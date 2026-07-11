---
title: 쿠버네티스에서 외부 API만 5초씩 느릴 때, ndots:5의 비밀
date: 2026-06-23 19:20:00 +09:00
categories: [Infra]
tags: [kubernetes, dns, coredns, ndots]
description: 파드의 ndots:5 기본값이 외부 도메인 조회를 검색 도메인 수만큼 증폭시키는 구조와 ndots 조정, NodeLocal DNSCache 대응을 정리했다.
---

쿠버네티스 파드에서 외부 API 호출이 간헐적으로 5초씩 걸리는 문제를 공부했습니다. 5초는 DNS 재시도 타임아웃의 냄새인데, 뿌리는 파드의 resolv.conf에 있었습니다.

## ndots:5가 만드는 증폭

파드의 /etc/resolv.conf는 대략 이렇게 생겼습니다.

```
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`ndots:5`의 의미는 "이름에 점이 5개 미만이면 완전한 도메인이 아니라고 간주하고 search 목록을 먼저 붙여 시도하라"입니다. `api.stripe.com`은 점이 2개라 이 규칙에 걸립니다. 그래서 조회 순서가 이렇게 됩니다.

```
api.stripe.com.default.svc.cluster.local  → NXDOMAIN
api.stripe.com.svc.cluster.local          → NXDOMAIN
api.stripe.com.cluster.local              → NXDOMAIN
api.stripe.com                            → 드디어 성공
```

A와 AAAA를 각각 조회하니 외부 도메인 하나에 최대 8번의 DNS 쿼리가 나갑니다. 트래픽이 늘면 CoreDNS가 증폭된 쿼리에 짓눌려 지연·타임아웃·OOM까지 이어집니다.

## 대응

**1. ndots 낮추기.** 외부 호출이 많은 워크로드는 파드 스펙에서 조정합니다.

```yaml
dnsConfig:
  options:
    - name: ndots
      value: "2"
```

**2. FQDN으로 쓰기.** 도메인 끝에 점을 붙이면(`api.stripe.com.`) search 순회를 아예 건너뜁니다. 설정 변경 없이 코드/환경변수로 가능한 우회입니다.

**3. NodeLocal DNSCache.** 노드마다 DNS 캐시를 둬서 반복 조회를 로컬에서 끝내면 CoreDNS 부하와 지연이 함께 내려갑니다.

## 정리

- 쿠버네티스의 "외부만 느린 DNS"는 ndots:5 증폭부터 의심한다.
- 외부 트래픽 위주 파드는 ndots:2, 또는 FQDN(끝에 점) 표기로 회피한다.
- 규모가 있는 클러스터는 NodeLocal DNSCache가 사실상 표준이다.

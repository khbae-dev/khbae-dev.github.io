---
title: curl은 되는데 큰 파일만 멈추는 미스터리, MTU 블랙홀
date: 2026-07-04 19:23:00 +09:00
categories: [Infra]
tags: [network, mtu, vxlan, kubernetes]
description: 오버레이 네트워크의 캡슐화 오버헤드로 실효 MTU가 줄었는데 맞추지 않으면 큰 패킷만 조용히 버려지는 블랙홀 현상과 MSS clamp 대응을 정리했다.
---

네트워크 장애 중 가장 악랄한 부류를 공부했습니다. 작은 요청(헬스체크, curl)은 멀쩡한데 큰 파일 전송만 수십 초씩 멈추는, 에러조차 없는 장애입니다.

## 캡슐화가 훔쳐가는 50바이트

VXLAN 같은 오버레이 네트워크는 원래 패킷에 자기 헤더를 덧씌웁니다(VXLAN 기준 50바이트). 물리 링크 MTU가 1500이면 실제 페이로드가 쓸 수 있는 건 1450인 셈입니다. 그런데 컨테이너 인터페이스가 여전히 MTU 1500이라고 알고 있으면, 1500짜리 패킷이 캡슐화 후 1550이 되어 물리 링크에서 조각화되거나 DF(Don't Fragment) 비트 때문에 버려집니다.

정상이라면 라우터가 ICMP "Fragmentation Needed"를 돌려줘 송신 측이 크기를 줄입니다(PMTUD). 그런데 방화벽이 ICMP를 일괄 차단하면 이 신호도 사라집니다. 송신 측은 영문도 모른 채 재전송만 반복합니다. 이게 MTU 블랙홀입니다. 작은 패킷(요청, ACK)은 1450 이하라 통과하니 "연결은 되는데 데이터만 안 가는" 증상이 됩니다.

## 진단과 해결

```bash
# DF 비트 켜고 크기를 바꿔가며 ping → 통과하는 최대치가 경로 MTU
ping -M do -s 1472 target   # 1500 가정: 실패하면
ping -M do -s 1422 target   # 1450 가정: 성공?
```

해결은 두 층에서 합니다.

```bash
# 1. 인터페이스 MTU를 실효값으로 정렬 (CNI 설정)
ip link set dev eth0 mtu 1450

# 2. 게이트웨이에서 TCP MSS를 경로 MTU에 맞게 clamp
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
```

MSS clamp는 TCP 핸드셰이크의 MSS 값을 중간에서 고쳐 써서, 양 끝이 처음부터 작은 세그먼트로 통신하게 만듭니다. ICMP가 막혀 있어도 동작하는 실전 안전망입니다.

## 정리

- "작은 건 되고 큰 것만 안 된다"는 MTU 블랙홀의 시그니처다.
- 오버레이를 쓰면 인터페이스 MTU를 캡슐화 오버헤드만큼 줄여 정렬한다.
- ICMP를 막는 환경일수록 MSS clamp가 필수다.

---
title: 같은 리전인데 데이터 전송비 폭탄, cross-AZ 트래픽
date: 2026-06-14 16:03:00 +09:00
categories: [Infra]
tags: [aws, network, cost, availability-zone]
description: 같은 리전이라도 AZ를 넘는 트래픽에 양방향 요금이 붙는 구조를 이해하고, 수다스러운 서비스 경로를 same-AZ로 정렬해 비용을 줄이는 방법을 정리했다.
---

AWS 청구서에서 EC2-Other 항목이 이상하게 큰 이유를 추적하는 과정을 공부했습니다. 범인은 리전 간도 인터넷 전송도 아닌, 같은 리전 안의 AZ 간 트래픽이었습니다.

## 조용히 쌓이는 $0.01/GB

같은 리전 내에서도 가용영역(AZ)을 넘는 트래픽에는 GB당 요금(보통 방향별 $0.01, 즉 왕복 $0.02)이 붙습니다. 단가가 작아 보이지만, 서비스 간 호출·DB 조회·캐시 접근처럼 "수다스러운" 내부 트래픽이 하루 수 TB면 월 수천 달러가 조용히 쌓입니다. 고가용성을 위해 파드와 복제본을 AZ에 골고루 뿌려둔 아키텍처일수록, 호출이 AZ를 넘나들 확률이 높아 요금도 커집니다.

## 진단과 정렬

먼저 VPC Flow Logs나 Cost Explorer(usage type: DataTransfer-Regional-Bytes)로 어떤 경로가 AZ를 넘는지 확인합니다. 그다음 트래픽이 많은 경로부터 same-AZ로 정렬합니다.

쿠버네티스라면 Topology Aware Routing이 대표 수단입니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  annotations:
    service.kubernetes.io/topology-mode: Auto   # 같은 AZ 엔드포인트 우선
```

애플리케이션 레벨에서는 AZ-aware 클라이언트(같은 AZ의 캐시/리더 replica 우선 접속), Kafka라면 follower fetching(RackAwareReplicaSelector) 같은 옵션이 같은 원리입니다.

주의할 점은 가용성과의 균형입니다. same-AZ 정렬은 "평상시 경로"의 이야기이고, 복제본 자체는 여전히 AZ별로 분산해 둬야 AZ 장애를 견딥니다. 트래픽은 같은 AZ 우선, 데이터·용량은 다중 AZ — 이 분리가 핵심입니다.

## 정리

- 같은 리전이라고 내부 트래픽이 공짜가 아니다. AZ 경계에 가격표가 있다.
- 수다스러운 경로(서비스 간 호출, 캐시)부터 same-AZ 라우팅으로 정렬한다.
- 비용 최적화가 가용성 설계를 훼손하지 않도록, 경로와 배치를 분리해 생각한다.

---
title: Lambda를 VPC에 넣자 콜드스타트가 8초가 된 이유
date: 2026-07-04 19:21:00 +09:00
categories: [Infra]
tags: [aws, lambda, vpc, cold-start]
description: VPC 연결 Lambda의 ENI 생성 대기와 서브넷 IP 고갈이 콜드스타트를 폭증시키는 구조, 그리고 서브넷 확장과 provisioned concurrency 대응을 정리했다.
---

RDS에 붙이려고 Lambda를 VPC에 넣는 순간 콜드스타트가 튀는 이유를 공부했습니다.

## VPC Lambda의 숨은 준비물, ENI

VPC 밖 Lambda는 AWS 관리 네트워크에서 바로 뜹니다. VPC 안 리소스(RDS, ElastiCache)에 접근하려면 Lambda 실행 환경에 VPC 소속 네트워크 인터페이스(ENI)가 필요합니다. 문제는 이 ENI를 만들고 붙이는 작업이 콜드스타트 경로에 들어간다는 점입니다.

Hyperplane ENI 도입 이후에는 서브넷+보안그룹 조합당 ENI를 공유해 과거보다 훨씬 나아졌지만, 여전히 함정이 남아 있습니다.

- 첫 조합의 ENI 생성은 여전히 느리다 (배포 직후, 새 SG 조합).
- 서브넷의 가용 IP가 부족하면 ENI 생성이 큐에 밀리며 수 초씩 대기한다.
- 트래픽 스파이크로 동시 실행이 급증하면 ENI/IP 수요도 함께 튄다.

## 대응

**1. 서브넷 여유 확보.** Lambda 전용 서브넷을 /24 이상 넉넉하게 잡고, 다른 서비스와 IP 풀을 공유하지 않습니다. `AvailableIpAddressCount`에 알람을 걸어둡니다.

**2. provisioned concurrency로 예열.** 지연에 민감한 함수는 실행 환경을 미리 띄워 둡니다.

```bash
aws lambda put-provisioned-concurrency-config \
  --function-name checkout-api \
  --qualifier live \
  --provisioned-concurrent-executions 20
```

콜드스타트 자체가 사라지므로 ENI 이슈까지 함께 흡수됩니다. 비용이 붙으니 트래픽 패턴에 맞춰 Application Auto Scaling으로 스케줄링합니다.

**3. 연결 방식 재검토.** RDS라면 RDS Proxy를 앞에 두면 커넥션 수립 비용도 함께 줄어듭니다. DynamoDB처럼 VPC가 필요 없는 저장소로 설계를 바꾸는 것도 근본 대안입니다.

## 정리

- VPC Lambda의 콜드스타트 스파이크는 ENI와 서브넷 IP 재고 문제일 가능성이 높다.
- 서브넷은 넉넉하게, 가용 IP는 모니터링 대상에 넣는다.
- 지연 민감 경로는 provisioned concurrency로 콜드스타트 자체를 없앤다.

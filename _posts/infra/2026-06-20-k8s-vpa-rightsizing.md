---
title: 노드 40대에 CPU 사용률 20%, requests 과다 설정 바로잡기
date: 2026-06-20 19:23:00 +09:00
categories: [Infra]
tags: [kubernetes, vpa, autoscaler, cost]
description: 실사용보다 크게 잡힌 requests가 스케줄러를 속여 빈 노드를 늘리는 구조와 VPA 권고 기반 rightsizing으로 비용을 줄이는 과정을 정리했다.
---

클러스터 비용 최적화의 표준 수순을 공부했습니다. 출발점은 이상한 숫자 조합입니다. 노드는 40대인데 실제 CPU 사용률은 20%.

## 스케줄러는 requests만 본다

쿠버네티스 스케줄러는 파드를 배치할 때 실제 사용량이 아니라 선언된 requests를 봅니다. 개발자들이 "혹시 몰라서" CPU 1000m를 요청했는데 실사용이 50m라면, 노드의 할당(allocatable)은 꽉 찼는데 실제 CPU는 노는 상태가 됩니다. 스케줄러는 "자리가 없다"고 판단해 새 노드를 띄우고, cluster autoscaler는 그 노드들을 회수하지 못합니다. 과다 requests가 곧 과다 노드, 곧 비용입니다.

## VPA 권고 모드로 근거 만들기

Vertical Pod Autoscaler를 권고(recommendation) 모드로 켜면, 실제 사용 이력 기반의 적정 requests를 계산해 줍니다. 자동 적용(Auto)은 파드 재시작을 유발하니, 처음엔 보수적으로 권고만 받는 게 안전합니다.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Off"     # 권고만 생성, 적용은 수동
```

```bash
kubectl describe vpa api-vpa   # target(권고값) 확인
```

권고값을 배포 매니페스트에 반영하면 파드들이 조밀하게 재배치되고, 비는 노드는 autoscaler가 자연스럽게 회수합니다. 노드 수가 절반 이하로 내려가는 사례가 드물지 않습니다. HPA(수평 확장)와 같은 메트릭을 두고 충돌하지 않게 대상 분리는 신경 써야 합니다.

## 정리

- 클러스터 비용의 숨은 주범은 실사용과 동떨어진 requests다.
- VPA 권고 모드 → 매니페스트 반영 → autoscaler 회수, 이 순서가 안전한 rightsizing이다.
- requests는 "감"이 아니라 사용 이력 데이터로 정하는 값이다.

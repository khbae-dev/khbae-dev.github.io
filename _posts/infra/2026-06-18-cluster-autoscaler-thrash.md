---
title: 노드가 떴다 사라졌다 반복하는 오토스케일러 thrash 잡기
date: 2026-06-18 19:23:00 +09:00
categories: [Infra]
tags: [kubernetes, autoscaler, pdb, stability]
description: 방금 띄운 노드를 성급하게 회수하며 생기는 스케일 진동의 원인과 scale-down 안정화 윈도우, requests 정렬, PDB 조합을 정리했다.
---

클러스터 노드 수 그래프가 톱니처럼 오르내리는 현상을 공부했습니다. 진동(thrash)은 그 자체로 비용이고, 노드 프로비저닝·파드 재배치가 반복되며 지연 스파이크도 만듭니다.

## 진동의 메커니즘

전형적인 루프는 이렇습니다.

1. 부하가 살짝 올라 HPA가 파드를 늘림 → 자리가 없어 노드 증설
2. 부하가 내려가 파드 축소 → 노드 사용률이 낮아짐
3. autoscaler가 저사용 노드를 회수하며 파드를 다른 노드로 이사
4. 이사 과정의 재시작·워밍업으로 지연 상승 → 다시 파드 증가 → 노드 증설…

원인은 대부분 "회수가 너무 성급해서"입니다. 여기에 requests가 실사용과 어긋나 있으면 사용률 계산 자체가 왜곡돼 진동이 더 심해집니다.

## 안정화 3종

**1. scale-down 안정화 윈도우.** 회수 판단에 뜸을 들입니다.

```yaml
# cluster-autoscaler 옵션
--scale-down-delay-after-add=15m       # 노드 추가 후 15분간 회수 금지
--scale-down-unneeded-time=15m         # 15분 연속 저사용일 때만 회수
--scale-down-utilization-threshold=0.5
```

HPA 쪽도 `behavior.scaleDown.stabilizationWindowSeconds`로 파드 축소에 같은 완충을 둡니다.

**2. requests 정렬.** VPA 권고로 requests를 실사용에 맞추면 사용률 판단이 정확해져 불필요한 증설·회수가 줄어듭니다.

**3. PDB로 이사 속도 제어.** PodDisruptionBudget은 노드 회수 시 동시에 빠질 수 있는 파드 수를 제한해, 회수가 일어나더라도 서비스 영향과 연쇄 재배치를 완만하게 만듭니다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 80%
  selector: { matchLabels: { app: api } }
```

## 정리

- 스케일 진동은 "성급한 회수 + 왜곡된 사용률"의 합작품이다.
- 안정화 윈도우로 회수에 관성을 주는 게 첫 번째 손잡이다.
- requests 정렬과 PDB까지 맞추면 노드 수가 계단이 아니라 완만한 곡선이 된다.

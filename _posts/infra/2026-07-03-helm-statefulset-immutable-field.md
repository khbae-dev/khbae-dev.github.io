---
title: helm upgrade가 StatefulSet에서 영원히 실패할 때, immutable 필드
date: 2026-07-03 19:24:00 +09:00
categories: [Infra]
tags: [kubernetes, helm, statefulset, devops]
description: StatefulSet의 변경 불가 필드를 수정하는 업그레이드가 release를 잠그는 문제와 cascade=orphan으로 파드를 살린 채 재생성하는 방법을 정리했다.
---

helm upgrade 한 번에 배포 파이프라인이 계속 실패 상태에 빠지는 사례를 공부했습니다. 에러 메시지는 이렇습니다.

```
Error: UPGRADE FAILED: cannot patch "kafka" with kind StatefulSet:
StatefulSet.apps "kafka" is invalid: spec: Forbidden:
updates to statefulset spec for fields other than 'replicas',
'template', 'updateStrategy', ... are forbidden
```

## StatefulSet은 절반이 불변이다

쿠버네티스 StatefulSet은 생성 후 `replicas`, `template`, `updateStrategy` 등 일부 필드만 수정할 수 있습니다. 특히 `volumeClaimTemplates`(디스크 크기·스토리지클래스), `serviceName`, `selector`는 불변입니다. 차트 값에서 디스크 크기만 늘려도 helm이 patch를 시도하다 거부당하고, release는 실패 상태로 남아 이후 모든 upgrade가 막힙니다.

## 파드는 살리고 껍데기만 갈아끼우기

데이터와 서비스를 유지한 채 StatefulSet 오브젝트만 재생성하는 절차입니다.

```bash
# 1. StatefulSet만 삭제하되 파드는 남긴다 (orphan)
kubectl delete statefulset kafka --cascade=orphan

# 2. 새 정의로 다시 적용 → 기존 파드를 라벨 셀렉터로 다시 입양
helm upgrade kafka ./chart -f values.yaml
```

`--cascade=orphan`이 핵심입니다. 컨트롤러 오브젝트만 지워지고 파드·PVC는 그대로 돌아가므로 다운타임이 없습니다. 새 StatefulSet이 생성되면 셀렉터가 일치하는 기존 파드들을 그대로 관리 대상으로 흡수합니다.

주의할 점이 있습니다. 이 방법으로 "정의"는 바꿀 수 있지만, 기존 PVC의 실제 크기는 volumeClaimTemplates를 바꿔도 소급 적용되지 않습니다. 기존 볼륨 확장은 StorageClass가 `allowVolumeExpansion: true`일 때 PVC를 직접 수정하는 별도 작업입니다.

## 정리

- StatefulSet 업그레이드 실패는 immutable 필드 변경을 먼저 의심한다.
- cascade=orphan 삭제 후 재적용이 무중단 표준 절차다.
- 볼륨 크기는 템플릿 수정과 별개로 PVC 직접 확장이 필요하다.

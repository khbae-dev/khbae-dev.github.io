---
title: Error acquiring the state lock, terraform 락 사고 복구 절차
date: 2026-06-29 19:24:00 +09:00
categories: [Infra]
tags: [terraform, state-lock, devops, ci]
description: 비정상 종료한 apply가 state 락을 잡은 채 사라졌을 때의 안전한 복구 절차와 재발 방지 설정을 정리했다.
---

모든 배포가 `Error acquiring the state lock`으로 멈추는 상황의 표준 대응을 정리했습니다.

## 락은 왜 남겨지나

terraform은 state 동시 수정을 막으려고 apply/plan 시작 시 백엔드(S3+DynamoDB, Terraform Cloud 등)에 락을 겁니다. 정상 종료하면 풀리지만, CI 러너가 강제 종료되거나(타임아웃, spot 회수), 네트워크가 끊기면 락만 남습니다. 이후 모든 실행이 락 획득 실패로 거부됩니다.

## 복구: 검증 먼저, 해제는 그 다음

에러 메시지에 찍힌 락 정보(ID, Who, Created)가 출발점입니다.

```
Lock Info:
  ID:        a1b2c3d4-...
  Who:       runner@ci-worker-7
  Created:   2026-06-29 09:12:33 UTC
```

**1. 그 실행이 정말 죽었는지 확인합니다.** Who/Created로 CI 잡을 찾아 상태를 확인합니다. 아직 돌고 있는 apply의 락을 풀면 state 파일이 두 실행에 의해 동시에 써져 손상될 수 있습니다. 이게 최악의 시나리오라 검증이 필수입니다.

**2. 죽은 게 확실하면 강제 해제합니다.**

```bash
terraform force-unlock a1b2c3d4-...
```

**3. state 정합성을 확인합니다.** `terraform plan`으로 예상 밖의 diff가 없는지 봅니다. 죽은 apply가 리소스를 절반만 만들었을 수 있으므로, diff가 이상하면 `terraform state list`와 실제 인프라를 대조합니다.

재발 방지는 두 가지입니다. 파이프라인에서 terraform 잡의 동시 실행을 금지(concurrency group 직렬화)하고, `-lock-timeout=5m`을 줘서 짧은 경합은 대기로 흡수하게 합니다.

## 정리

- force-unlock은 "죽은 실행" 확인 후에만. 산 실행의 락을 풀면 state가 깨진다.
- 해제 후 plan으로 반쪽 적용 여부를 반드시 점검한다.
- CI 직렬화 + lock-timeout이면 락 사고 자체가 드물어진다.

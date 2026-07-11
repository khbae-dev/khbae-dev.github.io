---
title: terraform apply 한 번에 운영 DB가 사라질 수 있다, forces replacement
date: 2026-06-08 16:02:00 +09:00
categories: [Infra]
tags: [terraform, iac, safety, rds]
description: 불변 속성 변경이 리소스 삭제 후 재생성으로 이어지는 forces replacement의 위험과 prevent_destroy, CI plan 검토 안전망을 정리했다.
---

IaC에서 가장 서늘한 시나리오를 공부했습니다. 설정 한 줄 바꾼 apply가 운영 데이터베이스를 "삭제 후 재생성"하는 경우입니다.

## replace는 destroy를 포함한다

클라우드 리소스의 일부 속성은 생성 후 변경이 불가능합니다. RDS의 `storage_encrypted`, 일부 인스턴스의 AZ, 리소스 이름류가 그렇습니다. 이런 속성을 바꾸면 terraform은 "수정"이 아니라 "기존 것을 부수고 새로 만들기"를 계획합니다. plan 출력에 그 신호가 있습니다.

```
# aws_db_instance.main must be replaced
-/+ resource "aws_db_instance" "main" {
      ~ storage_encrypted = false -> true # forces replacement
```

`-/+`와 `forces replacement`가 그 표식입니다. 이걸 못 보고 apply하면 DB가 destroy됩니다. 스냅샷 설정이 없다면 데이터도 함께 사라집니다.

## 안전망 3중

**1. 삭제 금지 선언.** 상태를 가진 핵심 리소스에는 lifecycle 가드를 박아둡니다.

```hcl
resource "aws_db_instance" "main" {
  # ...
  deletion_protection = true          # AWS 레벨 보호
  lifecycle {
    prevent_destroy = true            # terraform 레벨: destroy 계획 시 에러
  }
}
```

**2. CI에서 plan을 사람이 본다.** `terraform plan -detailed-exitcode`는 변경이 있으면 exit 2를 반환합니다. 파이프라인에서 plan 결과를 PR에 첨부하고, `must be replaced` 문자열이 있으면 라벨·경고를 자동으로 붙이는 게스트를 두면 실수를 걸러냅니다.

**3. 최후의 보루.** RDS라면 `skip_final_snapshot = false`와 최종 스냅샷 이름 설정, 백업 보존 기간 확보. 사고가 나도 돌아올 길을 남기는 겁니다.

## 정리

- plan의 -/+ 와 forces replacement는 "삭제 예정" 경고문이다. 읽는 습관이 안전장치의 절반.
- 상태 있는 리소스에는 prevent_destroy + deletion_protection을 기본으로.
- 자동화(plan 검사)와 복구책(스냅샷)까지가 IaC 안전망의 전체다.

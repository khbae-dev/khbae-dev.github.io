---
title: CI에 박아둔 영구 AWS 키가 유출됐을 때, OIDC 단기 토큰으로 갈아타기
date: 2026-06-22 19:22:00 +09:00
categories: [Security]
tags: [security, oidc, github-actions, iam]
description: CI 시크릿에 저장한 장기 클라우드 키의 위험과, GitHub OIDC로 요청 시점에 발급되는 단기 토큰으로 전환하는 방법을 정리했다.
---

CI에 저장한 영구 클라우드 키 하나가 유출돼 계정 전체가 털리는 사고 유형을 공부했습니다. 대안은 "키를 저장하지 않는" 구조로 바꾸는 것이었습니다.

## 장기 키의 근본 문제

CI secrets에 넣어둔 AWS 액세스 키는 편하지만 위험이 큽니다. 만료가 없고(회전하지 않으면 영원히 유효), 로그·서드파티 액션·포크 PR 등 유출 경로가 많고, 유출돼도 언제 새어나갔는지 알기 어렵습니다. 한 번 새면 공격자가 그 권한으로 무엇이든 할 수 있고, 탐지 전까지 시간이 무제한입니다.

## OIDC: 저장 대신 실시간 발급

GitHub Actions는 OIDC로 각 워크플로 실행마다 서명된 신원 토큰을 발급합니다. AWS가 이 토큰을 신뢰하도록 설정하면, 잡이 실행되는 순간 그 자리에서 1시간짜리 임시 자격증명을 받습니다. 저장할 장기 키가 아예 없어집니다.

```yaml
permissions:
  id-token: write        # OIDC 토큰 발급 권한
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
      aws-region: ap-northeast-2
      # 액세스 키 없음 — OIDC 토큰으로 role을 assume
```

AWS IAM 쪽에서는 신뢰 정책으로 "이 저장소의 이 브랜치에서 온 토큰만" 허용하도록 조건을 겁니다.

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:sub":
      "repo:myorg/myrepo:ref:refs/heads/main"
  }
}
```

이러면 자격증명은 1시간 뒤 자동 만료되고, 특정 저장소·브랜치·환경에서만 발급되며, 최소 권한 role에 묶입니다. 유출돼도 수명이 짧고 범위가 좁아 피해가 제한됩니다. 같은 원리(workload identity federation)를 GCP·Azure도 지원합니다.

## 정리

- CI에 저장한 장기 클라우드 키는 만료·범위·탐지 모두 취약하다.
- OIDC 연동으로 실행 시점에 단기 자격증명을 받으면 저장할 키가 사라진다.
- 신뢰 정책의 sub 조건(저장소·브랜치)과 최소 권한 role까지 맞춰야 완성이다.

---

*보안 취약점은 방어 목적으로만 다룹니다.*

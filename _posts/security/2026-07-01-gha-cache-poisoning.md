---
title: 외부 PR이 메인 CI에 악성코드를 심는 경로, GitHub Actions 캐시 포이즈닝
date: 2026-07-01 19:23:00 +09:00
categories: [Security]
tags: [security, github-actions, supply-chain, ci]
description: pull_request_target과 브랜치 미분리 캐시가 결합해 외부 PR이 신뢰 컨텍스트의 캐시를 오염시키는 공급망 공격 경로와 방어를 정리했다.
---

CI 공급망 공격 중 은근히 알려지지 않은 경로를 공부했습니다. GitHub Actions의 캐시를 매개로 외부 PR이 메인 빌드에 침투하는 방식입니다.

## pull_request_target + 공유 캐시

`pull_request_target` 이벤트는 포크 PR에 대해서도 "베이스 저장소의 컨텍스트"에서 워크플로를 실행합니다. 시크릿 접근이 필요해 쓰지만, 여기서 포크의 코드를 체크아웃해 실행하면 공격자 코드가 신뢰 컨텍스트에서 도는 위험이 생깁니다.

여기에 캐시가 얽히면 더 교묘해집니다. GitHub Actions 캐시는 기본적으로 브랜치 스코프가 있지만, 캐시 키를 브랜치 구분 없이 잡으면 포크 PR의 잡이 만든 캐시를 이후 메인 브랜치 잡이 복원할 수 있습니다. 공격자가 캐시에 악성 바이너리(조작된 node_modules, 빌드 산출물)를 심어두면, 나중에 메인 CI가 그걸 복원해 실행합니다. 공급망 오염입니다.

## 방어

```yaml
# 1. 포크 PR 코드를 신뢰 컨텍스트에서 실행하지 않는다
#    pull_request_target에서 PR 헤드를 체크아웃해 빌드/실행 금지
on: pull_request        # 가능하면 시크릿 없는 pull_request 사용

# 2. 캐시 키를 신뢰 경계로 스코프 분리
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ github.ref }}-npm-${{ hashFiles('**/package-lock.json') }}
    #    ^^^ ref를 키에 포함 → 브랜치 간 캐시 격리

# 3. 토큰 최소 권한
permissions:
  contents: read        # 기본을 read-only로
```

핵심 원칙은 "신뢰 경계를 넘는 데이터 재사용 금지"입니다. 포크가 만든 어떤 산출물(캐시, 아티팩트)도 신뢰 잡이 검증 없이 소비하지 않게 합니다. 캐시 키에 `github.ref`를 넣어 브랜치별로 격리하고, 워크플로 권한은 read-only를 기본값으로 두고 필요한 잡에만 상향합니다.

## 정리

- pull_request_target에서 포크 코드 실행 + 브랜치 미분리 캐시 = 공급망 오염 통로다.
- 캐시 키에 ref를 넣어 신뢰 경계별로 캐시를 격리한다.
- 워크플로 토큰은 최소 권한(read) 기본, 시크릿 필요한 잡만 예외로 좁힌다.

---

*보안 취약점은 방어 목적으로만 다룹니다.*

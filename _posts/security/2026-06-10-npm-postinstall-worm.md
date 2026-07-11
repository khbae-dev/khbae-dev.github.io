---
title: npm install 한 번에 토큰이 새는 postinstall 스크립트 공격
date: 2026-06-10 16:00:00 +09:00
categories: [Security]
tags: [security, npm, supply-chain, ci]
description: 악성 패키지의 postinstall 스크립트가 설치만으로 실행돼 토큰을 탈취하고 자가 전파하는 공격과 --ignore-scripts 방어를 정리했다.
---

의존성 하나 설치했을 뿐인데 CI 토큰이 유출되는 공급망 웜 공격을 공부했습니다. 핵심은 "설치 = 코드 실행"이라는 npm의 특성이었습니다.

## postinstall이 자동으로 돈다

npm 패키지는 `postinstall` 같은 lifecycle 스크립트를 정의할 수 있고, `npm install` 시 자동 실행됩니다. 원래는 네이티브 모듈 빌드 같은 정당한 용도지만, 악성 패키지에겐 완벽한 실행 시점입니다. 최근의 자가 전파형 웜(Shai-Hulud 등으로 알려진 사례)은 이 경로로 동작했습니다. 설치되는 순간 postinstall이 환경변수의 토큰(npm, GitHub, 클라우드 키)을 긁어 외부로 보내고, 접근 가능한 다른 패키지에 자신을 심어 전파합니다. 개발자 로컬이든 CI든 install만 하면 실행됩니다.

## 방어: 스크립트 자동 실행을 끈다

CI에서는 lifecycle 스크립트를 아예 비활성화하는 게 가장 효과적입니다.

```bash
# CI: lockfile 고정 설치 + 스크립트 차단
npm ci --ignore-scripts
```

```
# 또는 프로젝트 전역으로 (.npmrc)
ignore-scripts=true
```

`--ignore-scripts`를 켜면 대부분의 패키지는 문제없이 동작합니다(네이티브 빌드가 필요한 일부만 예외 처리). 여기에 여러 겹을 더합니다.

- 설치를 격리 환경에서: CI 러너를 최소 권한으로 두고 민감 토큰을 install 단계 환경에 노출하지 않습니다(OIDC 단기 토큰과 결합).
- 무결성·버전 고정: `npm ci`로 lockfile 해시를 강제하고, 새 버전은 며칠 관망 후 채택합니다.
- 의존성 감사: `npm audit`, 소셜 엔지니어링·타이포스쿼팅 탐지 도구를 파이프라인에 넣습니다.

## 정리

- npm의 lifecycle 스크립트는 "설치만으로 코드가 실행되는" 공격 표면이다.
- CI는 `npm ci --ignore-scripts`를 기본값으로 두는 게 안전하다.
- 토큰을 install 단계에 노출하지 않는 격리와 버전 고정까지 겹쳐야 한다.

---

*보안 취약점은 방어 목적으로만 다룹니다.*

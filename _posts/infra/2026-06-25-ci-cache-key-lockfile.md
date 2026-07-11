---
title: 캐시를 켰는데 CI가 여전히 12분 걸리는 이유, 캐시 키 설계
date: 2026-06-25 19:22:00 +09:00
categories: [Infra]
tags: [ci, cache, github-actions, build]
description: 캐시 키를 커밋 SHA로 잡아 매번 미스가 나는 실수와, lockfile 해시 키 + 계층형 restore-keys로 고치는 방법을 정리했다.
---

CI에 캐시 설정이 분명 있는데 빌드 시간이 그대로인 경우를 공부했습니다. 캐시의 존재 여부가 아니라 "키 설계"가 문제였습니다.

## 키가 매번 바뀌면 캐시는 장식이다

```yaml
# 나쁨: 커밋마다 키가 바뀐다 → 영원히 MISS
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ github.sha }}
```

캐시는 키가 정확히 일치해야 히트합니다. 커밋 SHA를 키에 넣으면 커밋마다 새 키가 되므로 저장만 하고 재사용은 못 하는 상태가 됩니다. 반대로 키를 `npm-cache`처럼 고정하면 의존성이 바뀌어도 낡은 캐시를 계속 물고 있는 문제가 생깁니다.

## 원칙: 키는 "내용물의 해시"로

캐시의 내용물을 결정하는 파일(lockfile)의 해시를 키로 씁니다. 의존성이 같으면 히트, 바뀌면 자연히 새 키가 됩니다.

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-
```

`restore-keys`가 두 번째 포인트입니다. 정확한 키가 없을 때 접두사가 일치하는 가장 최근 캐시를 "부분 복원"해 줍니다. lockfile이 바뀌어도 대부분의 패키지는 그대로이므로, 이전 캐시 위에서 바뀐 것만 새로 받으면 됩니다. 전량 다운로드와 부분 다운로드의 차이가 빌드 시간 12분과 2분의 차이를 만듭니다.

같은 원리가 Docker 레이어 캐시, Gradle, pip, cargo 전부에 적용됩니다. "이 캐시의 유효성을 결정하는 입력 파일이 무엇인가"를 찾아 그 해시를 키로 삼으면 됩니다.

## 정리

- 캐시 키에 커밋 SHA는 반(反)패턴이다. 내용물을 결정하는 파일의 해시를 쓴다.
- restore-keys 계층으로 "완전 미스"를 "부분 히트"로 바꾼다.
- 캐시 히트율을 CI 로그에서 실제로 확인하는 것까지가 설정의 완료다.

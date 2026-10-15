---
title: Git Bash에서만 빌드가 깨지는 이유, MSYS 경로 변환
date: 2026-10-15 19:10:00 +09:00
categories: [Infra]
tags: [windows, git-bash, msys, build, environment-variable]
description: Git Bash가 슬래시로 시작하는 환경변수를 Windows 경로로 바꿔 넘기면서 생기는 빌드 오류와, 빌드 스크립트에서 이를 감지해 멈추는 방법을 정리했다.
---

같은 명령으로 빌드했는데 PowerShell에서는 되고 Git Bash에서는 Sass 오류로 죽는 일이 있었습니다. 원인은 코드가 아니라 셸이었습니다.

## 증상

서브 경로(`/myapp`) 아래에 배포할 프런트엔드를 빌드하려고 기준 경로를 환경변수로 넘겼습니다.

```bash
BASE_PATH=/myapp npm run build
# SassError: expected ")".
```

빌드 스크립트에서 값을 찍어 보니 `/myapp`이 아니라 `C:/Program Files/Git/myapp`이 들어와 있었습니다. 이 값이 CSS의 `url(/assets/...)` 앞에 붙으면서 `url(/C:/Program Files/Git/myapp/assets/...)`가 됐고, 공백과 콜론 때문에 Sass 파서가 깨진 것입니다.

## 원인: MSYS의 POSIX 경로 자동 변환

Git for Windows에 들어 있는 Git Bash는 MSYS2 위에서 돕니다. MSYS는 Windows 네이티브 프로그램(`node.exe` 등)을 실행할 때 **인자와 환경변수 중 `/`로 시작하는 값을 POSIX 경로로 보고 Windows 경로로 바꿔 줍니다.** `/usr/bin`을 `C:\Program Files\Git\usr\bin`으로 바꿔 주는 편의 기능인데, URL 경로처럼 경로가 아닌 값도 똑같이 바꿔 버립니다.

`docker run -v /data:/data`나 `--prefix=/api` 같은 인자도 같은 이유로 깨질 수 있습니다.

## 피하는 방법

```bash
# 1. 앞 슬래시를 빼고 넘긴 뒤, 스크립트가 붙인다
BASE_PATH=myapp npm run build

# 2. 이 명령에 한해 변환을 끈다 (Git for Windows)
MSYS_NO_PATHCONV=1 BASE_PATH=/myapp npm run build
```

PowerShell이나 cmd에서는 이런 변환이 없으므로 `/myapp`을 그대로 넘겨도 됩니다.

## 빌드 스크립트가 직접 막기

사람이 매번 기억하는 건 믿을 수 없어서, 빌드 스크립트가 바뀐 값을 알아보고 이유를 적고 멈추게 했습니다. 추측해서 원래 값으로 되돌리지는 않았습니다. 잘못 복원하면 오류 없이 엉뚱한 경로로 배포되기 때문입니다.

```js
function getBasePath() {
  const raw = (process.env.BASE_PATH || '').trim();
  if (/^[A-Za-z]:[\\/]/.test(raw)) {
    throw new Error(
      `BASE_PATH가 Windows 경로로 바뀌어 들어왔습니다: "${raw}"\n` +
      'Git Bash라면 앞 슬래시 없이 BASE_PATH=myapp 으로 주십시오.');
  }
  const p = ('/' + raw.replace(/^\/+/, '')).replace(/\/$/, '');
  if (p === '') return '';
  // URL 경로 조각만 허용한다. 공백·따옴표·괄호는 CSS와 HTML을 깨뜨린다
  if (!/^(\/[A-Za-z0-9._~-]+)+$/.test(p)) {
    throw new Error(`BASE_PATH 형식이 올바르지 않습니다: "${raw}"`);
  }
  return p;
}
```

실제로 들어왔던 값을 그대로 테스트 케이스로 남겨 두면 다음에 누가 검증을 지워도 바로 드러납니다.

## 정리

- Git Bash는 Windows 프로그램에 넘기는 `/`로 시작하는 값을 경로로 보고 바꾼다. URL 경로도 예외가 아니다.
- 앞 슬래시를 빼고 넘기거나 `MSYS_NO_PATHCONV=1`로 변환을 꺼서 피한다.
- 빌드 스크립트는 환경변수 형식을 검증하고, 잘못된 값이면 추측해서 고치지 말고 이유를 적고 멈춘다.

---
title: 한 글자 고쳤는데 빌드 10분, Dockerfile 순서의 경제학
date: 2026-06-05 16:02:00 +09:00
categories: [Infra]
tags: [docker, build, cache, ci]
description: COPY 순서 때문에 의존성 설치 레이어 캐시가 매번 무효화되는 문제와, 변경 빈도 순으로 레이어를 배치하는 원칙을 정리했다.
---

CI에서 Docker 빌드가 코드 한 줄 수정에도 10분씩 걸리는 원인을 정리했습니다. Dockerfile 명령 순서가 전부였습니다.

## 캐시는 위에서부터 깨진다

Docker는 명령마다 레이어를 만들고, 입력이 같으면 캐시를 재사용합니다. 규칙이 하나 있는데, 어떤 레이어가 캐시 미스가 나면 그 아래 모든 레이어도 다시 빌드됩니다.

```dockerfile
# 나쁨: 소스 전체를 먼저 복사
COPY . .              # 소스 한 글자만 바뀌어도 이 레이어부터 미스
RUN npm ci            # → 의존성 설치가 매번 다시 (10분의 주범)
```

소스는 매 커밋 바뀌지만 의존성은 가끔 바뀝니다. 변경이 잦은 것을 위에 두면, 변경이 드문 비싼 작업(의존성 설치)까지 매번 끌려 내려가 다시 실행됩니다.

## 변경 빈도의 역순으로 쌓는다

```dockerfile
FROM node:20-slim
WORKDIR /app

# 1. 가장 안 바뀌는 것부터: 의존성 명세만 먼저
COPY package.json package-lock.json ./
RUN npm ci                    # lockfile이 같으면 항상 캐시 히트

# 2. 자주 바뀌는 소스는 마지막에
COPY . .
RUN npm run build
```

이제 소스만 바뀐 빌드에서는 `npm ci` 레이어가 캐시로 재사용됩니다. 10분 빌드가 1~2분이 되는 원리입니다.

CI 환경에서는 러너가 매번 초기화돼 로컬 캐시가 없을 수 있으므로, BuildKit의 외부 캐시를 함께 씁니다.

```bash
docker buildx build \
  --cache-from type=registry,ref=myapp:cache \
  --cache-to   type=registry,ref=myapp:cache,mode=max .
```

## 정리

- Dockerfile은 "변경 빈도가 낮은 것 → 높은 것" 순서로 쌓는 게 원칙이다.
- 의존성 명세 COPY와 설치를 소스 COPY보다 앞에 두는 것만으로 빌드 시간이 자릿수로 줄 수 있다.
- CI에서는 registry 캐시(cache-from/to)까지 연결해야 효과가 유지된다.

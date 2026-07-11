---
title: Docker 이미지 1.2GB를 80MB로, 멀티스테이지 빌드
date: 2026-06-04 02:32:00 +09:00
categories: [Infra]
tags: [docker, multi-stage, image-size, devops]
description: 빌드 도구까지 최종 이미지에 포함돼 비대해지는 문제를, 빌드 스테이지와 실행 스테이지를 분리해 산출물만 옮기는 멀티스테이지 빌드로 해결하는 방법을 정리했다.
---

Docker 이미지가 1GB를 훌쩍 넘는 이유와 다이어트 방법을 정리했습니다. 대부분은 "빌드에만 필요한 것"이 최종 이미지에 눌러앉아 있는 문제입니다.

## 최종 이미지에 컴파일러가 왜 있나

한 스테이지로 빌드하면 컴파일러, 빌드 툴체인, dev 의존성, 소스코드, 중간 산출물이 전부 이미지에 남습니다. 실행에는 결과물(바이너리 또는 번들)과 런타임만 있으면 되는데 말입니다. 이미지가 크면 세 가지 손해입니다. 레지스트리 push/pull이 느리고, 노드 디스크를 먹고, 불필요한 패키지만큼 공격 표면(취약점)이 넓어집니다.

## 스테이지를 나눠 산출물만 옮긴다

```dockerfile
# 1단계: 빌드 (무거운 도구 총동원)
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                    # dev 의존성 포함
COPY . .
RUN npm run build             # dist/ 산출

# 2단계: 실행 (slim 베이스에 결과물만)
FROM node:20-slim AS runtime
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev         # 런타임 의존성만
COPY --from=builder /app/dist ./dist   # 빌드 산출물만 가져온다
USER node
CMD ["node", "dist/server.js"]
```

핵심은 `COPY --from=builder`입니다. 최종 이미지에는 builder 스테이지의 도구·소스가 전혀 들어오지 않고, 지정한 산출물만 복사됩니다. Go나 Rust처럼 정적 바이너리를 만드는 언어라면 실행 스테이지를 `distroless`나 `scratch`로 두어 수 MB까지 줄일 수 있습니다.

추가로 챙길 것들도 있습니다. `.dockerignore`로 node_modules·.git을 빌드 컨텍스트에서 빼고, 비root 유저로 실행하고, slim/distroless 베이스를 고르면 크기와 보안이 함께 개선됩니다.

## 정리

- 이미지 비대화의 주범은 "빌드에만 필요한 것"의 잔류다.
- 빌드/실행 스테이지를 나누고 COPY --from으로 산출물만 옮기면 자릿수로 줄어든다.
- 작은 이미지는 배포 속도뿐 아니라 공격 표면 축소라는 보안 이득도 준다.

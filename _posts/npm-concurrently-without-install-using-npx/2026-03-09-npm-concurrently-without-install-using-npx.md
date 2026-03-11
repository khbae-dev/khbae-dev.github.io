---
title: npm concurrently 설치 없이 실행하는 방법
date: 2026-03-09 11:30:00 +09:00
tags: [npm, nodejs, nx, vite, javascript]
description: concurrently 설치가 실패하거나 dependency 충돌이 날 때 npx concurrently로 여러 npm script를 동시에 실행하는 방법을 정리했다.
---

여러 개의 npm script를 동시에 실행하려고 `concurrently`를 붙였는데, 정작 설치나 실행 단계에서 막히는 경우가 있습니다.
특히 Nx + Vite 같이 의존성이 많은 프로젝트에서는 `peer dependency conflict` 때문에 단순한 devDependency 추가도 번거로워질 수 있습니다.

이 글은 `concurrently`를 설치하지 않고 `npx`로 바로 실행하는 방법을 중심으로 정리한 스터디 리포트입니다.

## 이번에 다시 정리하면서 남긴 메모

- 처음부터 의존성 정리로 들어가기보다 실행 경로를 바꿔보는 우회가 더 싼 경우가 많았다.
- `concurrently` 설치 문제와 실제 개발 서버를 띄워야 하는 요구를 분리해서 보니 답이 더 단순해졌다.
- Nx 환경에서는 패키지 추가보다 `run-many --parallel`이나 `npx`처럼 기존 도구를 먼저 보는 편이 낫다.

## 1. 문제 상황

보통은 여러 스크립트를 동시에 실행할 때 아래처럼 작성합니다.

```json
{
  "scripts": {
    "dev": "concurrently \"npm run app1\" \"npm run app2\""
  }
}
```

이 방식은 `concurrently`가 프로젝트에 설치돼 있어야 합니다.

```bash
npm install concurrently --save-dev
```

문제는 여기서부터 시작될 수 있습니다.

- `concurrently` 명령을 찾지 못함
- `npm install` 도중 `peer dependency conflict`
- Nx, Vite, 기타 도구 버전 충돌
- 기존 `node_modules`가 이미 복잡해서 패키지 추가가 부담됨

이럴 때 npm 버전 업그레이드나 의존성 재정리 없이 우회할 수 있는 방법이 `npx`입니다.

## 2. 해결 방법: npx로 concurrently 실행

`npx`는 패키지를 프로젝트에 설치하지 않고도 실행할 수 있게 해주는 도구입니다.
즉 `devDependencies`에 `concurrently`를 추가하지 않아도 됩니다.

핵심은 아주 단순합니다.

```json
{
  "scripts": {
    "dev": "npx concurrently \"npm run app1\" \"npm run app2\""
  }
}
```

이제 실행은 그대로 하면 됩니다.

```bash
npm run dev
```

즉 기존 명령 앞에 `npx`만 붙여도 설치 의존성을 줄일 수 있습니다.

## 3. npx 방식이 동작하는 원리

흐름은 대체로 아래와 같습니다.

1. `npx`가 필요한 패키지를 찾는다.
2. 로컬에 없으면 임시 실행 가능한 형태로 가져온다.
3. 명령을 실행한다.
4. 이후에는 캐시를 재사용할 수 있다.

중요한 점은 프로젝트의 `package.json`이나 `node_modules`에 직접 의존성을 추가하지 않는다는 점입니다.
그래서 dependency 충돌을 줄이고, 빠르게 테스트하기에 좋습니다.

## 4. 실제 적용 예시

예를 들어 Nx 프로젝트에서 여러 앱을 동시에 띄우고 싶다고 가정해 보겠습니다.

기존 방식:

```json
{
  "scripts": {
    "localhost:all-plugins": "concurrently \"npm run port1\" \"npm run dashboard:plugin-localhost\""
  }
}
```

설치 없이 실행하는 방식:

```json
{
  "scripts": {
    "localhost:all-plugins": "npx concurrently \"npm run port1\" \"npm run dashboard:plugin-localhost\""
  }
}
```

이렇게 바꾸면 아래 장점이 있습니다.

- `concurrently`를 별도로 설치하지 않아도 된다.
- dependency 충돌 가능성을 줄일 수 있다.
- npm 버전을 당장 올릴 필요가 없다.

## 5. 장점과 단점

장점:

- devDependency를 추가하지 않아도 된다.
- `peer dependency conflict`를 피하기 쉽다.
- 임시 실행이나 빠른 검증에 적합하다.
- CI/CD에서 잠깐 필요한 실행에도 유용하다.

단점:

- 첫 실행 시 다운로드 때문에 약간 느릴 수 있다.
- 캐시에 의존하므로 환경에 따라 동작 편차가 있을 수 있다.
- 네트워크 제약이 있는 환경에서는 처음 실행이 막힐 수 있다.

즉 자주 고정적으로 쓰는 도구라면 설치형이 더 단순할 수 있지만, 설치 자체가 문제인 상황이라면 `npx` 방식이 꽤 실용적입니다.

## 6. CI/CD나 비대화형 환경에서는 한 가지 더

환경에 따라 `npx`가 처음 실행될 때 확인 프롬프트를 띄울 수 있습니다.
자동화 환경이라면 아래처럼 `-y`를 붙여두는 편이 안전합니다.

```json
{
  "scripts": {
    "dev": "npx -y concurrently \"npm run app1\" \"npm run app2\""
  }
}
```

로컬 개발에서는 보통 없어도 되지만, CI 파이프라인에서는 이런 차이가 꽤 중요합니다.

## 7. Nx를 쓰고 있다면 굳이 concurrently가 필요 없을 수도 있다

Nx 프로젝트라면 `concurrently` 대신 Nx 자체 병렬 실행 기능을 쓰는 편이 더 자연스러울 때도 많습니다.

```bash
nx run-many --target=serve --projects=app1,app2 --parallel
```

이 방식의 장점:

- 추가 패키지 없이 병렬 실행 가능
- Nx workspace 구조와 더 잘 맞음
- 캐시와 태스크 실행 흐름을 일관되게 가져가기 쉬움

즉 Nx 환경에서는 먼저 `run-many --parallel`로 해결 가능한지 보는 편이 좋고, 단순히 여러 npm script를 한 번에 묶고 싶을 때는 `npx concurrently`가 빠른 선택지가 됩니다.

## 8. 정리

`concurrently` 설치가 실패하거나 dependency 충돌 때문에 프로젝트를 건드리기 부담스럽다면, 가장 먼저 시도해볼 수 있는 방법은 아래 한 줄입니다.

```json
{
  "scripts": {
    "dev": "npx concurrently \"npm run app1\" \"npm run app2\""
  }
}
```

핵심은 설치형 도구를 꼭 설치형으로만 써야 하는 것은 아니라는 점입니다.
특히 Nx + Vite처럼 의존성 구조가 이미 복잡한 프로젝트에서는 `npx` 하나로 문제를 빠르게 우회하는 것이 실무적으로 더 효율적일 때가 많습니다.
이번에 다시 보면서도 개발 환경 이슈는 "근본 해결"을 바로 하려 하기보다 흐름을 먼저 살리는 선택지가 있는지 보는 편이 더 현실적이라는 점을 메모해두게 됐습니다.

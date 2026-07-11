---
title: Lambda 콜드스타트가 이제 요금에도 잡힌다, INIT 과금과 SnapStart
date: 2026-06-09 16:02:00 +09:00
categories: [Infra]
tags: [aws, lambda, cold-start, snapstart]
description: 온디맨드 Lambda의 INIT 단계가 과금 대상에 포함되면서 무거운 top-level import가 지연과 비용을 동시에 키우는 문제와 지연 로딩·SnapStart 대응을 정리했다.
---

Lambda 비용 구조 변화를 공부했습니다. 관리형 런타임 온디맨드 함수의 INIT(초기화) 시간이 과금 대상(Billed Duration)에 포함되면서, 콜드스타트가 성능 문제이자 비용 문제가 됐습니다.

## INIT에서 무슨 일이 벌어지나

INIT은 핸들러 실행 전, 런타임이 코드를 로드하고 top-level(핸들러 밖) 코드를 실행하는 구간입니다. 무거운 SDK와 라이브러리를 전부 상단에서 import하면 이 구간이 길어집니다.

```javascript
// 나쁨: top-level에서 전부 로드 → INIT이 길다
import { S3Client } from "@aws-sdk/client-s3";
import pdf from "pdf-lib";           // 무겁고, 특정 경로에서만 쓰임
import sharp from "sharp";

export const handler = async (e) => { ... };
```

INIT이 1.5초면 콜드스타트마다 1.5초의 지연 + 그만큼의 과금이 붙습니다.

## 대응 1: 지연 로딩

실제로 쓰는 경로에서만, 첫 호출 때 한 번 로드합니다.

```javascript
let sharp;
export const handler = async (e) => {
  if (e.type === "thumbnail") {
    sharp ??= (await import("sharp")).default;  // 필요할 때 1회 로드
    return makeThumb(sharp, e);
  }
  ...
};
```

번들러(esbuild)로 트리셰이킹해 패키지 자체를 다이어트하는 것도 같은 축의 개선입니다.

## 대응 2: SnapStart

초기화가 끝난 실행 환경의 스냅샷을 미리 떠 두고, 콜드스타트 때 스냅샷을 복원하는 기능입니다. INIT을 "실행"하지 않고 "복원"하므로 콜드스타트가 수초에서 수백 ms 이하로 내려가고, INIT 실행 과금도 사라집니다. 스냅샷 복원 특성상 초기화 시점의 랜덤 시드·커넥션 상태가 복제될 수 있어, 유니크해야 하는 값은 핸들러에서 재생성하는 주의가 필요합니다.

## 정리

- INIT 과금 이후 top-level 무거운 import는 지연과 비용을 이중으로 낸다.
- 조건부로 쓰는 무거운 의존성은 지연 로딩으로 내린다.
- 콜드스타트 민감 함수는 SnapStart(또는 provisioned concurrency)로 구조 자체를 바꾼다.

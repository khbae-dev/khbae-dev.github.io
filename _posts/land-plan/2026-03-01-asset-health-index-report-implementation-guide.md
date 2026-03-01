---
title: "Asset Health Index 리포트 구현 가이드 (Grandview Reportlet)"
date: 2026-03-01 12:30:00 +09:00
tags: [frontend, grandview, reportlet, report, asset-health-index]
description: "@grandview 기반에서 Asset Health Index 리포트를 설계, 구현, 연동, 테스트, 배포까지 완료하는 실전 기술 문서."
---

# Asset Health Index 리포트 구현 가이드 (Grandview Reportlet)

이 문서는 [`_posts/land-plan/2026-03-01-frontend-grandview-integration-reference.md`](./2026-03-01-frontend-grandview-integration-reference.md) 내용을 전제로,
**Asset Health Index를 보여주는 리포트(Reportlet Plugin)**를 실제로 개발해서 운영 배포까지 끝내는 절차를 정리한 실무 가이드입니다.

핵심 목표:
- Studio Report Builder에서 Asset Health Index 리포틀릿을 선택할 수 있어야 함
- 리포트 페이지/프린트 페이지에서 동일하게 렌더링되어야 함
- 개발(local)과 운영(prod) 경로 모두에서 plugin 로딩이 안정적으로 동작해야 함

---

## 1. 완료 기준 (Definition of Done)

아래 6개가 모두 충족되면 완료입니다.

1. Studio에서 신규 리포틀릿이 목록에 보인다.
2. 리포트에 배치 시 Asset Health Index 데이터가 정상 조회된다.
3. data/style form 수정 시 화면이 즉시 반영된다.
4. Report 앱 조회 및 Print 앱 출력에서 동일하게 렌더링된다.
5. backend `/plugins` 응답 또는 `plugin.dev.json`으로 remote가 안정적으로 로드된다.
6. `nx build` 산출물 기준으로 배포 경로와 `plugin-info`가 일치한다.

---

## 2. 구현 전략

Asset Health Index 리포트를 만드는 방법은 2가지입니다.

### 전략 A (권장): 기존 APM reportlet 재사용 + 커스터마이징
- `apm-default-reportlets`를 베이스로 신규 플러그인 생성
- 기존 `asset-health-index` 계열 컴포넌트를 재사용
- 장점: 빠르고 실패 확률 낮음

### 전략 B: 완전 신규 reportlet 컴포넌트 작성
- API/차트/폼을 모두 직접 구현
- 장점: 자유도 높음
- 단점: 개발량/검증량 증가

이 문서는 **전략 A** 기준으로 설명합니다. (실무에서 가장 빠름)

---

## 3. 사전 준비

### 3-1. 버전/환경
- Node.js `20.11.0`
- Nx `18.0.5`
- React `18.2.0`
- Webpack `5.95.0`

### 3-2. 확인할 기존 구조
- Host 앱: `apps/micro-apps/studio`, `apps/micro-apps/report`, `apps/micro-apps/print`
- 기존 reportlet plugin:
  - `plugins/reportlet/apm-default-reportlets`
  - `plugins/reportlet/seed-default-reportlets`

### 3-3. 필수 이해 포인트
- plugin 로딩 계약: `plugin-init`, `plugin-info`, `plugin-config-data-forms`, `plugin-config-style-forms`
- master config 핵심: `id`, `path`, `config.data`, `config.style`
- path 규칙: `master.path === module federation expose key`

---

## 4. 프로젝트 생성 (복제 방식)

가장 안정적인 방법은 기존 plugin을 복제해서 시작하는 것입니다.

```bash
cp -R plugins/reportlet/apm-default-reportlets plugins/reportlet/asset-health-reportlets
```

복제 후 아래를 변경합니다.

1. `project` 이름(예: `asset-health-reportlets`)
2. module federation `name`
3. `src/lib/plugin-config.ts`의 `name`, `pluginPath`
4. `src/package.js` plugin metadata
5. 불필요한 컴포넌트/master config 제거

권장 pluginPath 예시:
- `/plugins/apm/reportlet/asset-health-reportlets`

---

## 5. Module Federation 구성

파일: `plugins/reportlet/asset-health-reportlets/module-federation.config.ts`

최소 expose 구성:

```ts
const config = {
  name: 'asset-health-reportlets',
  exposes: {
    'plugin-init': './src/lib/plugin-init.ts',
    'plugin-info': './src/package.js',
    'plugin-config-data-forms': './src/lib/config-forms/config-data.index.ts',
    'plugin-config-style-forms': './src/lib/config-forms/config-style.index.ts',
    'asset-health-index-reportlet': './src/lib/components/asset-health-index-reportlet/index.ts',
  },
};

export default config;
```

주의:
- expose key(`asset-health-index-reportlet`)는 master `path`와 반드시 같아야 합니다.

---

## 6. plugin-init / plugin-config 구현

### 6-1. plugin-config
파일: `src/lib/plugin-config.ts`

```ts
export const reportletPluginConfig = {
  name: 'asset-health-reportlets',
  pluginPath: '/plugins/apm/reportlet/asset-health-reportlets',
};
```

### 6-2. plugin-init
파일: `src/lib/plugin-init.ts`

```ts
import { IReportletPluginConfig } from '@grandview/shared';
import { getMasterConfigById, getMasterConfigGroups } from './config/master-configs';

export default function initPlugin(): IReportletPluginConfig {
  return {
    pluginName: 'asset-health-reportlets',
    masterConfigById: getMasterConfigById,
    masterConfigGroups: getMasterConfigGroups,
  };
}
```

---

## 7. Asset Health Index Master Config

파일: `src/lib/config/master-configs.ts`

핵심은 `id/path`와 `data/style` 기본값입니다.

```ts
const ASSET_HEALTH_INDEX_MASTER_ID = 'asset-health-index-reportlet';

export const assetHealthIndexMasterConfig = {
  id: ASSET_HEALTH_INDEX_MASTER_ID,
  path: 'asset-health-index-reportlet',
  name: 'Asset Health Index',
  description: '자산 건강지수(Health Index) 추이/요약 리포틀릿',
  imagePath: 'asset-health-index-reportlet.png',
  config: {
    w: 12,
    h: 6,
    data: [
      {
        id: 'asset-health-index-data',
        values: {
          assetId: null,
          timeRange: 'last-7d',
          aggregation: 'avg',
          showThreshold: true,
        },
      },
    ],
    style: [
      {
        id: 'asset-health-index-style',
        values: {
          showTitle: true,
          showLegend: true,
          lineWidth: 2,
          goodColor: '#18A058',
          warningColor: '#F0A020',
          criticalColor: '#D03050',
        },
      },
    ],
  },
};
```

그룹 등록 예시:

```ts
export const getMasterConfigGroups = () => [
  {
    id: 'asset-health-group',
    name: 'Asset Health',
    items: ['asset-health-index-reportlet'],
  },
];
```

---

## 8. 컴포넌트 구현

파일: `src/lib/components/asset-health-index-reportlet/asset-health-index-reportlet.tsx`

구현 포인트:
- 입력: `data/style config`, `tabKey`
- 조회: 기존 APM domain hook 또는 공통 `httpService` 사용
- 출력: `GVReportlet`, 차트 컴포넌트(`@grandview/chart-echarts` 계열)

예시 스켈레톤:

```tsx
import { useMemo } from 'react';
import { GVReportlet, GVReportletHeader, GVReportletBody } from '@grandview/component';

export function AssetHealthIndexReportlet(props: any) {
  const { reportletConfig } = props;
  const dataConfig = reportletConfig?.data?.[0]?.values || {};
  const styleConfig = reportletConfig?.style?.[0]?.values || {};

  const queryParams = useMemo(
    () => ({
      assetId: dataConfig.assetId,
      range: dataConfig.timeRange,
      aggregation: dataConfig.aggregation,
    }),
    [dataConfig.assetId, dataConfig.timeRange, dataConfig.aggregation],
  );

  // TODO: query hook 연결
  // const { data, isLoading, error } = useAssetHealthIndexQuery(queryParams);

  return (
    <GVReportlet>
      <GVReportletHeader title="Asset Health Index" />
      <GVReportletBody>
        {/* TODO: 차트/요약 렌더링 */}
      </GVReportletBody>
    </GVReportlet>
  );
}
```

실무 팁:
- 최초 버전은 "선 그래프 1개 + 현재 값 카드"만 구현하고, 이후 기능 확장
- `assetId` 미선택 상태(empty state) 처리 필수
- print 모드에서 애니메이션/툴팁 의존 제거

---

## 9. Data/Style Form 구현

Report Builder에서 설정 가능해야 운영 편의성이 올라갑니다.

### 9-1. data form (필수)
권장 필드:
- `assetId`
- `timeRange`
- `aggregation`
- `showThreshold`

변경 이벤트:
- `sendDirtyReportletDataConfig`

### 9-2. style form (권장)
권장 필드:
- `showTitle`, `showLegend`
- `lineWidth`
- `good/warning/critical` 색상

변경 이벤트:
- `sendDirtyReportletStyleConfig`

### 9-3. 라우팅
- `plugin-config-data-forms`에서 `dataId`로 form 분기
- `plugin-config-style-forms`에서 `styleGroupId`로 form 분기

---

## 10. Studio/Report Host 연동

### 10-1. 개발 모드 (`plugin.dev.json`)
`apps/micro-apps/studio/src/assets/config/plugin.dev.json` 및
`apps/micro-apps/report/src/assets/config/plugin.dev.json`에 remote 등록

예시:

```json
{
  "asset-health-reportlets": {
    "name": "asset-health-reportlets",
    "typeCode": "REPORTLET",
    "remoteEntry": "http://localhost:7011/remoteEntry.js"
  }
}
```

### 10-2. 운영 모드 (backend plugin)
backend `/podo/api/plugins?activating=true&typeCodes=REPORTLET&size=100` 응답에
다음이 맞아야 합니다.

- `name`: `asset-health-reportlets`
- `pluginPath`: 배포 경로와 일치
- `activating`: `true`

---

## 11. 실행/검증 시나리오

### 11-1. 로컬 실행
1. studio 실행
2. report 실행
3. 신규 reportlet plugin 실행

(프로젝트 스크립트에 맞춰 명령 구성)

### 11-2. 기능 검증
1. Studio > Report Builder 진입
2. `Asset Health` 그룹에서 `Asset Health Index` 선택
3. asset/timeRange 변경 시 데이터 갱신 확인
4. style 변경 반영 확인
5. 저장 후 Report 앱 조회 확인
6. Print 앱 렌더 확인

### 11-3. 회귀 체크
- 기존 reportlet 목록/로드 성능 영향 없음
- 기존 plugin 충돌 없음(`id`, `path`, i18n namespace)

---

## 12. 빌드/배포

### 12-1. 빌드
- `npx nx build asset-health-reportlets`

### 12-2. 산출물 확인
- `remoteEntry.js` 생성 여부
- `plugin-info`의 `pluginPath`와 실제 배포 폴더 일치 여부

### 12-3. 배포 후 확인
- backend plugin 목록 노출 확인
- Studio/Report/Print 실제 사용자 권한에서 렌더 확인

---

## 13. 장애 포인트와 해결

### 문제 1) Builder에서 목록은 보이는데 생성이 안 됨
원인:
- `master.path`와 MF expose key 불일치

해결:
- 두 값을 동일하게 맞춤

### 문제 2) 개발에서는 되는데 운영에서 로딩 실패
원인:
- `pluginPath`와 실제 배포 경로 불일치

해결:
- `src/package.js` / `plugin-config.ts` / 서버 배포 경로 재검증

### 문제 3) 설정폼 변경이 즉시 반영되지 않음
원인:
- dirty 이벤트 미전송

해결:
- data/style form에서 각각 dirty event 호출 확인

### 문제 4) print 화면 깨짐
원인:
- interactive 옵션(tooltip/animation) 의존

해결:
- print 모드 조건에서 정적 옵션으로 강제

---

## 14. 권장 개발 순서 (압축 버전)

1. `apm-default-reportlets` 복제
2. Asset Health Index 1개만 남기고 정리
3. `module-federation.config.ts` expose 최소화
4. master config `id/path/data/style` 확정
5. data/style form 연결
6. `plugin.dev.json` 연동으로 Studio 확인
7. Report/Print 확인
8. build 후 backend plugin 등록
9. 운영 검증

---

## 15. 체크리스트

배포 직전 최종 확인:

- [ ] `pluginName`, `pluginPath`, backend plugin name 일치
- [ ] `master.id` 유니크
- [ ] `master.path === expose key`
- [ ] data/style form 라우팅 정상
- [ ] i18n namespace 충돌 없음
- [ ] studio/report/print 모두 렌더 확인
- [ ] local + prod 둘 다 로딩 확인

---

## 16. 마무리

Asset Health Index 리포트 개발의 핵심은 **"계약 일치"**입니다.

- module federation expose
- master config path
- plugin metadata/pluginPath
- backend plugin 응답

이 네 가지가 일치하면, 나머지는 컴포넌트 구현 품질의 문제로 좁혀집니다.
초기 버전은 단순하게 만들고(data 3개, style 3개 수준), 운영 적용 후 점진 확장하는 전략을 권장합니다.

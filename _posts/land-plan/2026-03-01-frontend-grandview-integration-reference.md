---
title: Frontend 전체 소스 + @grandview 통합 기술 문서
date: 2026-03-01 10:00:00 +09:00
tags: [frontend, architecture, module-federation, grandview, reference]
description: application-seed/frontend와 @grandview 패키지 통합 관점에서 설계, 구현, 테스트, 배포 기준을 정리한 실무 레퍼런스 문서.
published: false
---

# Frontend 전체 소스 + `@grandview` 통합 기술 문서

이 문서는 현재 저장소(`application-seed/frontend`)의 **실제 소스 구조/런타임/개발 방식**과 `node_modules/@grandview`에서 제공되는 기능을 통합한 **실무용 레퍼런스**입니다.

목표:
- 이 파일만으로도 다른 저장소에서 앱/페이지/위젯/리포틀릿을 설계·구현·테스트·배포할 수 있도록 한다.
- `@grandview` 패키지의 제공 기능(엔트리 export 기준)을 빠짐없이 찾을 수 있도록 한다.

---

## 1. 문서 범위와 전제

### 1-1. 범위
- Monorepo(Nx) 기반 프론트엔드 전체:
  - `apps/` (Portal + Micro Apps)
  - `plugins/` (Widget/Reportlet/Testbed)
  - `libs/config/` (사이트 커스터마이징/기본 설정 레이어)
- 런타임 핵심 기술:
  - Module Federation 기반 Plugin 동적 로딩
  - Runtime Theme (`theme-color-level.json`, `theme-podod.json`, `theme-antd.json`)
  - i18n, config.json, environment.json
- `@grandview/*` 패키지 기능 카탈로그 (엔트리 export 기준 전체 목록)

### 1-2. 전제 버전
- Node.js: `20.11.0` 권장
- Nx: `18.0.5`
- React: `18.2.0`
- Webpack: `5.95.0`
- Vite: `5.0.x` (주로 `libs/config/*` 패키지 빌드)

### 1-3. 용어
- Host Application: `dashboard`, `report`, `studio`, `print` 등 Plugin을 로드하는 앱
- Remote Plugin: `apm-default-widgets`, `seed-default-reportlets` 등
- Widget Plugin: Dashboard Builder에서 사용하는 컴포넌트 번들
- Reportlet Plugin: Report Builder에서 사용하는 컴포넌트 번들

---

## 2. 전체 아키텍처 (핵심)

### 2-1. 구조 요약
- Portal(`apps/portal/web`)이 최상위 진입점 역할
- Micro App(`apps/micro-apps/*`)들이 개별 도메인 UI 담당
- Plugin(`plugins/widget/*`, `plugins/reportlet/*`)은 MF Remote로 동적 로딩
- 공통 도메인/컴포넌트/서비스는 `@grandview/*` 패키지에서 제공

### 2-2. 런타임 공통 부트스트랩 패턴 (`main.ts`)
대부분 앱은 아래 순서로 동일하게 초기화합니다.

1. `getMicroAppConfig(...)`로 앱/테마/환경 config 로딩
2. `window.GV_CONFIG`, `GV_THEME_APPLICATION|PORTAL`, `GV_PRODUCTION` 세팅
3. `applyRuntimeTheme(...)` 적용
4. `initI18N(...)`, `initTimeLocale()`, `initDayjsPlugins()`
5. (Host 앱인 경우) Plugin 초기화
   - `setPluginCacheKey(...)`
   - `setPluginDev(...)` (개발 모드)
   - `initRemoteDefinitionsAndPlugins(...)` 또는 `initPlugins()`

### 2-3. Plugin 로딩 핵심
- Backend API:
  - Widget: `GET /podo/api/plugins?activating=true&typeCodes=WIDGET&size=100`
  - Reportlet: `GET /podo/api/plugins?activating=true&typeCodes=REPORTLET&size=100`
  - Studio: `typeCodes=WIDGET,REPORTLET`
- 동작:
  - Backend 응답 + `plugin.dev.json`를 바탕으로 remoteEntry URL 결정
  - `plugin-init` 모듈 호출 후 `masterConfigs`, `masterConfigGroups`, `masterConfigById` 등록
  - Builder가 이 master 정보를 읽어 위젯/리포틀릿 목록과 인스턴스 생성

---

## 3. 소스 트리 전수 맵

### 3-1. Nx 프로젝트 인벤토리
| 프로젝트 | 유형 | sourceRoot | build output | serve port |
|---|---|---|---|---|
| `ameta` | application | `apps/micro-apps/ameta/src` | `dist/apps/portal/public/ameta` | - |
| `apm` | application | `apps/micro-apps/apm/src` | `dist/apps/portal/public/apm` | - |
| `dashboard` | application | `apps/micro-apps/dashboard/src` | `dist/apps/portal/public/dashboard` | 7002 |
| `pipeline` | application | `apps/micro-apps/pipeline/src` | `dist/apps/portal/public/pipeline` | - |
| `print` | application | `apps/micro-apps/print/src` | `dist/apps/portal/public/print` | 7004 |
| `report` | application | `apps/micro-apps/report/src` | `dist/apps/portal/public/report` | 7003 |
| `studio` | application | `apps/micro-apps/studio/src` | `dist/apps/portal/public/studio` | 7001 |
| `system` | application | `apps/micro-apps/system/src` | `dist/apps/portal/public/system` | - |
| `user` | application | `apps/micro-apps/user/src` | `dist/apps/portal/public/user` | - |
| `portal` | application | `apps/portal/web/src` | `dist/apps/portal/public` | 7000 |
| `config-apm` | library | `libs/config/apm/src` | - | - |
| `config-application` | library | `libs/config/application/src` | - | - |
| `config-dashboard` | library | `libs/config/dashboard/src` | - | - |
| `config-icon` | library | `libs/config/icon/src` | - | - |
| `config-layout` | library | `libs/config/layout/src` | - | - |
| `config-report` | library | `libs/config/report/src` | - | - |
| `apm-default-reportlets` | application | `plugins/reportlet/apm-default-reportlets/src` | `dist/apps/portal/public/plugins/apm/reportlet/apm-default-reportlets` | 7002 |
| `seed-default-reportlets` | application | `plugins/reportlet/seed-default-reportlets/src` | `dist/apps/portal/public/plugins/seed/reportlet/seed-default-reportlets` | 7002 |
| `testbed` | application | `plugins/testbed/src` | `dist/plugins/testbed` | 7000 |
| `apm-default-widgets` | application | `plugins/widget/apm-default-widgets/src` | `dist/apps/portal/public/plugins/apm/widget/apm-default-widgets` | 7001 |
| `seed-default-widgets` | application | `plugins/widget/seed-default-widgets/src` | `dist/apps/portal/public/plugins/seed/widget/seed-default-widgets` | 7001 |

### 3-2. 앱별 메뉴/페이지 매핑 (`sideBarMenus`)
### ameta
- `ameta.asset-config-wizard` -> `./pages/asset-config-wizard-page/asset-config-wizard-page`
- `ameta.asset-hierarchy` -> `./pages/asset-hierarchy-page/asset-hierarchy-page`
- `ameta.parameter` -> `./pages/parameter-page/parameter-page`
- `ameta.parameter-group` -> `./pages/parameter-group-page/parameter-group-page`
- `ameta.asset-template` -> `./pages/asset-template-page/asset-template-page`
- `ameta.asset` -> `./pages/asset-page/asset-page`
- `ameta.asset-group` -> `./pages/asset-group-page/asset-group-page`
- `ameta.context` -> `./pages/context-page/context-page`
- `ameta.model` -> `./pages/model-page/model-page`
- `ameta.alarm-rule` -> `./pages/alarm-rule-page/alarm-rule-page`

### apm
- `apm.asset-list` -> `./pages/asset-list-page/asset-list-page`
- `apm.detail` -> `./pages/asset-detail-page/asset-detail-page`
- `apm.alarm-list` -> `./pages/alarm-group-list-page/alarm-group-list-page`
- `apm.alarm-detail` -> `./pages/alarm-group-detail-page/alarm-group-detail-page`
- `apm.work-order-list` -> `./pages/work-order-list-page/work-order-list-page`
- `apm.work-order-detail` -> `./pages/work-order-detail-page/work-order-detail-page`

### pipeline
- `pipeline.edge-flow` -> `./pages/edge-flow-page/edge-flow-page`

### report
- `report.history` -> `./pages/history-page/history-page`
- `report.on-demand` -> `./pages/on-demand-page/on-demand-page`

### studio
- `studio.dashboard` -> `./pages/dashboard-page/dashboard-page`
- `studio.layout` -> `./pages/layout-page/layout-page`
- `studio.report` -> `./pages/report-page/report-page`

### system
- `system.global-config` -> `./pages/global-config-page/global-config-page`
- `system.user` -> `./pages/user-page/user-page`
- `system.user-group` -> `./pages/user-group-page/user-group-page`
- `system.permission` -> `./pages/permission-page/permission-page`
- `system.datasource` -> `./pages/datasource-page/datasource-page`
- `system.license` -> `./pages/license-page/license-page`
- `system.plugin-store` -> `./pages/plugin-store-page/plugin-store-page`

### user
- `user.profile` -> `./pages/my-profile/my-profile-page`

### print
- sidebar menu 없음 (print 전용 단일 페이지)
- 엔트리: `apps/micro-apps/print/src/app/pages/print-page/print-page.tsx`

### 3-3. `libs/config/*` 역할

- `libs/config/application`
  - Portal Header/User/Login/Menu 컴포넌트 교체 포인트
  - `useInitDashboardApplicationSideMenu`, `useInitApplicationSideMenu`, `useInitApplicationSharedData`
  - 앱 공통 초기화에서 사이드바 메뉴/공통 타입/상태를 준비
- `libs/config/icon`
  - `getIcon(menuKey)` 기반 메뉴 아이콘 매핑
  - Dashboard 위젯 아이콘 매핑(`transferDashboardIcon`) 제공
- `libs/config/layout`
  - Layout Builder element 기본 config 생성
  - `getElementComponent`, `getElementMasterConfig`, `getImageElementConfig`
- `libs/config/report`
  - Report 기본 마스터 설정, 기본 페이지/리포틀릿 템플릿 구성
  - `getReportMasterConfig`, `getPagesForDefaultTemplate`, `getReportletsForDefaultTemplate`
- `libs/config/apm`
  - APM 상세 Overview 위젯 구성
  - 데이터 서브탭 구성
- `libs/config/dashboard`
  - 사이트 커스텀 Dashboard 필터 확장 포인트

---

## 4. 개발 표준: App/Page 개발

### 4-1. 신규 페이지 추가 표준
1. `src/app/pages/...`에 페이지 컴포넌트 생성 (default export)
2. 도메인 패키지 pagelet 래핑
3. `src/app/sidebar-menu.ts`에 menu key + loadable 등록
4. backend `/podo/api/menus`에 동일 key 등록
5. 필요 시 `hidden`, `sidebarMenuType: DYNAMIC`, `createTabType` 설정

기본 예시:

```tsx
import { memo } from 'react';
import { PageletProps } from '@grandview/shared';
import { GVAssetListPagelet } from '@grandview/web-domain-apm';

function AssetListPage(props: PageletProps) {
  return <GVAssetListPagelet {...props} />;
}

export default memo(AssetListPage);
```

### 4-2. 메뉴/라이선스 연동 필수
- Frontend key가 있어도 backend 메뉴 응답에 없으면 UI 미노출
- `/podo/api/applications/license/check`에서 `contextPath`별 `licenseCheckResult=true` 필요

---

## 5. 개발 표준: Widget Plugin

### 5-1. 필수 파일 구성
- `module-federation.config.ts`
  - 최소: `plugin-init`, `plugin-info`, 그리고 비즈 모듈 expose
- `src/lib/plugin-init.ts`
  - `IWidgetPluginConfig` 반환
- `src/lib/plugin-config.ts`
  - `{ name, pluginPath }`
- `src/lib/config/master-configs.ts`
  - `getWidgetMasterConfigs`, `getWidgetMasterConfigGroups`
- `src/package.js`
  - plugin package metadata 제공 함수

### 5-2. `plugin-init` 계약

```ts
import { IWidgetPluginConfig } from '@grandview/shared';

export default function initPlugin(): IWidgetPluginConfig {
  return {
    pluginName: 'my-widgets',
    masterConfigs: getWidgetMasterConfigs,
    masterConfigGroups: getWidgetMasterConfigGroups,
  };
}
```

### 5-3. Widget Master Config 핵심 필드
- `id`, `path`, `name`, `description`, `images`
- `type`: builder 분류 (Asset/Location/Alarm...)
- `config`:
  - `width`, `height`
  - `events.in/out`
  - `enableLink`, `links`
  - `enableHierarchy`, `hierarchies`

### 5-4. 컴포넌트 구현 패턴
- props: `IWidgetProps` + `tabKey`
- 상태/이벤트
  - `useSubscribeData`, `sendDataToSubscriber`
  - `useTableActionState`, `useChangedRealtimeFilterCondition`
- 데이터
  - `useRealtimeFilter`로 필터 수집
  - `useQuery` + `queryOptions(isRealtime, timeCycle)`
- 표시
  - `@grandview/component`의 `GVWidget`, `GVTable`, `GVWidgetTitle` 등 사용

### 5-5. i18n
- 공통 위젯 i18n: `initSharedWidgetPackageI18n()`
- 플러그인 스코프 i18n: 별도 `initXXXWidgetPackageI18n()`
- `useTranslation(I18N_NAMESPACE)` 권장

---

## 6. 개발 표준: Reportlet Plugin

### 6-1. 필수 파일 구성
- `module-federation.config.ts`
  - 최소: `plugin-init`, `plugin-info`
  - 권장: `plugin-config-data-forms`, `plugin-config-style-forms`
  - 리포틀릿 모듈 expose
- `src/lib/plugin-init.ts`
  - `IReportletPluginConfig` 반환
- `src/lib/config/master-configs.ts`
  - `getMasterConfigById`, `getMasterConfigGroups(baseType)`
- `src/lib/config-forms/*`
  - data/style form 라우팅

### 6-2. `plugin-init` 계약

```ts
import { IReportletPluginConfig } from '@grandview/shared';

export default function initPlugin(): IReportletPluginConfig {
  return {
    pluginName: 'my-reportlets',
    masterConfigById: getMasterConfigById,
    masterConfigGroups: getMasterConfigGroups,
  };
}
```

### 6-3. Reportlet Master Config 핵심 필드
- `id`, `path`, `name`, `description`, `imagePath`
- `config`:
  - `w`, `h`
  - `data`: `IReportletDataGroup[]`
  - `style`: `IReportletStyleGroup[]`

### 6-4. 설정 폼 확장
- data form 분기: `dataId` 기반
- style form 분기: `styleGroupId` 기반
- 변경 이벤트 전송:
  - `sendDirtyReportletDataConfig`
  - `sendDirtyReportletStyleConfig`

### 6-5. 공통 스타일 그룹
- `titleStyleGroup`
- `textStyleGroup`
- `borderStyleGroup`
- `lineStyleGroup`
- `boxStyleGroup`
- `commonStyleGroup`
- `printInfoStyleGroup`

---

## 7. Plugin Module Federation 상세

### 7-1. 실제 expose 목록
### `plugins/widget/apm-default-widgets/module-federation.config.ts`
- `plugin-init` -> `./src/lib/plugin-init.ts`
- `plugin-info` -> `./src/package.js`
- `default-layout` -> `./src/lib/components/layouts/default-layout-widget/index.ts`
- `apm-asset-status-heatmap` -> `./src/lib/components/assets/asset-health-heatmap-widget/index.ts`
- `apm-asset-status-summary` -> `./src/lib/components/assets/asset-status-summary-widget/index.ts`
- `apm-asset-list` -> `./src/lib/components/assets/asset-health-list-widget/index.ts`
- `apm-asset-parameters` -> `./src/lib/components/assets/asset-parameters-widget/index.ts`
- `apm-geo-status-summary` -> `./src/lib/components/maps/geo-status-summary-widget/index.ts`
- `apm-locations-status-summary` -> `./src/lib/components/maps/locations-status-summary-widget/index.ts`
- `apm-kpi-status-summary` -> `./src/lib/components/maps/kpi-status-summary-widget/index.ts`
- `apm-alarm-type` -> `./src/lib/components/alarms/alarm-type-widget/index.ts`
- `apm-alarm-timeline` -> `./src/lib/components/alarms/alarm-heatmap-widget/index.ts`
- `apm-alarm-severity` -> `./src/lib/components/alarms/alarm-severity-widget/index.ts`
- `apm-alarm-count-trend` -> `./src/lib/components/alarms/alarm-count-trend-widget/index.ts`
- `apm-alarm-list` -> `./src/lib/components/alarms/alarm-list-widget/index.ts`
- `apm-sensor-overall-status` -> `./src/lib/components/sensors/sensor-overall-status-widget/index.ts`
- `apm-sensor-summary-status` -> `./src/lib/components/sensors/sensor-summary-status-widget/index.ts`
- `apm-sensor-worst-top5` -> `./src/lib/components/sensors/asset-sensor-worst-top5-widget/index.ts`
- `apm-sensor-asset-collect-rate-list` -> `./src/lib/components/sensors/asset-sensor-collect-rate-list-widget/index.ts`

### `plugins/widget/seed-default-widgets/module-federation.config.ts`
- `plugin-init` -> `./src/lib/plugin-init.ts`
- `plugin-info` -> `./src/package.js`
- `seed-asset-list` -> `./src/lib/components/asset-health-list-widget/index.ts`

### `plugins/reportlet/apm-default-reportlets/module-federation.config.ts`
- `plugin-init` -> `./src/lib/plugin-init.ts`
- `plugin-info` -> `./src/package.js`
- `plugin-config-data-forms` -> `./src/lib/config-forms/config-data.index.ts`
- `plugin-config-style-forms` -> `./src/lib/config-forms/config-style.index.ts`
- `line-reportlet` -> `./src/lib/components/common/line/index.ts`
- `text-reportlet` -> `./src/lib/components/common/text/index.ts`
- `box-reportlet` -> `./src/lib/components/common/box/index.ts`
- `print-info-reportlet` -> `./src/lib/components/common/print-info/index.ts`
- `asset-information` -> `./src/lib/components/asset-base/asset/asset-info-reportlet/index.ts`
- `asset-image` -> `./src/lib/components/asset-base/asset/asset-image-reportlet/index.ts`
- `asset-models-information` -> `./src/lib/components/asset-base/asset/asset-models-info-reportlet/index.ts`
- `asset-alarm-summary` -> `./src/lib/components/asset-base/asset/asset-alarm-summary-reportlet/index.ts`
- `rul-trend` -> `./src/lib/components/asset-base/asset/asset-rul-reportlet/index.ts`
- `health-index-trend` -> `./src/lib/components/asset-base/asset/asset-health-index-reportlet/index.ts`
- `running-trend` -> `./src/lib/components/asset-base/asset/asset-running-reportlet/index.ts`
- `parameter-charts` -> `./src/lib/components/asset-base/parameter/parameter-charts-reportlet/index.ts`
- `parameter-worst-perf-top5-trend` -> `./src/lib/components/asset-base/parameter/worst-parameter-trend-reportlet/index.ts`
- `parameter-worst-perf-top5-rate` -> `./src/lib/components/asset-base/parameter/worst-parameter-rate-reportlet/index.ts`
- `parameter-worst-perf-top5-chart` -> `./src/lib/components/asset-base/parameter/worst-parameter-chart-reportlet/index.ts`
- `alarm-total-by-asset` -> `./src/lib/components/asset-base/alarm/asset-alarm-total-reportlet/index.ts`
- `alarm-type-by-asset` -> `./src/lib/components/asset-base/alarm/asset-alarm-type-reportlet/index.ts`
- `alarm-acknowledgement-status-by-asset` -> `./src/lib/components/asset-base/alarm/asset-alarm-status-reportlet/index.ts`
- `alarm-quality-by-asset` -> `./src/lib/components/asset-base/alarm/asset-alarm-quality-reportlet/index.ts`
- `asset-status-summary` -> `./src/lib/components/location-base/asset/asset-status-summary-reportlet/index.ts`
- `asset-status-count` -> `./src/lib/components/location-base/asset/asset-status-count-reportlet/index.ts`
- `asset-health-heatmap` -> `./src/lib/components/location-base/asset/asset-health-heatmap-reportlet/index.ts`
- `asset-health-list` -> `./src/lib/components/location-base/asset/asset-health-list-reportlet/index.ts`
- `location-information` -> `./src/lib/components/location-base/location/location-info/index.ts`
- `location-alarm-heatmap-timeline` -> `./src/lib/components/location-base/alarm/alarm-heatmap-timeline-reportlet/index.ts`
- `sensor-connection-status` -> `./src/lib/components/location-base/sensor/sensor-connection-status-reportlet/index.ts`
- `sensor-summary-status` -> `./src/lib/components/location-base/sensor/sensor-summary-status-reportlet/index.ts`
- `sensor-worst-perf-top5` -> `./src/lib/components/location-base/sensor/sensor-worst-top5-reportlet/index.ts`
- `kpi-summary` -> `./src/lib/components/shared-base/kpi/kpi-summary-reportlet/index.ts`
- `kpi-map` -> `./src/lib/components/shared-base/kpi/kpi-map-reportlet/index.ts`
- `kpi-list` -> `./src/lib/components/shared-base/kpi/kpi-list-reportlet/index.ts`
- `alarm-type` -> `./src/lib/components/shared-base/alarm/alarm-type-reportlet/index.ts`
- `alarm-severity` -> `./src/lib/components/shared-base/alarm/alarm-severity-reportlet/index.ts`
- `alarm-count-trend` -> `./src/lib/components/shared-base/alarm/alarm-count-trend-reportlet/index.ts`
- `work-order-action-rate` -> `./src/lib/components/shared-base/worker-order/worker-order-action-rate-reportlet/index.ts`
- `work-order-step-rate` -> `./src/lib/components/shared-base/worker-order/worker-order-step-rate-reportlet/index.ts`
- `work-order-type-rate` -> `./src/lib/components/shared-base/worker-order/worker-order-type-rate-reportlet/index.ts`

### `plugins/reportlet/seed-default-reportlets/module-federation.config.ts`
- `plugin-init` -> `./src/lib/plugin-init.ts`
- `plugin-info` -> `./src/package.js`
- `plugin-config-data-forms` -> `./src/lib/config-forms/config-data.index.ts`
- `plugin-config-style-forms` -> `./src/lib/config-forms/config-style.index.ts`
- `line-reportlet` -> `./src/lib/components/common/line/index.ts`
- `text-reportlet` -> `./src/lib/components/common/text/index.ts`
- `box-reportlet` -> `./src/lib/components/common/box/index.ts`
- `print-info-reportlet` -> `./src/lib/components/common/print-info/index.ts`
- `seed-asset-information` -> `./src/lib/components/asset-base/asset-info-reportlet/index.ts`
- `seed-health-cause-reportlet` -> `./src/lib/components/asset-base/health-cause-reportlet/index.ts`
- `seed-asset-health-list` -> `./src/lib/components/location-base/asset-health-list-reportlet/index.ts`


### 7-2. Master ID 목록
### reportlet master IDs
- `plugins/reportlet/apm-default-reportlets/src/lib/config/master-configs.ts`
  - `asset-information`
  - `asset-image`
  - `asset-models-information`
  - `asset-alarm-summary`
  - `health-index-trend`
  - `rul-trend`
  - `running-trend`
  - `alarm-total-by-asset`
  - `alarm-type-by-asset`
  - `alarm-acknowledgement-status-by-asset`
  - `alarm-quality-by-asset`
  - `parameter-trace-chart`
  - `parameter-charts`
  - `parameter-worst-perf-top5-trend`
  - `parameter-worst-perf-top5-rate`
  - `parameter-worst-perf-top5-chart`
  - `location-information`
  - `asset-status-summary`
  - `asset-status-count`
  - `asset-health-heatmap`
  - `asset-health-list`
  - `sensor-connection-status`
  - `sensor-summary-status`
  - `sensor-worst-perf-top5`
  - `location-alarm-heatmap-timeline`
  - `health-index-summary`
  - `health-index-wosrt-perf-top5`
  - `kpi-summary`
  - `kpi-map`
  - `kpi-list`
  - `work-order-step-rate`
  - `work-order-type-rate`
  - `work-order-action-rate`
  - `alarm-type`
  - `alarm-severity`
  - `alarm-count-trend`
  - `line-reportlet`
  - `text-reportlet`
  - `box-reportlet`
  - `print-info-reportlet`
  - `empty-reportlet`
- `plugins/reportlet/seed-default-reportlets/src/lib/config/master-configs.ts`
  - `seed-asset-information`
  - `seed-health-cause-reportlet`
  - `seed-asset-health-list`
  - `line-reportlet`
  - `text-reportlet`
  - `box-reportlet`
  - `print-info-reportlet`
  - `empty-reportlet`

### widget master IDs
- `plugins/widget/apm-default-widgets/src/lib/config/master-configs/asset.ts`
  - `apm-asset-list`
  - `apm-asset-status-heatmap`
  - `apm-asset-parameters`
  - `apm-asset-status-summary`
- `plugins/widget/apm-default-widgets/src/lib/config/master-configs/location.ts`
  - `apm-locations-status-summary`
  - `apm-geo-status-summary`
- `plugins/widget/apm-default-widgets/src/lib/config/master-configs/kpi.ts`
  - `apm-kpi-status-summary`
- `plugins/widget/apm-default-widgets/src/lib/config/master-configs/alarm.ts`
  - `apm-alarm-count-trend`
  - `apm-alarm-list`
  - `apm-alarm-severity`
  - `apm-alarm-timeline`
  - `apm-alarm-type`
- `plugins/widget/apm-default-widgets/src/lib/config/master-configs/sensor.ts`
  - `apm-sensor-worst-top5`
  - `apm-sensor-asset-collect-rate-list`
  - `apm-sensor-overall-status`
  - `apm-sensor-parameter-collect-rate-list`
  - `apm-sensor-summary-status`
- `plugins/widget/apm-default-widgets/src/lib/config/master-configs/layout.ts`
  - `default-layout`
- `plugins/widget/seed-default-widgets/src/lib/config/master-configs/asset.ts`
  - `seed-asset-list`

---

## 8. Theme / i18n / Environment

### 8-1. Theme 파일 로딩 규칙
- 공통(Portal 기준):
  - `theme-color-level.json`
  - `theme-podod.json`
  - `theme-antd.json`
- 앱별:
  - `theme-[app].json` (예: `theme-dashboard.json`)
- 적용 함수:
  - `applyRuntimeTheme(themeColorLevel, themeAntD, themeCustomComponent, appName?)`

### 8-2. Environment 주요 키
- 공통 핵심:
  - `GV_PODO_API_PREFIX`, `GV_APM_API_PREFIX`, `GV_ASSET_META_API_PREFIX`
  - `AUTH`, `API_REQUEST_OPTION`
  - `LIMIT_TAB_COUNT`, `USE_COOKIE_AUTH`, `INIT_PAGE_KEY`
- 시각/레이아웃:
  - `ECHART_FONT_STYLE`, `STYLE`, `MAP_CUSTOM_SVG_STYLE`, `WIDGETS_GRID_LAYOUT`
- 옵션:
  - `REALTIME_AGGREGATION`, `PRINT_APP_DEV_PORT`, `USE_AUTH_MICRO_APP`

### 8-3. i18n
- 앱 locale 파일: `apps/*/src/assets/i18n/*-locale-{en,ko,ja,cn}.json`
- 초기화: `initI18N(getI18nResources({ en, ko, ja, cn }))`
- 패키지/플러그인 스코프 i18n은 plugin init에서 반드시 초기화

---

## 9. 실행/빌드/배포 오퍼레이션

### 9-1. `package.json` 스크립트 요약
### 앱 실행
- `npm run ameta` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve ameta`
- `npm run apm` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve apm`
- `npm run dashboard` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve dashboard`
- `npm run dashboard-prod` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve dashboard --prod`
- `npm run pipeline` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve pipeline`
- `npm run pipeline-prod` : `npx nx serve pipeline --prod`
- `npm run portal` : `cross-env NODE_ENV=development:./proxy.conf-podo.json:podo-purple:all npx nx serve portal`
- `npm run portal-prod` : `cross-env NODE_ENV=development:./proxy.conf-podo.json:podo-purple:all npx nx serve portal --prod`
- `npm run print` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve print`
- `npm run print-prod` : `npx nx serve print --prod`
- `npm run report` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve report`
- `npm run report-prod` : `npx nx serve report --prod`
- `npm run studio` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve studio`
- `npm run studio-podo` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve studio`
- `npm run studio-prod` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve studio --prod`
- `npm run system` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve system`
- `npm run system-prod` : `npx nx serve system --prod`
- `npm run user` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve user`
- `npm run user-prod` : `npx nx serve user --prod`

### 앱 빌드
- `npm run build:all-apps` : `npm run build:portal && npm run build:dashboard && npm run build:report && npm run build:print && npm run build:pipeline && npm run build:studio && npm run build:system && npm run build:user`
- `npm run build:ameta` : `cross-env NODE_ENV=prod:: npx nx build ameta --prod`
- `npm run build:apm` : `cross-env NODE_ENV=prod:: npx nx build apm --prod`
- `npm run build:dashboard` : `cross-env NODE_ENV=prod:: npx nx build dashboard --prod`
- `npm run build:pipeline` : `cross-env NODE_ENV=prod:: npx nx build pipeline --prod`
- `npm run build:portal` : `cross-env NODE_ENV=prod::: npx nx build portal --prod`
- `npm run build:print` : `cross-env NODE_ENV=prod:: npx nx build print --prod`
- `npm run build:report` : `cross-env NODE_ENV=prod:: npx nx build report --prod`
- `npm run build:studio` : `cross-env NODE_ENV=prod:: npx nx build studio --prod`
- `npm run build:system` : `cross-env NODE_ENV=prod:: npx nx build system --prod`
- `npm run build:user` : `cross-env NODE_ENV=prod:: npx nx build user --prod`

### 플러그인 실행/테스트
- `npm run plugin:apm-default-reportlets` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve apm-default-reportlets`
- `npm run plugin:apm-default-widgets` : `cross-env NODE_ENV=development:./proxy.conf-localtest.json:remotetest npx nx serve apm-default-widgets`
- `npm run plugin:localtest:apm-default-widgets` : `cross-env NODE_ENV=development:./proxy.conf-localtest.json:localtest npx nx serve apm-default-widgets --open`
- `npm run plugin:localtest:seed-default-widgets` : `cross-env NODE_ENV=development:./proxy.conf-localtest.json:localtest npx nx serve seed-default-widgets --open`
- `npm run plugin:seed-default-reportlets` : `cross-env NODE_ENV=development:podo-purple:all npx nx serve seed-default-reportlets`
- `npm run plugin:seed-default-widgets` : `cross-env NODE_ENV=development:./proxy.conf-localtest.json:remotetest npx nx serve seed-default-widgets`
- `npm run plugin:testbed` : `cross-env NODE_ENV=development:./proxy.conf.json:podo-purple npx nx serve testbed --open`
- `npm run plugin:testbed-static` : `cross-env NODE_ENV=development:./proxy.conf.json:podo-purple npx nx serve-static testbed`

### 플러그인 빌드
- `npm run plugin:build:apm-default-reportlets` : `cross-env NODE_ENV=prod:: npx nx build apm-default-reportlets`
- `npm run plugin:build:apm-default-widgets` : `cross-env NODE_ENV=prod:: npx nx build apm-default-widgets`
- `npm run plugin:build:seed-default-reportlets` : `cross-env NODE_ENV=prod:: npx nx build seed-default-reportlets`
- `npm run plugin:build:seed-default-widgets` : `cross-env NODE_ENV=prod:: npx nx build seed-default-widgets`

### 플러그인 정보
- `npm run plugin:show:apm-default-reportlets` : `cross-env NODE_ENV=development:podo-purple:all npx nx show project apm-default-reportlets --web`
- `npm run plugin:show:apm-default-widgets` : `cross-env NODE_ENV=development:./proxy.conf-localtest.json:remotetest npx nx show project apm-default-widgets --web`
- `npm run plugin:show:seed-default-reportlets` : `cross-env NODE_ENV=development:podo-purple:all npx nx show project seed-default-reportlets --web`
- `npm run plugin:show:seed-default-widgets` : `cross-env NODE_ENV=development:./proxy.conf-localtest.json:remotetest npx nx show project seed-default-widgets --web`

### 문서/테스트
- `npm run docs` : `rm -rf ./node_modules/.cache && npm run docs:dev --open`
- `npm run docs:build` : `dumi build`
- `npm run docs:dev` : `dumi dev`
- `npm run docs:prepare` : `dumi setup`
- `npm run docs:preview` : `dumi preview --port 7777`
- `npm run test` : `npx vitest`
- `npm run test:coverage` : `npx vitest run --coverage`
- `npm run test:ui` : `npx vitest --ui`
- `npm run test:update` : `npx vitest -u`

### 패키지 빌드/배포
- `npm run build-pkg:all` : `npm run build-pkg:config-apm && npm run build-pkg:config-icon && npm run build-pkg:config-application && npm run build-pkg:config-dashboard && npm run build-pkg:config-layout && npm run build-pkg:config-report`
- `npm run build-pkg:config-apm` : `cd ./libs/config/apm && vite build`
- `npm run build-pkg:config-application` : `cd ./libs/config/application && vite build`
- `npm run build-pkg:config-dashboard` : `cd ./libs/config/dashboard && vite build`
- `npm run build-pkg:config-icon` : `cd ./libs/config/icon && vite build`
- `npm run build-pkg:config-layout` : `cd ./libs/config/layout && vite build`
- `npm run build-pkg:config-report` : `cd ./libs/config/report && vite build`
- `npm run bump-beta-pkg:all` : `node ./tools/bump-version.mjs --part=beta --strategy=beta ./package.json ./libs/config/icon/package.json ./libs/config/application/package.json ./libs/config/dashboard/package.json ./libs/config/layout/package.json  ./libs/config/report/package.json`
- `npm run bump-major-pkg:all` : `node ./tools/bump-version.mjs --part=major --strategy=highest ./package.json ./libs/config/icon/package.json ./libs/config/application/package.json ./libs/config/dashboard/package.json ./libs/config/layout/package.json  ./libs/config/report/package.json`
- `npm run bump-minor-pkg:all` : `node ./tools/bump-version.mjs --part=minor --strategy=highest ./package.json ./libs/config/icon/package.json ./libs/config/application/package.json ./libs/config/dashboard/package.json ./libs/config/layout/package.json  ./libs/config/report/package.json`
- `npm run bump-path-pkg:all` : `node ./tools/bump-version.mjs --part=patch --strategy=highest ./package.json ./libs/config/icon/package.json ./libs/config/application/package.json ./libs/config/dashboard/package.json ./libs/config/layout/package.json  ./libs/config/report/package.json`
- `npm run bump-reset-pkg-1.3.0:all` : `node ./tools/bump-version.mjs --part=reset --strategy=reset --reversion=1.3.0 ./package.json ./libs/config/icon/package.json ./libs/config/application/package.json ./libs/config/dashboard/package.json ./libs/config/layout/package.json  ./libs/config/report/package.json`
- `npm run cp-pkg:all` : `npm run cp-pkg:config-apm && npm run cp-pkg:config-icon && npm run cp-pkg:config-application && npm run cp-pkg:config-dashboard && npm run cp-pkg:config-layout && npm run cp-pkg:config-report`
- `npm run cp-pkg:config-apm` : `cp libs/config/apm/package.json dist/libs/config/apm`
- `npm run cp-pkg:config-application` : `cp libs/config/application/package.json dist/libs/config/application`
- `npm run cp-pkg:config-dashboard` : `cp libs/config/dashboard/package.json dist/libs/config/dashboard`
- `npm run cp-pkg:config-icon` : `cp libs/config/icon/package.json dist/libs/config/icon`
- `npm run cp-pkg:config-layout` : `cp libs/config/layout/package.json dist/libs/config/layout`
- `npm run cp-pkg:config-report` : `cp libs/config/report/package.json dist/libs/config/report`
- `npm run graph` : `npx nx dep-graph`
- `npm run nx:report` : `npx nx report`
- `npm run show:ameta` : `npx nx show project ameta --web`
- `npm run show:apm` : `npx nx show project apm --web`
- `npm run show:dashboard` : `npx nx show project dashboard --web`
- `npm run show:pipeline` : `npx nx show project pipeline --web`
- `npm run show:portal` : `npx nx show project portal --web`
- `npm run show:print` : `npx nx show project print --web`
- `npm run show:report` : `npx nx show project report --web`
- `npm run show:studio` : `npx nx show project studio --web`
- `npm run show:system` : `npx nx show project system --web`
- `npm run show:user` : `npx nx show project user --web`


### 9-2. 배포용 쉘 스크립트
- `gv-build-application.sh`
  - 전체 앱 build + plugin build + production env 교체
- `gv-build-plugins.sh`
  - 기본 widget/reportlet plugin build + 불필요 파일 제거
- `gv-build-plugins-tgz.sh`
  - plugin build 후 `.tgz` 패키징
- `gv-build-plugin-widget-tgz.sh`, `gv-build-plugin-reportlet-tgz.sh`
  - 타입별 패키징
- `gv-9-replace-production-env.sh`
  - dist의 `config.json`, `environment.json`를 `*.prod.json`으로 교체

---

## 10. 다른 저장소에서 재사용할 때 최소 구현 순서

### 10-1. 앱(Host) 먼저
1. `apps/micro-apps/[app]` 생성
2. `main.ts` 초기화 패턴 적용
3. `sidebar-menu.ts` + page wrapper 구성
4. portal proxy 등록 (`apps/portal/web/proxy.conf-*.json`)
5. backend menu/license 연결

### 10-2. 위젯/리포틀릿 플러그인
1. plugin 프로젝트 생성 (`plugins/widget/*`, `plugins/reportlet/*`)
2. `module-federation.config.ts` expose 구성
3. `plugin-init`, `plugin-config`, `master-configs` 작성
4. (reportlet) data/style form expose 구성
5. local test (`plugin:testbed` 또는 `plugin:localtest:*`)로 검증
6. build + 패키징 + 배포

### 10-3. 핵심 체크리스트
- plugin 이름과 `pluginPath`가 서버 경로와 일치하는가
- expose key와 master `path`가 일치하는가
- i18n namespace 충돌이 없는가
- backend `/plugins` 응답에서 `activating=true`로 노출되는가
- host `plugin.dev.json` 또는 backend plugin URL이 올바른가

---

## 11. `@grandview` 실사용 가이드 (핵심 패키지)

### 11-1. `@grandview/shared`
- 역할: 런타임 config/theme/i18n/http/state/module federation/event bus 핵심
- 자주 쓰는 API
  - config/theme: `getMicroAppConfig`, `applyRuntimeTheme`, `gvPODOApiPrefix`
  - i18n/time: `initI18N`, `initTimeLocale`, `initDayjsPlugins`
  - plugin: `setPluginCacheKey`, `setPluginDev`, `initRemoteDefinitionsAndPlugins`
  - widget/reportlet event: `sendDataToSubscriber`, `useSubscribeData`
  - realtime filter: `useRealtimeFilter`, `useChangedRealtimeFilterCondition`

### 11-2. `@grandview/component`
- 역할: 공통 UI, Builder 컨테이너, Widget/Reportlet 컨테이너, 필터/폼
- 자주 쓰는 컴포넌트
  - Widget: `GVWidget`, `GVWidgetTitle`, `GVTable`, `GVTableCount`
  - Reportlet: `GVReportlet`, `GVReportletHeader`, `GVReportletBody`
  - 상태 표시: `GVCustomColorStatus`, `GVNumberFormat`, `GVLocationAssetName`
  - Builder/Plugin: `GVPluginWidgetList`, `GVPluginReportletList`, `GVPluginReportletDataForms`, `GVPluginReportletStyleForms`

### 11-3. 도메인 패키지
- `@grandview/web-domain-apm`: APM 페이지/포틀릿/상세 위젯
- `@grandview/web-domain-report`: 리포트 히스토리/온디맨드/프린트 페이지
- `@grandview/web-domain-studio`: Dashboard/Layout/Report Builder 페이지
- `@grandview/web-domain-system`, `@grandview/web-domain-user`, `@grandview/web-domain-ameta`, `@grandview/web-domain-pipeline`

### 11-4. Builder 템플릿 패키지
- `@grandview/web-template-dashboard`: `GVDashboardTemplate`
- `@grandview/web-template-report`: `GVReportPageTemplate`, `GVPrintReportPageTemplate`
- `@grandview/web-layout-micro-app`: 탭/레이아웃/사이드바 host 컨테이너

### 11-5. APM 플러그인 컴포넌트 패키지
- `@grandview/web-widget-apm`: APM widget 컴포넌트 세트
- `@grandview/web-reportlet-apm`: APM reportlet 컴포넌트 + config form 세트

### 11-6. 스타일/그래프/기타
- `@grandview/chart-echarts`: 차트 wrapper/options generator
- `@grandview/icons`: 제품 전용 아이콘
- `@grandview/stylet`: 디자인 토큰 함수
- `@grandview/style`: SCSS/CSS 스타일 자산 패키지
- `@grandview/podo-grid-layout`, `@grandview/podo-edge-flow`, `@grandview/node-red`: 레이아웃/플로우 빌더 영역

---

## 12. 코드 템플릿 모음

### 12-1. Host 앱 Plugin 초기화 템플릿

```ts
Promise.all([
  getMicroAppConfig(GV_MICRO_APP.DASHBOARD),
  getMicroAppConfig(GV_MICRO_APP.PORTAL, 'theme-color-level.json'),
  getMicroAppConfig(GV_MICRO_APP.PORTAL, 'theme-podod.json'),
  getMicroAppConfig(GV_MICRO_APP.PORTAL, 'theme-antd.json'),
  getMicroAppConfig(GV_MICRO_APP.DASHBOARD, 'theme-dashboard.json'),
  getMicroAppConfig(GV_MICRO_APP.DASHBOARD, 'environment.json'),
  getMicroAppConfig(GV_MICRO_APP.DASHBOARD, 'plugin.dev.json'),
  httpService.get<any>(`${gvPODOApiPrefix()}/plugins?activating=true&typeCodes=WIDGET&size=100`),
]).then((result) => {
  (window as any).GV_CONFIG = result[0]?.data;
  applyRuntimeTheme(result[1]?.data || {}, result[3]?.data || {}, result[2]?.data || {}, GV_MICRO_APP.DASHBOARD);
  initI18N(getI18nResources({ en, ko, ja, cn }));
  initTimeLocale();
  initDayjsPlugins();

  setPluginCacheKey(GV_MICRO_APP.DASHBOARD);
  setPluginDev(result[6]?.data || {});
  initRemoteDefinitionsAndPlugins(result[7]?.content)?.then(() => import('./bootstrap'));
});
```

### 12-2. Widget Plugin `module-federation.config.ts` 템플릿

```ts
const config: ModuleFederationConfig = {
  name: 'my-default-widgets',
  exposes: {
    'plugin-init': './src/lib/plugin-init.ts',
    'plugin-info': './src/package.js',
    'my-widget-a': './src/lib/components/my-widget-a/index.ts',
    'my-widget-b': './src/lib/components/my-widget-b/index.ts',
  },
};
```

### 12-3. Reportlet Plugin Data/Style Form expose 템플릿

```ts
exposes: {
  'plugin-init': './src/lib/plugin-init.ts',
  'plugin-info': './src/package.js',
  'plugin-config-data-forms': './src/lib/config-forms/config-data.index.ts',
  'plugin-config-style-forms': './src/lib/config-forms/config-style.index.ts',
  'my-reportlet': './src/lib/components/my-reportlet/index.ts',
}
```

---

## 13. 주의사항 / 리스크

- `plugin-info`와 실제 배포 path 불일치 시 동적 로딩 실패
- `master.path`와 MF expose key 불일치 시 Builder에서 생성 불가
- backend 메뉴/라이선스가 누락되면 앱/페이지가 보여도 접근 불가
- plugin dev proxy 경로가 잘못되면 로컬 연동 실패
- theme 파일 key 누락 시 일부 컴포넌트 색상 깨짐
- `NODE_ENV` 커스텀 포맷(`development:proxy:theme:server`) 의존 로직 주의

---

## 14. 부록 A: `@grandview` 패키지 전체 API 카탈로그

아래 목록은 `node_modules/@grandview/*`의 `package.json` + `index.d.ts` export를 기준으로 추출한 전체 참조입니다.

### @grandview/chart-echarts
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 36
- exports:
  - `./lib/echarts/zoom-echart/zoom-echart`
  - `./lib/utils/utils`
  - `./lib/echarts/echarts.index`
  - `./types`
  - `./lib/echarts/extension/dataTool`
  - `./lib/echarts/echarts-api.service`
  - `./lib/echarts/echarts-api.hook`
  - `./lib/echarts/y-axis-range/y-axis-range`
  - `./lib/echarts/legend-popover/legend-popover`
  - `./lib/echarts/legend-popover/legend-popover-content/legend-popover-content`
  - `./lib/echarts/normalizer/normalizer`
  - `./lib/echarts/chart-header/chart-header`
  - `./lib/echarts/threshold/threshold`
  - `./lib/echarts/vibration-fft-threshold/vibration-fft-threshold`
  - `./lib/options/define/common`
  - `./lib/options/define/image`
  - `./lib/options/option/axis-option`
  - `./lib/options/option/color-option`
  - `./lib/options/option/grid-option`
  - `./lib/options/option/legend-option`
  - `./lib/options/option/mark-area-option`
  - `./lib/options/option/mark-line-option`
  - `./lib/options/option/mark-point-option`
  - `./lib/options/option/series-option`
  - `./lib/options/option/toolbox-option`
  - `./lib/options/option/tooltip-option`
  - `./lib/options/option/brush-option`
  - `./lib/options/option/zoom-option`
  - `./lib/options/util/chart-util`
  - `./lib/options/util/threshold-util`
  - `./lib/options/chart-option-wrapper`
  - `./lib/options/options-generator`
  - `./lib/echarts-map/echarts-map.index`
  - `./lib/options/map/effects-generator`
  - `./lib/options/map/animation-option`
  - `./lib/options/map/element-option`

### @grandview/component
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 150
- exports:
  - `./lib/index`
  - `lib/index/common/alert-message/alert-message`
  - `lib/index/common/backdrop/backdrop`
  - `lib/index/common/backdrop/header-backdrop`
  - `lib/index/common/custom-color-status/custom-color-status`
  - `lib/index/common/empty-chart/empty-chart`
  - `lib/index/common/number-format/number-format`
  - `lib/index/common/percent-progress/percent-progress`
  - `lib/index/common/round-text/round-text`
  - `lib/index/common/round-value/round-value`
  - `lib/index/common/text-progress/text-progress`
  - `lib/index/common/bookmark/bookmark`
  - `lib/index/common/collection-interval/collection-interval`
  - `lib/index/common/contrast-before/contrast-before`
  - `lib/index/common/create-update-info/create-update-info`
  - `lib/index/common/data-loading/data-loading`
  - `lib/index/common/delete-circle-button/delete-circle-button`
  - `lib/index/common/file-import/file-import`
  - `lib/index/common/more-popover/more-popover`
  - `lib/index/common/required/required`
  - `lib/index/common/start-end-date-info/start-end-date-info`
  - `lib/index/common/status-info/status-info`
  - `lib/index/common/table-count/table-count`
  - `lib/index/common/content/content`
  - `lib/index/common/ellipsis-tooltip-paragraph/ellipsis-tooltip-paragraph`
  - `lib/index/common/info-message/info-message`
  - `lib/index/common/input-mention/input-mention`
  - `lib/index/common/panel/panel`
  - `lib/index/common/text-highlight/high-light.pipe`
  - `lib/index/common/text-highlight/hight-light/hight-light`
  - `lib/index/common/text-information/text-information`
  - `lib/index/common/tier-two-content/tier-two-content`
  - `lib/index/common/value-unit/value-unit`
  - `lib/index/common/work-order-status-badge-button/work-order-status-badge-button`
  - `lib/index/image/asset/add-image-button/add-image-button`
  - `lib/index/image/asset/image-box/image-box`
  - `lib/index/image/asset/image-box/image-object.service`
  - `lib/index/image/asset/image-carousel/image-carousel`
  - `lib/index/image/user/add-user-image/add-user-image`
  - `lib/index/image/user/user-avatar/more-avatar/more-avatar`
  - `lib/index/image/user/user-avatar/user-avatar`
  - `lib/index/image/user/user-image-editor/user-image-editor`
  - `lib/index/image/user/user-image-url/user-image-url`
  - `lib/index/image/user/user-image/user-image`
  - `lib/index/antd/button/button`
  - `lib/index/antd/carousel/carousel`
  - `lib/index/antd/custom-select/custom-select`
  - `lib/index/antd/data-spinner/data-spinner`
  - `lib/index/antd/empty/empty`
  - `lib/index/antd/empty/no-menu-permission`
  - `lib/index/antd/form/form`
  - `lib/index/antd/form/form.service`
  - `lib/index/antd/form/form.type`
  - `lib/index/antd/front-pagination/front-pagination`
  - `lib/index/antd/icon-button/icon-button`
  - `lib/index/antd/input/input`
  - `lib/index/antd/link-button/link-button`
  - `lib/index/antd/list-box/list-box`
  - `lib/index/antd/pagination/pagination`
  - `lib/index/antd/popup-confirm/popup-confirm`
  - `lib/index/antd/search-enter-input/search-enter-input`
  - `lib/index/antd/search-input/search-input`
  - `lib/index/antd/search-selector/search-selector`
  - `lib/index/antd/spinner/spinner`
  - `lib/index/antd/table`
  - `lib/index/antd/text-tooltip/text-tooltip`
  - `lib/index/antd/tooltip/alarm-tooltip-content/alarm-tooltip-content`
  - `lib/index/antd/tooltip/common-tooltip-content/common-tooltip-content`
  - `lib/index/antd/tooltip/health-tooltip-content/health-tooltip-content`
  - `lib/index/antd/tooltip/tooltip`
  - `lib/index/antd/transfer-table/transfer-table`
  - `lib/index/antd/tree/tree`
  - `lib/index/dynamic-field/dynamic-field`
  - `lib/index/editable-fields/change-title/change-title`
  - `lib/index/editable-fields/editable-basic-field/editable-basic-field`
  - `lib/index/editable-fields/editable-collection-interval-field/editable-collection-interval-field`
  - `lib/index/editable-fields/editable-dynamic-field/editable-dynamic-field`
  - `lib/index/editable-fields/editable-hierarchy-field/editable-hierarchy-field-form-item`
  - `lib/index/editable-fields/editable-image-list-field/editable-image-list-field`
  - `lib/index/editable-fields/editable-input-mention-field/editable-input-mention-field`
  - `lib/index/editable-fields/editable-progress-bar-field/editable-progress-bar-field`
  - `lib/index/editable-fields/editable-table-field/editable-table-field`
  - `lib/index/editable-fields/editable-title-field/editable-title-field`
  - `lib/index/editable-fields/editable-upload-field/editable-upload-field`
  - `lib/index/editable-fields/header-actions-field/header-actions-field`
  - `lib/index/editable-fields/selector-field/selector-field`
  - `lib/index/editable-fields/single-editable-container/single-editable-container`
  - `lib/index/editable-fields/widget-title/widget-title`
  - `lib/index/hierarchy-maker/hierarchy-maker`
  - `lib/index/validators`
  - `lib/index/containers/asset-header/asset-header`
  - `lib/index/containers/management-header/management-header`
  - `lib/index/containers/page/page`
  - `lib/index/domain/alarm-rule-condition/alarm-rule-condition`
  - `lib/index/domain/alarm-rule-domain/alarm-rule-domain`
  - `lib/index/domain/alarm-rule-threshold/alarm-rule-threshold`
  - `lib/index/domain/change-alarm-group-status/change-alarm-group-status`
  - `lib/index/domain/global-state-alert/global-state-refresher/global-state-refresher`
  - `lib/index/domain/license/license.service`
  - `lib/index/domain/location-asset/location-asset`
  - `lib/index/domain/location-path/location-path`
  - `lib/index/domain/priority-type/priority-type`
  - `lib/index/domain/progress-type/progress-type`
  - `lib/index/domain/rul-day/rul-day`
  - `lib/index/domain/status-batch-index/status-batch-index`
  - `lib/index/domain/status-types-ui/status-types-ui`
  - `lib/index/domain/text-tooltip-select/text-tooltip-select`
  - `lib/index/domain/types-ui/types-ui`
  - `lib/index/domain/work-order-custom-type/work-order-custom-type`
  - `lib/index/filters/aggregation-filter/aggregation-filter`
  - `lib/index/filters/asset-hierarchy-filter/asset-hierarchy-filter`
  - `lib/index/filters/dashboard-filter/dashboard-filter`
  - `lib/index/filters/health-status-filter/health-status-filter`
  - `lib/index/filters/location-filter/location-filter`
  - `lib/index/filters/location-filter/widget-location-filter/widget-location-filter`
  - `lib/index/filters/realtime-filter/realtime-filter`
  - `lib/index/filters/realtime-filter/realtime-filter-options/realtime-option-selector/realtime-option-selector`
  - `lib/index/filters/realtime-filter/realtime-filter-options/realtime-option-selector/realtime-option-selector.service`
  - `lib/index/filters/realtime-filter/realtime-filter.service`
  - `lib/index/filters/refresh-filter/refresh-filter`
  - `lib/index/date-pickers/range-picker`
  - `lib/index/boxes/alarm-status-box/alarm-status-box`
  - `lib/index/date-pickers/date-picker`
  - `lib/index/date-pickers/time-picker`
  - `lib/index/resizable-panel/index`
  - `lib/index/color-picker/color-picker`
  - `lib/index/error-boundary`
  - `lib/index/plugins/reportlet`
  - `lib/index/plugins/widget`
  - `./lib2/index`
  - `lib2/index/ai-assistant-helper/ai-assistant-helper`
  - `lib2/index/common/help-info/help-info`
  - `lib2/index/common/modal/index`
  - `lib2/index/common/no-data/no-data`
  - `lib2/index/panel`
  - `lib2/index/plugins/reportlet/containers`
  - `lib2/index/plugins/widget/containers`
  - `lib2/index/plugins/genlet/containers`
  - `lib2/index/plugins/genlet/expand-genlet/expand-genlet`
  - `lib2/index/plugins/genlet/genlet.state`
  - `lib2/index/plugins/genlet/page-link/page-link`
  - `lib2/index/editable-fields/custom-select-field/custom-select-field`
  - `lib2/index/editable-fields/custom-tree-field/custom-tree-field`
  - `lib2/index/editable-fields/editable-slider-field/editable-slider-field`
  - `lib2/index/common/app-shower/app-shower`
  - `lib2/index/file/file-list/file-list`
  - `lib2/index/file/file-list/data-api/file.hook`
  - `lib2/index/file/add-file-button/add-file-button`
  - `lib2/index/editable-fields/editable-file-list-field/editable-file-list-field`
  - `lib2/index/containers/management-header/management-header`

### @grandview/element
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 9
- exports:
  - `./lib/konva-basic-element/index`
  - `lib/konva-basic-element/index/elements/text-element`
  - `lib/konva-basic-element/index/elements/box-element`
  - `lib/konva-basic-element/index/elements/circle-element`
  - `lib/konva-basic-element/index/elements/ring-element`
  - `lib/konva-basic-element/index/elements/line-element`
  - `lib/konva-basic-element/index/elements/arrow-element`
  - `lib/konva-basic-element/index/elements/image-element`
  - `lib/konva-basic-element/index/services/common.service`

### @grandview/icons
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 74
- exports:
  - `./lib/antd-icon-generator/index`
  - `lib/antd-icon-generator/index/Icon`
  - `./lib/charts/chart-toggle-icon`
  - `./lib/charts/chart-view-icon`
  - `./lib/alarm/alarm-caused-parameter-toggle-icon`
  - `./lib/alarm/alarm-status-icon`
  - `./lib/kpi-status/apm/smiley-icon`
  - `./lib/kpi-status/kpi-status-icon`
  - `./lib/apm/work-progress-icon`
  - `./lib/etc/group-title-icon`
  - `./lib/etc/priority-icon`
  - `./lib/etc/unload-icon`
  - `./lib/etc/ups-downs-icon`
  - `./lib/metadata/asset-group-icon`
  - `./lib/metadata/context-icon`
  - `./lib/metadata/model/model-source-list-toggle-icon`
  - `./lib/metadata/model/training-icon`
  - `./lib/new-icons/alarm-detail-icon`
  - `./lib/new-icons/alarm-icon`
  - `./lib/new-icons/alarm-rule-icon`
  - `./lib/new-icons/alarm-summary-icon`
  - `./lib/new-icons/asset-filled-icon`
  - `./lib/new-icons/asset-icon`
  - `./lib/new-icons/asset-template-icon`
  - `./lib/new-icons/assets-detail-icon`
  - `./lib/new-icons/assets-icon`
  - `./lib/new-icons/chart-multi-series-icon`
  - `./lib/new-icons/chart-normalize-icon`
  - `./lib/new-icons/chart-y-range-icon`
  - `./lib/new-icons/compare-before-down`
  - `./lib/new-icons/compare-before-none`
  - `./lib/new-icons/compare-before-up`
  - `./lib/new-icons/context-chart-icon`
  - `./lib/new-icons/defect-info-icon`
  - `./lib/new-icons/goto-page-icon`
  - `./lib/new-icons/group-title-icon`
  - `./lib/new-icons/hierarchy-icon`
  - `./lib/new-icons/layer-merge-icon`
  - `./lib/new-icons/layer-single-icon`
  - `./lib/new-icons/model-icon`
  - `./lib/new-icons/model-training-icon`
  - `./lib/new-icons/parameter-group-icon`
  - `./lib/new-icons/parameter-icon`
  - `./lib/new-icons/step-line-icon`
  - `./lib/new-icons/step-spec-icon`
  - `./lib/new-icons/tree-collapse-icon`
  - `./lib/new-icons/tree-expand-icon`
  - `./lib/new-icons/work-order-detail-icon`
  - `./lib/new-icons/work-order-icon`
  - `./lib/optima/alarm-management-icon`
  - `./lib/optima/asset-configuration-icon`
  - `./lib/optima/data-viewer-icon`
  - `./lib/optima/no-period-icon`
  - `./lib/optima/parameter-selection-icon`
  - `./lib/optima/stack-chart-icon`
  - `./lib/platform/elements-icon`
  - `./lib/platform/flow-builder-icon`
  - `./lib/platform/page-config-icon`
  - `./lib/platform/report-builder-icon`
  - `./lib/platform/report-history-icon`
  - `./lib/platform/report-on-demand-icon`
  - `./lib/platform/report-preview-icon`
  - `./lib/platform/reportlet-config-icon`
  - `./lib/platform/reportlets-icon`
  - `./lib/platform/size-fit-icon`
  - `./lib/platform/style-icon`
  - `./lib/ai/ai-assistant-filled`
  - `./lib/ai/ai-assistant-outlined`
  - `./lib/ai/ai-chat-config`
  - `./lib/ai/ai-feature`
  - `./lib/ai/ai-model`
  - `./lib/apb/asset-compare`
  - `./lib/apb/boxplot-chart`
  - `./lib/apb/kpi-overview`

### @grandview/markdown
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 8
- exports:
  - `./lib/markdown/markdown`
  - `./lib/markdown-plugins/rehype-podo-genlet.plugin`
  - `./lib/markdown-plugins/test-response-genai`
  - `./lib/genlet/index`
  - `lib/genlet/index/i18n-loader`
  - `lib/genlet/index/plugin-config`
  - `lib/genlet/index/components/podo-genlet`
  - `lib/genlet/index/common.style`

### @grandview/node-red
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 2
- exports:
  - `./lib/node-red`
  - `lib/node-red/node-red.service`

### @grandview/plugin-testbed
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 4
- exports:
  - `./lib/login/testbed-login`
  - `./lib/builder/dashboard-builder/dashboard-builder`
  - `./lib/builder/report-builder/report-builder`
  - `./lib/template/dashboard/lib/local-test-dashboard-template/local-test-dashboard-template`

### @grandview/podo-edge-flow
- version: `3.1.1`
- main: `./index.js`
- export count: 0

### @grandview/podo-grid-layout
- version: `1.4.4`
- main: `index.js`
- export count: 0

### @grandview/shared
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 172
- exports:
  - `./lib/config/config`
  - `./lib/config/config.constant`
  - `./lib/config/personalization-config.service`
  - `./lib/config/session-storage.service`
  - `./lib/i18n/browser-lang`
  - `./lib/i18n/i18n`
  - `./lib/i18n/i18n.hook`
  - `./lib/i18n/tab-i18n`
  - `./lib/ajax/api-response.service`
  - `./lib/ajax/http.service`
  - `./lib/cookie/cookie.service`
  - `./lib/timezone`
  - `lib/timezone/date-format`
  - `lib/timezone/time-format-locale`
  - `lib/timezone/time-util`
  - `lib/timezone/timezone.service`
  - `lib/timezone/timezone-list`
  - `./lib/iframe/active-page-history-manager`
  - `./lib/iframe/commands/app-shower.hook`
  - `./lib/iframe/commands/micro-app/change-global-state.command`
  - `./lib/iframe/commands/micro-app/check-form-dirty.command`
  - `./lib/iframe/commands/micro-app/fire-action-from-portal.command`
  - `./lib/iframe/commands/micro-app/init-iframe.command`
  - `./lib/iframe/commands/micro-app/init-micro-app.command`
  - `./lib/iframe/commands/micro-app/load-page.command`
  - `./lib/iframe/commands/micro-app/set-active-micro-app.command`
  - `./lib/iframe/commands/modal-mask.hook`
  - `./lib/iframe/commands/switch-page.hook`
  - `./lib/iframe/commands/unauthorized.hook`
  - `./lib/iframe/communication/iframe-host-to-child.sender`
  - `./lib/iframe/communication/iframe-host.listener`
  - `./lib/iframe/communication/micro-app/iframe-child-to-host.sender`
  - `./lib/iframe/communication/micro-app/iframe-child.listener`
  - `./lib/iframe/communication/portal/iframe-resizer`
  - `./lib/iframe/iframe-micro-app-manager`
  - `./lib/hooks/creation-i18n.hook`
  - `./lib/hooks/shared/changed-global-state.hook`
  - `./lib/hooks/shared/load-script.hook`
  - `./lib/hooks/shared/query-paging.hook`
  - `./lib/hooks/shared/scroll-into-view.hook`
  - `./lib/hooks/shared/shared-one-way-subject`
  - `./lib/hooks/shared/window-resize.hook`
  - `./lib/hooks/tabkey/clear-state-init-params.hook`
  - `./lib/hooks/tabkey/common-one-way-data.hook`
  - `./lib/hooks/tabkey/common-one-way-subject`
  - `./lib/hooks/tabkey/realtime-filter-one-way-subject`
  - `./lib/hooks/tabkey/realtime-filter.hook`
  - `./lib/hooks/unmount-hooks`
  - `./lib/pipes/asset-status.pipe`
  - `./lib/pipes/empty-value.pipe`
  - `./lib/filters/realtime-filter-options.service`
  - `./lib/styles/common.style`
  - `./lib/utilities/characters/title-case`
  - `./lib/utilities/csv-util`
  - `./lib/utilities/dom-util`
  - `./lib/utilities/helper`
  - `lib/utilities/helper/is-function`
  - `lib/utilities/helper/is-string`
  - `lib/utilities/helper/pick`
  - `./lib/utilities/model-util`
  - `./lib/utilities/number-parser`
  - `./lib/utilities/number-util`
  - `./lib/utilities/query-util`
  - `./lib/utilities/random-generator`
  - `./lib/utilities/rul-util`
  - `./lib/utilities/table-util`
  - `./lib/utilities/tree-util`
  - `./lib/utilities/user-agent-util`
  - `./lib/utilities/utils`
  - `./lib/browser/history`
  - `./lib/browser/location`
  - `./lib/domain/alarm-rule.service`
  - `./lib/domain/common-types.service`
  - `./lib/domain/hooks/index`
  - `lib/domain/hooks/index/app-menus.hook`
  - `lib/domain/hooks/index/dashboard-sidebar-menu.hook`
  - `lib/domain/hooks/index/common-status.hook`
  - `lib/domain/hooks/index/common-types.hook`
  - `./lib/domain/image/image-url.hook`
  - `./lib/domain/image/image.hook`
  - `./lib/domain/image/user-image.hook`
  - `./lib/domain/image/user-image.service`
  - `./lib/domain/kpi-status/common-status.service`
  - `./lib/domain/kpi-status/kpi-status.service`
  - `./lib/domain/kpi-status/layout-status.service`
  - `./lib/domain/data-api/index`
  - `lib/domain/data-api/index/hooks/menu-permission.hook`
  - `lib/domain/data-api/index/models/menu-permission.model`
  - `./lib/auth/auth-user.service`
  - `./lib/auth/license`
  - `./lib/message/status-message.service`
  - `./lib/services/custom-property.service`
  - `./lib/services/drag-and-drop.service`
  - `./lib/services/file-to-json-upload.service`
  - `./lib/services/file-upload.service`
  - `./lib/services/form.service`
  - `./lib/services/image.service`
  - `./lib/services/menu.service`
  - `./lib/services/table.service`
  - `./lib/services/text.service`
  - `./lib/services/transaction.service`
  - `./lib/services/translate.service`
  - `./lib/services/tree.service`
  - `./lib/routing/page-link.service`
  - `./lib/routing/page-url-params.service`
  - `./lib/i18n/date-picker/ko`
  - `./lib/filters/location-filter.service`
  - `./lib/filters/table-filter.service`
  - `./lib/filters/type-filter.service`
  - `./lib/animation/animation`
  - `./lib/animation/animation.model`
  - `./lib/model/index`
  - `lib/model/index/model`
  - `lib/model/index/interface`
  - `lib/model/index/interface/table.interface`
  - `lib/model/index/types`
  - `lib/model/index/types/custom-field-properties-type`
  - `./lib/state/index`
  - `lib/state/index/shared/micro-app/web-management-app.state`
  - `lib/state/index/shared/web-ai-assistant.state`
  - `lib/state/index/shared/web-ai-genlet.state`
  - `lib/state/index/shared/web-auth.state`
  - `lib/state/index/shared/web-common.state`
  - `lib/state/index/shared/web-micro-app-layout.state`
  - `lib/state/index/shared/web-portal-layout.state`
  - `lib/state/index/tabkey/web-asset-alarm.state`
  - `lib/state/index/tabkey/web-asset-work-order.state`
  - `lib/state/index/tabkey/web-dirty-form.state`
  - `lib/state/index/tabkey/web-edit-mode.state`
  - `lib/state/index/tabkey/web-page-data-ux.state`
  - `lib/state/index/tabkey/web-page-transaction.state`
  - `lib/state/index/tabkey/web-page.state`
  - `lib/state/index/tabkey/web-realtime-filter.state`
  - `lib/state/index/tabkey/web-selected-asset.state`
  - `lib/state/index/tabkey/web-url-params.state`
  - `lib/state/index/tabkey/web-widget.state`
  - `lib/state/index/tabkey/actions/alarm/alarm-page-action.state`
  - `lib/state/index/tabkey/actions/batch-action.state`
  - `lib/state/index/tabkey/actions/chart-action.state`
  - `lib/state/index/tabkey/actions/component-props-action.state`
  - `lib/state/index/tabkey/actions/kpi-action.state`
  - `lib/state/index/tabkey/actions/location-action.state`
  - `lib/state/index/tabkey/actions/management/management-header-action.state`
  - `lib/state/index/tabkey/actions/management/management-page-action.state`
  - `lib/state/index/tabkey/actions/pagelet/pagelet-action.state`
  - `lib/state/index/tabkey/actions/share-state/share-state-page-action.state`
  - `lib/state/index/tabkey/actions/table-action.state`
  - `lib/state/index/tabkey/actions/web-favorite-action.state`
  - `lib/state/index/shared/web-shared.context`
  - `lib/state/index/tabkey/asset-trend-detail/asset-trend-detail.state`
  - `lib/state/index/tabkey/context/web-page.context`
  - `lib/state/index/tabkey/builder/dashboard-builder.state`
  - `lib/state/index/tabkey/builder/layout-builder.state`
  - `lib/state/index/tabkey/builder/report-builder.state`
  - `lib/state/index/tabkey/unmount-states`
  - `./lib/module-federation/dynamic-remote-module-nx.service`
  - `./lib/module-federation/plugin/plugin-config.service`
  - `./lib/module-federation/plugin/plugin.service`
  - `./lib/theme/antd-theme-config.service`
  - `./lib/theme/antd-theme-provider`
  - `./lib/theme/runtime-theme.service`
  - `./lib/theme/types/index`
  - `lib/theme/types/index/theme-antd.type`
  - `lib/theme/types/index/theme-podod.type`
  - `lib/theme/types/index/theme-color-level.type`
  - `./lib/broadcast-channel/index`
  - `lib/broadcast-channel/index/broadcast-channel.model`
  - `lib/broadcast-channel/index/broadcast-channel.service`
  - `lib/broadcast-channel/index/services/notification.service`
  - `./lib/license/app-license.hook`
  - `./lib/styles/create-podo-styles`
  - `./lib/styles/podo-token.type`

### @grandview/style
- version: `2.1.0-beta.1`
- exports: 타입 선언 파일(`index.d.ts`)이 없어 정적 리소스/스크립트 패키지로 사용
- 주요 파일:
  - `animate-example.css`
  - `animate.css`
  - `antd/button.scss`
  - `antd/check-box.scss`
  - `antd/cursor.scss`
  - `antd/date-picker.scss`
  - `antd/dropdown.scss`
  - `antd/form.scss`
  - `antd/layout.scss`
  - `antd/message.scss`
  - `antd/modal_alert.scss`
  - `antd/modal_confirm.scss`
  - `antd/popover.scss`
  - `antd/radio.scss`
  - `antd/select.scss`
  - `antd/sub-tab.scss`
  - `antd/switch.scss`
  - `antd/table.scss`
  - `antd/transfer.scss`
  - `antd/typography.scss`
  - `antd-components-theme.scss`
  - `gv-grid-layout.scss`
  - `gv-react-resizable.scss`
  - `hide-floating-window.scss`
  - `podo-grid-layout.scss`
  - `podo-react-resizable.scss`
  - `reset-font-family.scss`
  - `reset-single-application.scss`
  - `reset.scss`
  - `rt-vars-color.scss`
  - `select-blink-effect.scss`
  - `tooltip.scss`
  - `vars-color.scss`
  - `vars.scss`

### @grandview/stylet
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 4
- exports:
  - `./lib/vars`
  - `lib/vars/vars-color`
  - `./lib/vars-chart`
  - `./lib/root-style.util`

### @grandview/tween
- version: `2.1.0-beta.1`
- exports: 타입 선언 파일(`index.d.ts`)이 없어 정적 리소스/스크립트 패키지로 사용
- 주요 파일:
  - `konva-plugin.js`
  - `timeline-lite.min.js`
  - `tween-lite.min.js`

### @grandview/web-ai-assistant
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 1
- exports:
  - `./lib/ai-assistant`

### @grandview/web-domain-ameta
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 78
- exports:
  - `./lib/pagelet/index`
  - `lib/pagelet/index/alarm-rule-pagelet/alarm-rule-pagelet`
  - `lib/pagelet/index/asset-hierarchy-pagelet/asset-hierarchy-pagelet`
  - `lib/pagelet/index/asset-pagelet/asset-pagelet`
  - `lib/pagelet/index/asset-template-pagelet/asset-template-pagelet`
  - `lib/pagelet/index/model-pagelet/model-pagelet`
  - `lib/pagelet/index/parameter-group-pagelet/parameter-group-pagelet`
  - `lib/pagelet/index/parameter-pagelet/parameter-pagelet`
  - `lib/pagelet/index/asset-wizard-pagelet/asset-wizard-pagelet`
  - `lib/pagelet/index/context-pagelet/context-pagelet`
  - `lib/pagelet/index/asset-group-pagelet/asset-group-pagelet`
  - `./lib/form/index`
  - `lib/form/index/parameter-bulk-form/parameter-bulk-form`
  - `lib/form/index/parameter-form/parameter-form`
  - `lib/form/index/parameter-group-form/parameter-group-form`
  - `lib/form/index/parameter-select-form/parameter-select-form`
  - `lib/form/index/asset-info-basic-form/asset-info-basic-form`
  - `lib/form/index/asset-info-detail-form/asset-info-detail-form`
  - `lib/form/index/asset-template-form/asset-template-form`
  - `lib/form/index/asset-template-parameter-group-form/asset-template-parameter-group-form`
  - `lib/form/index/asset-bulk-form/asset-bulk-form`
  - `lib/form/index/asset-custom-properties-form/asset-custom-properties-form`
  - `lib/form/index/model/model-base-info-form/model-base-info-form`
  - `lib/form/index/model/model-info-form/model-info-form`
  - `lib/form/index/model/model-vibration-fc-form/model-vibration-fc-form`
  - `lib/form/index/model/model-training-form/model-training-form`
  - `lib/form/index/model/model-source-list-form/model-source-list-form`
  - `lib/form/index/hierarchy-detail-form/hierarchy-detail-form`
  - `lib/form/index/alarm-info-form/alarm-info-form`
  - `lib/form/index/alarm-rule-bulk-form/alarm-rule-bulk-form`
  - `lib/form/index/asset-wizard-assets-form/asset-wizard-assets-form`
  - `lib/form/index/asset-wizard-parameters-form/asset-wizard-parameters-form`
  - `lib/form/index/context-config-form/context-config-form`
  - `lib/form/index/asset-template-select-form/asset-template-select-form`
  - `./lib/portlet-ameta/index`
  - `lib/portlet-ameta/index/asset/asset-parameter-auto-spec-table/asset-parameter-auto-spec-table`
  - `lib/portlet-ameta/index/asset/asset-table-by-template/asset-table-by-template`
  - `lib/portlet-ameta/index/asset/asset-template-table/asset-template-table`
  - `lib/portlet-ameta/index/asset/asset-table-using-parameter/asset-table-using-parameter`
  - `lib/portlet-ameta/index/asset/asset-group-list-table/asset-group-list-table`
  - `lib/portlet-ameta/index/model/main-model-table/main-model-table`
  - `lib/portlet-ameta/index/model/model-copy-asset-table/model-copy-asset-table`
  - `lib/portlet-ameta/index/model/model-copy-copied-table/model-copy-copied-table`
  - `lib/portlet-ameta/index/model/model-copy-model-table/model-copy-model-table`
  - `lib/portlet-ameta/index/model/asset-select-by-template/asset-select-by-template`
  - `lib/portlet-ameta/index/model/model-health-index/model-health-index`
  - `lib/portlet-ameta/index/model/main-model/main-model`
  - `lib/portlet-ameta/index/model/model-table/model-table`
  - `lib/portlet-ameta/index/model/model-rul-chart/model-rul-chart`
  - `lib/portlet-ameta/index/model/model-rul-preview-chart/model-rul-preview-chart`
  - `lib/portlet-ameta/index/model/model-rul-preview-chart/model-rul-preview-chart.service`
  - `lib/portlet-ameta/index/model/rul-statuses-info/rul-statuses-info`
  - `lib/portlet-ameta/index/alarm-rule/alarm-rule-preview-modal/alarm-rule-preview-modal`
  - `lib/portlet-ameta/index/alarm-rule/alarm-rule-preview-modal/alarm-rule-preview-modal.service`
  - `lib/portlet-ameta/index/alarm-rule/alarm-rule-table/alarm-rule-table`
  - `lib/portlet-ameta/index/parameter/parameter-table/parameter-table`
  - `lib/portlet-ameta/index/parameter/parameter-group-table/parameter-group-table`
  - `lib/portlet-ameta/index/parameter/parameter-table-by-group/parameter-table-by-group`
  - `lib/portlet-ameta/index/asset-wizard/asset-wizard-asset-table/asset-wizard-asset-table`
  - `lib/portlet-ameta/index/context/context-table/context-table`
  - `lib/portlet-ameta/index/common/asset-hierarchy-tree/asset-hierarchy-tree`
  - `lib/portlet-ameta/index/common/location-selector/location-selector`
  - `lib/portlet-ameta/index/common/hierarchy-asset-tree-selector/select-hierarchy-asset-tree/select-hierarchy-asset-tree`
  - `lib/portlet-ameta/index/common/hierarchy-asset-tree-selector/hierarchy-tree-filter/hierarchy-tree-filter`
  - `lib/portlet-ameta/index/asset-wizard/asset-wizard-preview-table/asset-wizard-preview-table`
  - `lib/portlet-ameta/index/asset-wizard/asset-wizard-hierarchy-title/asset-wizard-hierarchy-title`
  - `lib/portlet-ameta/index/asset-wizard/asset-wizard-asset-hierarchy-tree/asset-wizard-asset-hierarchy-tree`
  - `lib/portlet-ameta/index/asset-wizard/asset-wizard-parameter-table/asset-wizard-parameter-table`
  - `lib/portlet-ameta/index/asset-hierarchy-filter/asset-hierarchy-filter`
  - `./lib/portlet-apm/index`
  - `lib/portlet-apm/index/asset-selector/asset-selector`
  - `lib/portlet-apm/index/health-statuses-info/health-statuses-info`
  - `lib/portlet-apm/index/worker/worker/worker`
  - `lib/portlet-apm/index/worker/worker-select/worker-select`
  - `lib/portlet-apm/index/worker/worker-select/worker-select.state`
  - `lib/portlet-apm/index/worker/editable-worker-field/editable-worker-field`
  - `lib/portlet-apm/index/user-search-selector/user-search-selector`
  - `lib/portlet-apm/index/charts/threshold-change-detector-by-asset-parameter-chart/threshold-change-detector-by-asset-parameter-chart`

### @grandview/web-domain-apm
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 82
- exports:
  - `./lib/pagelet/index`
  - `lib/pagelet/index/alarm-group-list-pagelet/alarm-group-list-pagelet`
  - `lib/pagelet/index/alarm-group-detail-pagelet/alarm-group-detail-pagelet`
  - `lib/pagelet/index/asset-list-pagelet/asset-list-pagelet`
  - `lib/pagelet/index/asset-detail-pagelet/asset-detail-pagelet`
  - `lib/pagelet/index/work-order-list-pagelet/work-order-list-pagelet`
  - `lib/pagelet/index/work-order-detail-pagelet/work-order-detail-pagelet`
  - `./lib/form/index`
  - `lib/form/index/work-order-detail-form/work-order-detail-form`
  - `./lib/portlet/index`
  - `lib/portlet/index/charts/alarm-group-model-chart/alarm-group-model-chart`
  - `lib/portlet/index/charts/alarm-group-parameter-chart/alarm-group-parameter-chart`
  - `lib/portlet/index/charts/asset-fft-parameter-chart-box/asset-fft-parameter-chart-box`
  - `lib/portlet/index/charts/asset-fft-parameter-index-chart-panel/asset-fft-parameter-index-chart-panel`
  - `lib/portlet/index/charts/asset-model-chart-box/asset-model-chart-box`
  - `lib/portlet/index/charts/batch-channel-raw-chart/batch-channel-raw-chart`
  - `lib/portlet/index/charts/batch-channel-parameter-chart/batch-channel-parameter-chart`
  - `lib/portlet/index/charts/trend-fft-chart/trend-fft-chart`
  - `lib/portlet/index/charts/trend-multi-parameter-chart/trend-multi-parameter-chart`
  - `lib/portlet/index/charts/trend-feature-chart/trend-feature-chart`
  - `lib/portlet/index/charts/trend-batch-analysis-chart/trend-batch-analysis-chart`
  - `lib/portlet/index/charts/trend-batch-parameter-chart/trend-batch-parameter-chart`
  - `lib/portlet/index/charts/trend-parameter-chart/trend-parameter-chart`
  - `lib/portlet/index/overview/asset-health-index/asset-health-index`
  - `lib/portlet/index/overview/asset-models-info/asset-models-info`
  - `lib/portlet/index/overview/asset-images-info/asset-images-info`
  - `lib/portlet/index/overview/asset-information/asset-information`
  - `lib/portlet/index/overview/asset-running-state/asset-running-state`
  - `lib/portlet/index/overview/asset-running-rate/asset-running-rate`
  - `lib/portlet/index/overview/asset-rul/asset-rul`
  - `lib/portlet/index/overview/asset-cause-parameter/asset-cause-parameter`
  - `lib/portlet/index/overview/asset-cause-parameter-rate/asset-cause-parameter-rate`
  - `lib/portlet/index/models/model-fc/model-fc`
  - `lib/portlet/index/models/model-health/model-health`
  - `lib/portlet/index/models/model-rul/model-rul`
  - `lib/portlet/index/models/model-caused-chart/model-caused-chart`
  - `lib/portlet/index/asset-selector/asset-selector`
  - `lib/portlet/index/health-statuses-info/health-statuses-info`
  - `lib/portlet/index/asset-table/asset-table-summary/asset-table-summary`
  - `lib/portlet/index/asset-table/asset-table/asset-table`
  - `lib/portlet/index/trend/trend-bookmark/trend-bookmark`
  - `lib/portlet/index/trend/favorite-box/favorite-box`
  - `lib/portlet/index/trend/batch-channel-table/batch-channel-table`
  - `lib/portlet/index/trend/batch-channel-detail/batch-channel-detail`
  - `lib/portlet/index/trend/download-parameter-table/download-parameter-table`
  - `lib/portlet/index/trend/parameter-data-download/parameter-data-download`
  - `lib/portlet/index/alarm/alarm-timeline/alarm-timeline`
  - `lib/portlet/index/alarm/asset-alarm-rate-by-true-alarm/asset-alarm-rate-by-true-alarm`
  - `lib/portlet/index/alarm/asset-alarm-rate-by-action/asset-alarm-rate-by-action`
  - `lib/portlet/index/alarm/asset-alarm-rate-by-type/asset-alarm-rate-by-type`
  - `lib/portlet/index/alarm/asset-alarm-total-count/asset-alarm-total-count`
  - `lib/portlet/index/alarm/alarm-group-table/alarm-group-table`
  - `lib/portlet/index/alarm/alarm-detail-by-alarm-group/alarm-detail-by-alarm-group`
  - `lib/portlet/index/alarm/alarm-table/alarm-table`
  - `lib/portlet/index/alarm/alarm-caused-parameter/alarm-caused-parameter`
  - `lib/portlet/index/alarm/alarm-group-status-change/alarm-group-status-change`
  - `lib/portlet/index/alarm/alarm-count-circle-by-alarm-severity-status/alarm-count-circle-by-alarm-severity-status`
  - `lib/portlet/index/alarm/bell-icons-by-alarm-severity-status/bell-icons-by-alarm-severity-status`
  - `lib/portlet/index/alarm/alarm-rule-table/alarm-rule-table`
  - `lib/portlet/index/work-order/work-order-table/work-order-table`
  - `lib/portlet/index/work-order/worker-order-rate/worker-order-rate`
  - `lib/portlet/index/work-order/work-order-status-change/work-order-status-change`
  - `lib/portlet/index/worker/worker/worker`
  - `lib/portlet/index/worker/worker-select/worker-select`
  - `lib/portlet/index/worker/worker-select/worker-select.state`
  - `lib/portlet/index/worker/editable-worker-field/editable-worker-field`
  - `lib/portlet/index/user-search-selector/user-search-selector`
  - `lib/portlet/index/common/asset-hierarchy-tree/asset-hierarchy-tree`
  - `lib/portlet/index/common/location-selector/location-selector`
  - `lib/portlet/index/common/hierarchy-asset-tree-selector/select-hierarchy-asset-tree/select-hierarchy-asset-tree`
  - `lib/portlet/index/common/hierarchy-asset-tree-selector/hierarchy-tree-filter/hierarchy-tree-filter`
  - `./lib/overview/index`
  - `lib/overview/index/asset-status-information-widget/asset-status-information-widget`
  - `lib/overview/index/asset-health-index-widget/asset-health-index-widget`
  - `lib/overview/index/asset-rul-widget/asset-rul-widget`
  - `lib/overview/index/asset-running-trend-widget/asset-running-trend-widget`
  - `lib/overview/index/asset-running-rate-widget/asset-running-rate-widget`
  - `lib/overview/index/asset-parameter-trace-widget/asset-parameter-trace-widget`
  - `lib/overview/index/asset-parameter-rate-widget/asset-parameter-rate-widget`
  - `lib/overview/index/asset-alarm-widget/asset-alarm-widget`
  - `lib/overview/index/asset-work-order-widget/asset-work-order-widget`
  - `lib/overview/index/template/index`

### @grandview/web-domain-pipeline
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 2
- exports:
  - `./lib/pagelet/index`
  - `lib/pagelet/index/edge-pagelet/edge-pagelet`

### @grandview/web-domain-report
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 6
- exports:
  - `./lib/pagelet/index`
  - `lib/pagelet/index/history-report-pagelet/history-report-pagelet`
  - `lib/pagelet/index/on-demand-report-pagelet/on-demand-report-pagelet`
  - `lib/pagelet/index/print-report-pagelet/print-report-pagelet`
  - `lib/pagelet/index/print-auto-asset-report-pagelet/print-auto-asset-report-pagelet`
  - `lib/pagelet/index/print-auto-location-report-pagelet/print-auto-location-report-pagelet`

### @grandview/web-domain-studio
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 13
- exports:
  - `./lib/pagelet/index`
  - `lib/pagelet/index/dashboard-pagelet/dashboard-pagelet`
  - `lib/pagelet/index/layout-pagelet/layout-pagelet`
  - `lib/pagelet/index/report-pagelet/report-pagelet`
  - `./lib/builder/dashboard-builder/index`
  - `lib/builder/dashboard-builder/index/dashboard-builder/dashboard-builder`
  - `lib/builder/dashboard-builder/index/dashboard-list/dashboard-list`
  - `./lib/builder/layout-builder/index`
  - `lib/builder/layout-builder/index/layout-builder/layout-builder`
  - `lib/builder/layout-builder/index/layout-list/layout-list`
  - `./lib/builder/report-builder/index`
  - `lib/builder/report-builder/index/report-builder/report-builder`
  - `lib/builder/report-builder/index/report-list/report-list`

### @grandview/web-domain-system
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 27
- exports:
  - `./lib/pagelet/index`
  - `lib/pagelet/index/global-config-pagelet/global-config-pagelet`
  - `lib/pagelet/index/license-pagelet/license-pagelet`
  - `lib/pagelet/index/user-group-pagelet/user-group-pagelet`
  - `lib/pagelet/index/user-pagelet/user-pagelet`
  - `lib/pagelet/index/datasource-pagelet/datasource-pagelet`
  - `lib/pagelet/index/plugin-store-pagelet/plugin-store-pagelet`
  - `lib/pagelet/index/permission-pagelet/permission-pagelet`
  - `./lib/form/index`
  - `lib/form/index/license-form/license-form`
  - `lib/form/index/user-form/user-form`
  - `lib/form/index/user-group-select-form/user-group-select-form`
  - `lib/form/index/user-group-form/user-group-form`
  - `lib/form/index/user-select-form/user-select-form`
  - `lib/form/index/plugin-form/plugin-form`
  - `lib/form/index/type-form/type-form`
  - `lib/form/index/permission-form/permission-form`
  - `lib/form/index/permission-menu-form/permission-menu-form`
  - `./lib/portlet/index`
  - `lib/portlet/index/user-group-table/user-group-table`
  - `lib/portlet/index/user-table/user-table`
  - `lib/portlet/index/user-selected-table/user-selected-table`
  - `lib/portlet/index/user-group-selected-table/user-group-selected-table`
  - `lib/portlet/index/plugin-table/plugin-table`
  - `lib/portlet/index/plugin-histories/plugin-histories`
  - `lib/portlet/index/type-table/type-table`
  - `lib/portlet/index/permission-table/permission-table`

### @grandview/web-domain-user
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 7
- exports:
  - `./lib/pagelet/index`
  - `lib/pagelet/index/my-profile-pagelet/my-profile-pagelet`
  - `./lib/form/index`
  - `lib/form/index/my-profile-form/my-profile-form`
  - `./lib/portlet/index`
  - `lib/portlet/index/change-password/change-password`
  - `lib/portlet/index/user-group-selected-table/user-group-selected-table`

### @grandview/web-layout-micro-app
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 26
- exports:
  - `./lib/sidebar/micro-app-sidebar`
  - `./lib/flexlayout`
  - `lib/flexlayout/view/Layout`
  - `lib/flexlayout/model/Action`
  - `lib/flexlayout/model/Actions`
  - `lib/flexlayout/model/BorderNode`
  - `lib/flexlayout/model/BorderSet`
  - `lib/flexlayout/model/ICloseType`
  - `lib/flexlayout/model/IDraggable`
  - `lib/flexlayout/model/IDropTarget`
  - `lib/flexlayout/model/IJsonModel`
  - `lib/flexlayout/model/Model`
  - `lib/flexlayout/model/Node`
  - `lib/flexlayout/model/RowNode`
  - `lib/flexlayout/model/SplitterNode`
  - `lib/flexlayout/model/TabNode`
  - `lib/flexlayout/model/TabSetNode`
  - `lib/flexlayout/DockLocation`
  - `lib/flexlayout/DragDrop`
  - `lib/flexlayout/DropInfo`
  - `lib/flexlayout/I18nLabel`
  - `lib/flexlayout/Orientation`
  - `lib/flexlayout/Rect`
  - `lib/flexlayout/Types`
  - `./lib/main/micro-app-main`
  - `./lib/micro-app`

### @grandview/web-layout-portal
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 4
- exports:
  - `./lib/header/header`
  - `./lib/header/alarm/alarm-container`
  - `./lib/body/body`
  - `./lib/portal/portal`

### @grandview/web-login-account
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 5
- exports:
  - `./lib/forget-password/forget-password`
  - `./lib/create-account/create-account`
  - `./lib/auth-login.service`
  - `./lib/auth-menu.service`
  - `./lib/sso-auth-login.service`

### @grandview/web-reportlet-apm
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 77
- exports:
  - `./lib/i18n-loader`
  - `./lib/components/asset-base/asset/asset-info-reportlet/asset-info-reportlet`
  - `./lib/components/asset-base/asset/asset-image-reportlet/asset-image-reportlet`
  - `./lib/components/asset-base/asset/asset-models-info-reportlet/asset-models-info-reportlet`
  - `./lib/components/asset-base/asset/asset-alarm-summary-reportlet/asset-alarm-summary-reportlet`
  - `./lib/components/asset-base/asset/asset-rul-reportlet/asset-rul-reportlet`
  - `./lib/components/asset-base/asset/asset-running-reportlet/asset-running-reportlet`
  - `./lib/components/asset-base/asset/asset-health-index-reportlet/asset-health-index-reportlet`
  - `./lib/components/asset-base/alarm/asset-alarm-total-reportlet/asset-alarm-total-reportlet`
  - `./lib/components/asset-base/alarm/asset-alarm-type-reportlet/asset-alarm-type-reportlet`
  - `./lib/components/asset-base/alarm/asset-alarm-status-reportlet/asset-alarm-status-reportlet`
  - `./lib/components/asset-base/alarm/asset-alarm-quality-reportlet/asset-alarm-quality-reportlet`
  - `./lib/components/asset-base/parameter/worst-parameter-trend-reportlet/worst-parameter-trend-reportlet`
  - `./lib/components/asset-base/parameter/worst-parameter-rate-reportlet/worst-parameter-rate-reportlet`
  - `./lib/components/asset-base/parameter/worst-parameter-chart-reportlet/worst-parameter-chart-reportlet`
  - `./lib/components/asset-base/parameter/parameter-charts-reportlet/parameter-charts-reportlet`
  - `./lib/components/asset-base/parameter/parameter-trace-chart-reportlet/parameter-trace-chart-reportlet`
  - `./lib/components/location-base/location/location-info/location-info-reportlet`
  - `./lib/components/location-base/asset/asset-status-summary-reportlet/asset-status-summary-reportlet`
  - `./lib/components/location-base/asset/asset-status-count-reportlet/asset-status-count-reportlet`
  - `./lib/components/location-base/asset/asset-health-heatmap-reportlet/asset-health-heatmap-reportlet`
  - `./lib/components/location-base/asset/asset-health-list-reportlet/asset-health-list-reportlet`
  - `./lib/components/location-base/sensor/sensor-summary-status-reportlet/sensor-summary-status-reportlet`
  - `./lib/components/location-base/sensor/sensor-connection-status-reportlet/sensor-connection-status-reportlet`
  - `./lib/components/location-base/sensor/sensor-worst-top5-reportlet/sensor-worst-top5-reportlet`
  - `./lib/components/location-base/alarm/alarm-heatmap-timeline-reportlet/alarm-heatmap-timeline-reportlet`
  - `./lib/components/shared-base/alarm/alarm-type-reportlet/alarm-type-reportlet`
  - `./lib/components/shared-base/alarm/alarm-severity-reportlet/alarm-severity-reportlet`
  - `./lib/components/shared-base/alarm/alarm-count-trend-reportlet/alarm-count-trend-reportlet`
  - `./lib/components/shared-base/kpi/kpi-list-reportlet/kpi-list-reportlet`
  - `./lib/components/shared-base/kpi/kpi-map-reportlet/kpi-map-reportlet`
  - `./lib/components/shared-base/kpi/kpi-summary-reportlet/kpi-summary-reportlet`
  - `./lib/components/shared-base/worker-order/worker-order-action-rate-reportlet/worker-order-action-rate-reportlet`
  - `./lib/components/shared-base/worker-order/worker-order-step-rate-reportlet/worker-order-step-rate-reportlet`
  - `./lib/components/shared-base/worker-order/worker-order-type-rate-reportlet/worker-order-type-rate-reportlet`
  - `./lib/config-forms/asset-base/index`
  - `lib/config-forms/asset-base/index/asset-info-style.config`
  - `lib/config-forms/asset-base/index/asset-info-style.form`
  - `lib/config-forms/asset-base/index/asset-image-style.config`
  - `lib/config-forms/asset-base/index/asset-image-style.form`
  - `lib/config-forms/asset-base/index/asset-model-info-style.config`
  - `lib/config-forms/asset-base/index/asset-model-info-style.form`
  - `lib/config-forms/asset-base/index/asset-model-info-data/asset-model-info-data.config`
  - `lib/config-forms/asset-base/index/asset-model-info-data/asset-model-info-data.form`
  - `lib/config-forms/asset-base/index/asset-parameter-trace-data/asset-parameter-trace-data.config`
  - `lib/config-forms/asset-base/index/asset-parameter-trace-data/asset-parameter-trace-data.form`
  - `lib/config-forms/asset-base/index/asset-running-style.config`
  - `lib/config-forms/asset-base/index/asset-running-style.form`
  - `lib/config-forms/asset-base/index/worst-parameter-rate-style.config`
  - `lib/config-forms/asset-base/index/worst-parameter-rate-style.form`
  - `./lib/config-forms/location-base/index`
  - `lib/config-forms/location-base/index/asset-status-summary-style.config`
  - `lib/config-forms/location-base/index/asset-status-summary-style.form`
  - `lib/config-forms/location-base/index/asset-status-count-style.config`
  - `lib/config-forms/location-base/index/asset-status-count-style.form`
  - `lib/config-forms/location-base/index/asset-health-heatmap-style.config`
  - `lib/config-forms/location-base/index/asset-health-heatmap-style.form`
  - `lib/config-forms/location-base/index/location-alarm-heatmap-style.config`
  - `lib/config-forms/location-base/index/location-alarm-heatmap-style.form`
  - `lib/config-forms/location-base/index/location-info-style.config`
  - `lib/config-forms/location-base/index/location-info-style.form`
  - `lib/config-forms/location-base/index/sensor-summary-status-style.config`
  - `lib/config-forms/location-base/index/sensor-summary-status-style.form`
  - `lib/config-forms/location-base/shared-base/health-criteria-data.config`
  - `lib/config-forms/location-base/shared-base/health-criteria-data.form`
  - `lib/config-forms/location-base/index/location-kpi-list-data/kpi-list-data.config`
  - `lib/config-forms/location-base/index/location-kpi-list-data/kpi-list-data.form`
  - `./lib/config-forms/shared-base/index`
  - `lib/config-forms/shared-base/index/location-alarm-type-style.config`
  - `lib/config-forms/shared-base/index/location-alarm-type-style.form`
  - `lib/config-forms/shared-base/index/location-alarm-trend-style.config`
  - `lib/config-forms/shared-base/index/location-alarm-trend-style.form`
  - `lib/config-forms/shared-base/index/asset-workorder-step-rate-style.config`
  - `lib/config-forms/shared-base/index/asset-workorder-step-rate-style.form`
  - `./lib/config-forms/config-data.form`
  - `./lib/config-forms/config-style.form`
  - `./lib/config-forms/config-style.type`

### @grandview/web-shared-aim
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 3
- exports:
  - `./lib/data-api/index`
  - `lib/data-api/index/models`
  - `lib/data-api/index/hooks`

### @grandview/web-shared-ameta
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 71
- exports:
  - `./lib/domain-api/index`
  - `lib/domain-api/index/apis/categorys-asset.api`
  - `lib/domain-api/index/hooks/model-status.hook`
  - `lib/domain-api/index/hooks/location-asset-tree.hook`
  - `lib/domain-api/index/hooks/location-type.hook`
  - `lib/domain-api/index/hooks/kpi-status.hook`
  - `lib/domain-api/index/hooks/asset/asset-detail.hook`
  - `lib/domain-api/index/hooks/asset/asset-worker.hook`
  - `lib/domain-api/index/hooks/asset/asset.hook`
  - `lib/domain-api/index/hooks/alarm/alarm-rule.hook`
  - `lib/domain-api/index/hooks/alarm/alarm-model.hook`
  - `lib/domain-api/index/hooks/sensor-status.hook`
  - `lib/domain-api/index/hooks/user/user-detail.hook`
  - `lib/domain-api/index/hooks/user/user-group-detail.hook`
  - `lib/domain-api/index/hooks/user/my-profile.hook`
  - `lib/domain-api/index/hooks/user/user.hook`
  - `./lib/chart/index`
  - `lib/chart/index/alarm/alarm-health-index-chart/alarm-health-index-chart`
  - `lib/chart/index/alarm/alarm-parameter-chart/alarm-parameter-chart`
  - `lib/chart/index/asset/asset-status-chart/asset-status-chart`
  - `lib/chart/index/asset/running-trend-chart/running-trend-chart`
  - `lib/chart/index/asset/asset-health-heatmap-chart/asset-health-heatmap-chart`
  - `lib/chart/index/asset/asset-health-heatmap-chart/health-status-box/health-status-box`
  - `lib/chart/index/parameter/fft-chart/fft-chart`
  - `lib/chart/index/parameter/top-cause-parameter-chart/top-cause-parameter-chart`
  - `lib/chart/index/parameter/top-cause-parameter-rate-chart/top-cause-parameter-rate-chart`
  - `lib/chart/index/parameter/status-chart/status-chart`
  - `lib/chart/index/parameter/status-chart/status-chart-option`
  - `lib/chart/index/parameter/alarm-chart/alarm-chart`
  - `lib/chart/index/parameter/quantity-chart/quantity-chart`
  - `lib/chart/index/parameter/guage-chart/guage-chart`
  - `lib/chart/index/parameter/temperature-chart/temperature-chart`
  - `lib/chart/index/parameter/multi-range-trace-chart/multi-range-trace-chart`
  - `lib/chart/index/parameter/trace-chart/trace-chart`
  - `lib/chart/index/parameter/trace-zoom-chart/trace-zoom-chart`
  - `lib/chart/index/parameter/parameter-summary/parameter-summary`
  - `lib/chart/index/model/asset-score-chart/asset-score-chart`
  - `lib/chart/index/model/health-index-chart/health-index-chart`
  - `lib/chart/index/model/model-health-index-chart/model-health-index-chart`
  - `lib/chart/index/model/rul-band-chart/rul-band-chart`
  - `lib/chart/index/model/asset-score-zoom-chart/asset-score-zoom-chart`
  - `lib/chart/index/batch/batch-box-plot-chart/batch-box-plot-chart`
  - `lib/chart/index/batch/batch-line-chart/batch-line-chart`
  - `lib/chart/index/batch/batch-channel-raw-chart/batch-channel-raw-chart`
  - `lib/chart/index/batch/batch-analysis-chart/batch-analysis-chart`
  - `lib/chart/index/batch/batch-golden-tunnel-chart/batch-golden-tunnel-chart`
  - `./lib/data-api/index`
  - `lib/data-api/index/hooks/parameter-group.hook`
  - `lib/data-api/index/hooks/parameter.hook`
  - `lib/data-api/index/hooks/asset-hierarchy.hook`
  - `lib/data-api/index/hooks/asset-wizard.hook`
  - `lib/data-api/index/hooks/data-source.hook`
  - `lib/data-api/index/hooks/asset.hook`
  - `lib/data-api/index/hooks/asset-template.hook`
  - `lib/data-api/index/hooks/context.hook`
  - `lib/data-api/index/hooks/asset-group.hook`
  - `lib/data-api/index/hooks/model.hook`
  - `lib/data-api/index/hooks/analysis-model.hook`
  - `lib/data-api/index/hooks/user-group.hook`
  - `lib/data-api/index/models`
  - `./lib/service/index`
  - `lib/service/index/parameter.service`
  - `lib/service/index/asset-hierarchy.service`
  - `lib/service/index/model.service`
  - `lib/service/index/analysis-model.service`
  - `lib/service/index/multi-range-custom-property.service`
  - `./lib/component/index`
  - `lib/component/index/asset-template-selector/asset-template-selector`
  - `lib/component/index/editable-asset-template-field/editable-asset-template-field`
  - `lib/component/index/asset-hierarchy-node/asset-hierarchy-node`
  - `./lib/state/model-revision.state`

### @grandview/web-shared-apm
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 71
- exports:
  - `./lib/qrcode/index`
  - `lib/qrcode/index/create-asset-overview-qr-code/create-asset-overview-qr-code`
  - `./lib/domain-api/index`
  - `lib/domain-api/index/hooks/model-status.hook`
  - `lib/domain-api/index/hooks/location-asset-tree.hook`
  - `lib/domain-api/index/hooks/location-type.hook`
  - `lib/domain-api/index/hooks/model.hook`
  - `lib/domain-api/index/hooks/kpi-status.hook`
  - `lib/domain-api/index/hooks/asset/asset-detail.hook`
  - `lib/domain-api/index/hooks/asset/asset-worker.hook`
  - `lib/domain-api/index/hooks/asset/asset-status.hook`
  - `lib/domain-api/index/hooks/asset/asset.hook`
  - `lib/domain-api/index/hooks/alarm/alarm-rule.hook`
  - `lib/domain-api/index/hooks/alarm/alarm-model.hook`
  - `lib/domain-api/index/hooks/sensor-status.hook`
  - `lib/domain-api/index/hooks/user/user-detail.hook`
  - `lib/domain-api/index/hooks/user/user-group-detail.hook`
  - `lib/domain-api/index/hooks/user/my-profile.hook`
  - `lib/domain-api/index/hooks/user/user.hook`
  - `./lib/icon-button/index`
  - `lib/icon-button/index/download-chart-data/download-icon-asset-parameter-chart`
  - `./lib/data-api/index`
  - `lib/data-api/index/apis/work-order.api`
  - `lib/data-api/index/apis/meta-data-source.api`
  - `lib/data-api/index/apis/param-values.api`
  - `lib/data-api/index/apis/vibration-fft.api`
  - `lib/data-api/index/hooks/work-order.hook`
  - `lib/data-api/index/hooks/meta-data-source.hook`
  - `lib/data-api/index/hooks/param-values.hook`
  - `lib/data-api/index/hooks/vibration-fft-parameter.hook`
  - `lib/data-api/index/models/meta-data-source.model`
  - `./lib/chart/index`
  - `lib/chart/index/alarm/alarm-health-index-chart/alarm-health-index-chart`
  - `lib/chart/index/alarm/alarm-parameter-chart/alarm-parameter-chart`
  - `lib/chart/index/asset/asset-status-chart/asset-status-chart`
  - `lib/chart/index/asset/running-trend-chart/running-trend-chart`
  - `lib/chart/index/asset/asset-health-heatmap-chart/asset-health-heatmap-chart`
  - `lib/chart/index/asset/asset-health-heatmap-chart/health-status-box/health-status-box`
  - `lib/chart/index/parameter/fft-chart/fft-chart`
  - `lib/chart/index/parameter/top-cause-parameter-chart/top-cause-parameter-chart`
  - `lib/chart/index/parameter/top-cause-parameter-rate-chart/top-cause-parameter-rate-chart`
  - `lib/chart/index/parameter/status-chart/status-chart`
  - `lib/chart/index/parameter/status-chart/status-chart-option`
  - `lib/chart/index/parameter/alarm-chart/alarm-chart`
  - `lib/chart/index/parameter/quantity-chart/quantity-chart`
  - `lib/chart/index/parameter/guage-chart/guage-chart`
  - `lib/chart/index/parameter/temperature-chart/temperature-chart`
  - `lib/chart/index/parameter/trace-chart/trace-chart`
  - `lib/chart/index/parameter/merged-parameter-chart/merged-parameter-chart`
  - `lib/chart/index/parameter/multi-trace-chart/multi-trace-chart`
  - `lib/chart/index/parameter/trace-zoom-chart/trace-zoom-chart`
  - `lib/chart/index/parameter/fft-index-chart/fft-index-chart`
  - `lib/chart/index/parameter/parameter-summary/parameter-summary`
  - `lib/chart/index/parameter/parameter-chart.service`
  - `lib/chart/index/model/asset-score-chart/asset-score-chart`
  - `lib/chart/index/model/health-index-chart/health-index-chart`
  - `lib/chart/index/model/model-health-index-chart/model-health-index-chart`
  - `lib/chart/index/model/parameter-radar-chart/parameter-radar-chart`
  - `lib/chart/index/model/rul-band-chart/rul-band-chart`
  - `lib/chart/index/model/asset-score-zoom-chart/asset-score-zoom-chart`
  - `lib/chart/index/model/model-health-bar/model-health-bar`
  - `lib/chart/index/batch/batch-box-plot-chart/batch-box-plot-chart`
  - `lib/chart/index/batch/batch-line-chart/batch-line-chart`
  - `lib/chart/index/batch/batch-channel-raw-chart/batch-channel-raw-chart`
  - `lib/chart/index/batch/batch-analysis-chart/batch-analysis-chart`
  - `lib/chart/index/batch/batch-golden-tunnel-chart/batch-golden-tunnel-chart`
  - `./lib/state/index`
  - `lib/state/index/tabkey/chart-header.state`
  - `lib/state/index/tabkey/menu-permission.state`
  - `./lib/types/trend-tab-type`
  - `./lib/service/alarm.service`

### @grandview/web-shared-platform
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 51
- exports:
  - `./lib/portlet-common/index`
  - `lib/portlet-common/index/asset-hierarchy-tree/asset-hierarchy-tree`
  - `lib/portlet-common/index/location-selector/location-selector`
  - `lib/portlet-common/index/hierarchy-asset-tree-selector/object-mapping-hierarchy-asset-tree/object-mapping-hierarchy-asset-tree`
  - `lib/portlet-common/index/hierarchy-asset-tree-selector/select-hierarchy-asset-tree/select-hierarchy-asset-tree`
  - `lib/portlet-common/index/hierarchy-asset-tree-selector/hierarchy-tree-filter/hierarchy-tree-filter`
  - `lib/portlet-common/index/multi-location-asset-selector/multi-location-asset-selector`
  - `lib/portlet-common/index/location-multi-asset-tree/location-multi-asset-tree`
  - `./lib/chart/index`
  - `lib/chart/index/asset/asset-status-chart/asset-status-chart`
  - `lib/chart/index/asset/asset-health-heatmap-chart/asset-health-heatmap-chart`
  - `lib/chart/index/asset/asset-health-heatmap-chart/health-status-box/health-status-box`
  - `lib/chart/index/parameter/fft-chart/fft-chart`
  - `lib/chart/index/parameter/top-cause-parameter-chart/top-cause-parameter-chart`
  - `lib/chart/index/parameter/top-cause-parameter-rate-chart/top-cause-parameter-rate-chart`
  - `lib/chart/index/parameter/status-chart/status-chart`
  - `lib/chart/index/parameter/status-chart/status-chart-option`
  - `lib/chart/index/parameter/alarm-chart/alarm-chart`
  - `lib/chart/index/parameter/quantity-chart/quantity-chart`
  - `lib/chart/index/parameter/guage-chart/guage-chart`
  - `lib/chart/index/parameter/temperature-chart/temperature-chart`
  - `lib/chart/index/parameter/trace-chart/trace-chart`
  - `lib/chart/index/parameter/multi-trace-chart/multi-trace-chart`
  - `lib/chart/index/parameter/trace-zoom-chart/trace-zoom-chart`
  - `lib/chart/index/parameter/parameter-summary/parameter-summary`
  - `lib/chart/index/parameter/parameter-chart.service`
  - `lib/chart/index/model/asset-score-chart/asset-score-chart`
  - `lib/chart/index/model/health-index-chart/health-index-chart`
  - `lib/chart/index/model/model-health-index-chart/model-health-index-chart`
  - `lib/chart/index/model/parameter-radar-chart/parameter-radar-chart`
  - `lib/chart/index/model/rul-band-chart/rul-band-chart`
  - `lib/chart/index/model/asset-score-zoom-chart/asset-score-zoom-chart`
  - `lib/chart/index/model/model-health-bar/model-health-bar`
  - `./lib/websocket/sockjs.service`
  - `./lib/data-api/index`
  - `lib/data-api/index/hooks/datasource.hook`
  - `lib/data-api/index/hooks/edge-instance.hook`
  - `lib/data-api/index/hooks/report.hook`
  - `lib/data-api/index/hooks/user.hook`
  - `lib/data-api/index/hooks/user-group.hook`
  - `lib/data-api/index/hooks/plugin.hook`
  - `lib/data-api/index/hooks/dashboard.hook`
  - `lib/data-api/index/hooks/layout.hook`
  - `lib/data-api/index/hooks/license.hook`
  - `lib/data-api/index/hooks/system.hook`
  - `lib/data-api/index/hooks/location-asset-tree.hook`
  - `lib/data-api/index/hooks/asset-detail.hook`
  - `lib/data-api/index/hooks/location-info.hook`
  - `lib/data-api/index/models`
  - `lib/data-api/index/hooks/permission.hook`
  - `./lib/services/launch-print-app.service`

### @grandview/web-template-dashboard
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 1
- exports:
  - `./lib/dashboard-template/dashboard-template`

### @grandview/web-template-layout
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 1
- exports:
  - `./lib/layout-template/layout-template`

### @grandview/web-template-report
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 2
- exports:
  - `./lib/report-page-template/report-page-template`
  - `./lib/print-report-page-template/print-report-page-template`

### @grandview/web-widget-apm
- version: `2.1.0-beta.1`
- main: `./index.es.js`
- module: `./index.es.js`
- types: `./index.d.ts`
- export count: 23
- exports:
  - `./lib/i18n-loader`
  - `./lib/components/layouts/default-layout-widget/default-layout-widget`
  - `./lib/components/assets/asset-health-heatmap-widget/asset-health-heatmap-widget`
  - `./lib/components/assets/asset-status-summary-widget/asset-status-summary-widget`
  - `./lib/components/assets/asset-status-heatmap-widget/asset-status-heatmap-widget`
  - `./lib/components/assets/asset-health-list-widget/asset-health-list-widget`
  - `./lib/components/assets/asset-health-list-modal-widget/asset-health-list-modal-widget`
  - `./lib/components/assets/asset-parameters-widget/asset-parameters-widget`
  - `./lib/components/maps/geo-status-summary-widget/geo-status-summary-widget`
  - `./lib/components/maps/locations-status-summary-widget/locations-status-summary-widget`
  - `./lib/components/maps/kpi-status-summary-widget/kpi-status-summary-widget`
  - `./lib/components/alarms/alarm-type-widget/alarm-type-widget`
  - `./lib/components/alarms/alarm-heatmap-widget/alarm-heatmap-widget`
  - `./lib/components/alarms/alarm-severity-widget/alarm-severity-widget`
  - `./lib/components/alarms/alarm-count-trend-widget/alarm-count-trend-widget`
  - `./lib/components/alarms/alarm-list-widget/alarm-list-widget`
  - `./lib/components/alarms/alarm-common-list-widget/alarm-common-list-widget`
  - `./lib/components/sensors/sensor-overall-status-widget/sensor-overall-status-widget`
  - `./lib/components/sensors/sensor-summary-status-widget/sensor-summary-status-widget`
  - `./lib/components/sensors/asset-sensor-worst-top5-widget/asset-sensor-worst-top5-widget`
  - `./lib/components/sensors/asset-sensor-collect-rate-list-widget/asset-sensor-collect-rate-list-widget`
  - `./lib/components/sensors/parameter-sensor-collect-rate-list-widget/parameter-sensor-collect-rate-list-widget`
  - `./lib/components/demos/sk-hynix/widget/alarm-group-detail-widget/alarm-group-detail-widget`


---

## 15. 부록 B: 핵심 참고 파일

- 런타임 공유 라이브러리: `node_modules/@grandview/shared/index.d.ts`
- 공통 컴포넌트: `node_modules/@grandview/component/lib/index.d.ts`
- Plugin 테스트 툴: `node_modules/@grandview/plugin-testbed/index.d.ts`
- Portal 진입: `apps/portal/web/src/main.ts`
- Dashboard Host 진입: `apps/micro-apps/dashboard/src/main.ts`
- Report Host 진입: `apps/micro-apps/report/src/main.ts`
- Studio Host 진입: `apps/micro-apps/studio/src/main.ts`
- Print Host 진입: `apps/micro-apps/print/src/main.ts`
- APM Widget Plugin MF: `plugins/widget/apm-default-widgets/module-federation.config.ts`
- Seed Widget Plugin MF: `plugins/widget/seed-default-widgets/module-federation.config.ts`
- APM Reportlet Plugin MF: `plugins/reportlet/apm-default-reportlets/module-federation.config.ts`
- Seed Reportlet Plugin MF: `plugins/reportlet/seed-default-reportlets/module-federation.config.ts`
- 공통 MF 공유 정책: `module-federation.base.config.js`
- 빌드/배포 스크립트: `gv-build-application.sh`, `gv-build-plugins.sh`, `gv-9-replace-production-env.sh`

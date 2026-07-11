---
title: 안 여는 S3 객체가 매달 최고 요금을 내는 문제, Lifecycle 정책
date: 2026-06-17 19:23:00 +09:00
categories: [Infra]
tags: [aws, s3, lifecycle, cost]
description: 접근 빈도가 낮은 객체를 STANDARD에 방치하는 비용 낭비를 Lifecycle 전환과 Intelligent-Tiering으로 줄이는 기준을 정리했다.
---

S3 비용 청구서를 뜯어보는 법을 공부했습니다. 대부분의 낭비는 단순한 사실 하나에서 나옵니다. 거의 안 여는 객체가 가장 비싼 스토리지 클래스에 눌러앉아 있다는 것.

## 스토리지 클래스는 접근 빈도의 가격표다

S3 요금은 클래스별로 보관 단가와 조회 비용의 트레이드오프입니다. 대략의 구도는 이렇습니다.

- STANDARD: 보관 비싸고 조회 무료에 가까움 — 자주 읽는 데이터용
- STANDARD-IA: 보관 ~45% 저렴, 조회 시 GB당 비용 — 월 1회 미만 접근
- Glacier 계열: 보관이 수 배~수십 배 저렴, 꺼내는 데 비용·시간 — 아카이브

로그, 백업, 오래된 업로드 파일처럼 "쓰고 나서 거의 안 읽는" 데이터가 STANDARD에 쌓여 있으면 그 차액을 매달 내는 셈입니다.

## Lifecycle로 자동 강등

```json
{
  "Rules": [{
    "ID": "archive-logs",
    "Filter": { "Prefix": "logs/" },
    "Status": "Enabled",
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER" }
    ],
    "Expiration": { "Days": 365 }
  }]
}
```

30일 후 IA로, 90일 후 Glacier로 자동 전환하고 1년 후 삭제하는 규칙입니다. 접근 패턴을 모르겠으면 Intelligent-Tiering이 답입니다. 객체별 접근을 관찰해 자동으로 계층을 옮겨주고, 모니터링 비용(객체당 소액)만 받습니다. 작은 객체가 수억 개라면 이 모니터링 비용이 역효과일 수 있으니 객체 크기 분포는 확인해야 합니다.

시작은 S3 Storage Lens나 Storage Class Analysis로 "무엇이 얼마나 안 읽히는지" 데이터를 확인하는 것부터입니다. IA·Glacier는 최소 보관 기간(30/90일)과 조회 비용이 있어, 자주 읽는 데이터를 잘못 내리면 오히려 비쌉니다.

## 정리

- S3 비용 최적화의 본질은 "접근 빈도와 가격표 맞추기"다.
- 로그·백업 같은 라이프사이클이 명확한 데이터는 규칙으로 자동 강등한다.
- 패턴을 모르면 Intelligent-Tiering, 알면 명시적 Transition이 더 싸다.

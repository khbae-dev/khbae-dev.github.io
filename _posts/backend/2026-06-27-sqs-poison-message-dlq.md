---
title: 메시지 한 통이 SQS 큐 전체를 막는 poison message 대응
date: 2026-06-27 19:22:00 +09:00
categories: [Backend]
tags: [sqs, aws, dlq, queue]
description: 처리 불가능한 메시지가 무한 재배달되며 컨슈머를 반복해서 죽이는 poison message 문제와, DLQ·부분 배치 실패 응답으로 격리하는 방법을 정리했다.
---

큐 기반 시스템의 고전 장애 하나를 정리했습니다. 딱 한 통의 잘못된 메시지가 큐 전체를 멈추게 하는 poison message 문제입니다.

## 무한 재배달 루프

SQS에서 컨슈머가 메시지를 받고 삭제하지 못한 채 죽으면(파싱 불가 페이로드로 예외 발생 등), visibility timeout이 지나 메시지가 다시 큐에 나타납니다. 그 메시지를 받은 컨슈머가 또 죽고, 다시 나타나고… 이 루프가 도는 동안 정상 메시지 처리량은 급감하고 큐는 적체됩니다.

## 격리 장치 두 가지

**1. DLQ + maxReceiveCount.** 재수신 횟수가 상한을 넘으면 SQS가 자동으로 메시지를 Dead Letter Queue로 옮깁니다.

```json
{
  "RedrivePolicy": {
    "deadLetterTargetArn": "arn:aws:sqs:...:orders-dlq",
    "maxReceiveCount": 5
  }
}
```

문제 메시지는 5회 시도 후 격리되고, 본 큐는 정상으로 돌아갑니다. DLQ에 쌓인 메시지는 알람을 걸어 원인 분석 후 재주입(redrive)합니다.

**2. 부분 배치 실패 응답.** 배치(예: 10건)로 소비할 때 1건 실패로 배치 전체를 실패 처리하면 정상 9건까지 재배달됩니다. 실패한 것만 지정해서 반환하면 정상 메시지는 삭제됩니다.

```javascript
// Lambda + SQS: 실패 건만 보고
return {
  batchItemFailures: failedIds.map(id => ({ itemIdentifier: id })),
};
```

(Lambda 이벤트 소스 설정에서 `ReportBatchItemFailures` 활성화가 전제입니다.)

## 정리

- 큐를 만들 때 DLQ 연결과 maxReceiveCount는 기본 옵션이지 선택이 아니다.
- 배치 소비라면 부분 실패 응답까지 해야 정상 메시지 중복 처리를 막는다.
- DLQ 깊이에 알람을 걸어두면 poison message가 "조용한 장애"가 되지 않는다.

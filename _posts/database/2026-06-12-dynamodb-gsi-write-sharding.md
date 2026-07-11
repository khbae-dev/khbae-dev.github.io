---
title: DynamoDB 쓰기 스로틀, 테이블이 아니라 GSI 핫파티션이 범인일 때
date: 2026-06-12 16:00:00 +09:00
categories: [Database]
tags: [dynamodb, aws, partition, sharding]
description: GSI 파티션 키의 카디널리티가 낮아 생기는 핫파티션이 테이블 쓰기까지 막는 구조와, write sharding으로 분산하는 방법을 정리했다.
---

DynamoDB에서 프로비저닝 용량은 충분한데 `WriteThrottleEvents`가 치솟는 사례를 공부했습니다. 원인이 테이블이 아니라 GSI에 있다는 게 포인트였습니다.

## GSI가 테이블 쓰기를 막는 구조

DynamoDB는 파티션 키의 해시로 데이터를 분산합니다. 문제는 GSI의 파티션 키가 `status` 같은 저카디널리티 값일 때입니다. 대부분의 아이템이 `status=PENDING`이라면 GSI 쓰기가 한 파티션에 몰리고, 파티션당 처리량 한도(초당 1,000 WCU)에 걸립니다. 더 아픈 건, GSI 쓰기가 스로틀되면 베이스 테이블 쓰기까지 함께 거부된다는 점입니다.

## write sharding으로 분산

파티션 키에 무작위 suffix를 붙여 강제로 카디널리티를 만듭니다.

```javascript
// 쓰기: status 뒤에 0~9 suffix를 붙여 10개 파티션으로 분산
const shard = Math.floor(Math.random() * 10);
item.gsiPk = `PENDING#${shard}`;
```

```javascript
// 조회: 10개 샤드를 병렬 Query 후 병합
const results = await Promise.all(
  [...Array(10).keys()].map(s =>
    ddb.query({
      IndexName: "status-index",
      KeyConditionExpression: "gsiPk = :pk",
      ExpressionAttributeValues: { ":pk": `PENDING#${s}` },
    })
  )
);
```

쓰기는 자연 분산되고, 조회는 샤드 수만큼의 병렬 Query로 대체됩니다. 샤드 수는 목표 쓰기 처리량 ÷ 파티션 한도로 잡으면 됩니다.

## 정리

- DynamoDB 스로틀은 총 용량이 아니라 "파티션 단위 쏠림"을 먼저 본다.
- GSI 스로틀은 테이블 쓰기까지 전파된다. GSI 키 설계가 테이블만큼 중요하다.
- 저카디널리티 키는 suffix 샤딩으로 강제 분산하고, 읽기는 병렬 Query로 흡수한다.

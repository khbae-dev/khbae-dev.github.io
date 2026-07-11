---
title: Kafka 컨슈머가 자꾸 멈추고 랙이 쌓일 때, 리밸런스 폭풍
date: 2026-06-22 19:24:00 +09:00
categories: [Backend]
tags: [kafka, consumer, rebalance, tuning]
description: 배치 처리가 max.poll.interval.ms를 넘겨 컨슈머가 추방되고 그룹 전체가 반복 정지하는 리밸런스 폭풍의 구조와 설정 처방을 정리했다.
---

Kafka 컨슈머 그룹이 주기적으로 통째로 멈추는 문제의 전형적인 원인을 정리했습니다. 리밸런스가 리밸런스를 부르는 악순환입니다.

## 추방 → 리밸런스 → 또 추방

컨슈머는 `poll()`을 주기적으로 호출해야 "살아있음"을 증명합니다. 한 번의 poll로 가져온 배치 처리가 `max.poll.interval.ms`(기본 5분)를 넘기면, 브로커는 그 컨슈머를 그룹에서 추방하고 리밸런스를 시작합니다. 문제는 여기서부터입니다.

- 리밸런스 동안 그룹 전체의 소비가 멈춘다 (eager 프로토콜 기준).
- 멈춘 사이 랙이 쌓이고, 재개 후 더 큰 배치를 가져와 처리가 더 길어진다.
- 다시 interval 초과 → 추방 → 리밸런스. 폭풍의 완성입니다.

## 처방 두 줄

**1. 한 번에 덜 가져온다.**

```properties
max.poll.records=50          # 기본 500 → 처리 시간이 interval 안에 들어오도록
max.poll.interval.ms=300000  # 처리 시간 실측 후 여유 있게
```

배치당 처리 시간 = records × 건당 처리 시간이 interval을 확실히 밑돌게 맞춥니다. 건당 처리가 무겁다면 records를 줄이는 쪽이 안전합니다.

**2. cooperative-sticky assignor.**

```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

eager 방식은 리밸런스 때 전원이 파티션을 내려놓고 다시 받지만, cooperative는 이동이 필요한 파티션만 옮깁니다. 리밸런스가 나더라도 "전원 정지"가 사라집니다.

## 정리

- 컨슈머 반복 정지는 대부분 처리 시간과 max.poll.interval의 충돌이다.
- max.poll.records를 실측 기반으로 줄이는 게 첫 번째 조치.
- cooperative-sticky로 리밸런스 비용 자체를 낮추면 폭풍이 구조적으로 사라진다.

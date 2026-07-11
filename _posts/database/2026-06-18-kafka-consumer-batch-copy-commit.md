---
title: Kafka 컨슈머가 DB CPU를 100%로 만들 때, 단건 커밋을 의심하자
date: 2026-06-18 19:26:00 +09:00
categories: [Database]
tags: [kafka, aurora, postgresql, batch, throughput]
description: 적체된 Kafka 메시지를 소진하는 동안 DB CPU가 폭주하는 원인이 메시지별 단건 INSERT·commit이라는 점과, 배치 COPY 적재로 해결하는 방법을 정리했다.
---

Kafka 적체가 풀리는 순간 Aurora CPU가 100%에 박히는 사례를 공부했습니다. 처음엔 트래픽 총량 문제라고 생각했는데, 진짜 원인은 쓰기 "방식"이었습니다.

## 단건 commit이 왜 치명적인가

컨슈머가 메시지 하나 받을 때마다 INSERT 한 번 + commit 한 번을 하면, DB 입장에서는 매 건마다 WAL flush(fsync)와 트랜잭션 오버헤드를 치릅니다. 초당 수천 메시지가 밀려오면 실제 데이터 양은 얼마 안 되는데도 커밋 비용만으로 CPU와 I/O가 포화됩니다.

## 배치로 묶으면 같은 데이터가 가벼워진다

해결은 두 단계입니다.

**1. N행을 모아 한 번에 적재.** PostgreSQL 계열이라면 다중 VALUES INSERT보다 `COPY`가 훨씬 빠릅니다.

```python
# 1,000건 모아서 COPY 한 번 + commit 한 번
buf = io.StringIO()
for msg in batch:  # poll()로 받은 배치
    buf.write(to_tsv_row(msg))
buf.seek(0)
cur.copy_expert("COPY events FROM STDIN", buf)
conn.commit()
consumer.commit()  # DB 커밋 성공 후 오프셋 커밋
```

**2. 커밋 단위 정렬.** DB commit 성공 후에 Kafka 오프셋을 커밋해야 크래시 시 중복은 있어도 유실은 없습니다(at-least-once). 중복은 PK나 unique 제약 + `ON CONFLICT DO NOTHING`으로 흡수합니다.

이 전환만으로 CPU 100%가 25% 수준으로 내려가고 쓰기 처리량이 자릿수로 뛰는 걸 확인할 수 있습니다.

## 정리

- 스트림 소비의 병목은 대체로 데이터 양이 아니라 트랜잭션 횟수다.
- 모아서 COPY + 배치당 commit 1회가 기본기.
- 오프셋 커밋은 DB 커밋 뒤에, 중복은 멱등 적재로 처리한다.

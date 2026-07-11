---
title: DB가 점점 느려질 때 의심할 것, idle in transaction
date: 2026-06-26 19:21:00 +09:00
categories: [Database]
tags: [postgresql, vacuum, transaction, bloat]
description: BEGIN 후 방치된 트랜잭션이 오래된 스냅샷과 락을 붙잡아 autovacuum을 막고 테이블을 부풀리는 문제와 대응을 정리했다.
---

특별히 트래픽이 늘지 않았는데 DB가 서서히 느려지는 패턴을 공부하다가 `idle in transaction`이라는 상태를 제대로 이해하게 됐습니다.

## 무엇이 문제인가

`BEGIN`만 열어두고 커밋도 롤백도 하지 않은 세션은 `pg_stat_activity`에 `idle in transaction`으로 표시됩니다. 이 세션이 위험한 이유는 두 가지입니다.

- 트랜잭션 시작 시점의 스냅샷을 계속 유지하므로, autovacuum이 그 이후의 죽은 튜플(dead tuple)을 치우지 못한다. 테이블과 인덱스가 계속 부푼다(bloat).
- 잡아둔 락이 있으면 다른 DDL이나 쿼리가 그 뒤에 줄을 선다.

주범은 보통 애플리케이션 코드입니다. 트랜잭션을 열어놓고 외부 API를 호출하거나, 예외 경로에서 커밋/롤백을 빼먹는 경우입니다.

## 확인과 대응

```sql
-- 방치된 트랜잭션 찾기
SELECT pid, state, xact_start, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

```sql
-- 일정 시간 이상 방치되면 서버가 강제 종료
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';
SELECT pg_reload_conf();
```

근본 대책은 애플리케이션에서 트랜잭션 범위를 최소화하는 것입니다. 트랜잭션 안에서는 DB 작업만 하고, 외부 호출·파일 처리는 트랜잭션 밖으로 빼는 원칙을 지키는 게 핵심입니다.

## 정리

- 서서히 느려지는 DB는 bloat를 의심하고, bloat는 idle in transaction을 의심한다.
- `idle_in_transaction_session_timeout`은 안전망, 트랜잭션 범위 최소화가 본질적 해법.
- 모니터링에 idle in transaction 세션 수와 최장 트랜잭션 나이를 넣어두면 조기에 잡을 수 있다.

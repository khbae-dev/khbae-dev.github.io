---
title: PostgreSQL 커넥션 풀이 꽉 찼을 때 관리자까지 잠기는 문제
date: 2026-06-04 19:34:00 +09:00
categories: [Database]
tags: [postgresql, database, connection-pool, incident]
description: 커넥션 폭주 장애 때 superuser조차 접속하지 못하는 상황과, superuser_reserved_connections로 비상용 슬롯을 예약해 두는 방법을 정리했다.
---

커넥션 관련 장애를 공부하다가 무서운 시나리오를 하나 알게 됐습니다. 애플리케이션이 커넥션을 폭주시켜 `max_connections`가 가득 차면, 문제 세션을 정리하러 들어가야 할 관리자조차 접속할 수 없다는 점입니다. DB는 죽어가는데 죽일 방법이 없는 상태가 됩니다.

## 왜 관리자까지 못 들어가나

PostgreSQL은 기본적으로 접속 슬롯을 선착순으로 소비합니다. 폭주한 애플리케이션 세션이 슬롯을 전부 점유하면 `psql`로 붙으려는 superuser도 똑같이 `too many clients already` 에러를 받습니다.

이때를 위한 설정이 `superuser_reserved_connections`입니다. 전체 슬롯 중 일부를 superuser 전용으로 떼어놓는 설정으로, 기본값은 3입니다.

```sql
-- postgresql.conf
max_connections = 200
superuser_reserved_connections = 5
```

이렇게 설정하면 일반 유저는 195개까지만 사용할 수 있고, 나머지 5개는 superuser만 사용할 수 있습니다. 장애 상황에서 예약 슬롯으로 접속한 뒤 폭주 세션을 정리하면 됩니다.

```sql
-- 폭주 세션 확인 후 정리
SELECT pid, usename, state, query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'app_user' AND state = 'active';
```

## 정리

- 커넥션이 가득 차면 관리자 접속도 막힌다. 복구 진입로부터 확보해야 한다.
- `superuser_reserved_connections`로 비상용 슬롯을 예약해 두면 장애 시 바로 들어가 정리할 수 있다. 복구 시간이 분 단위에서 초 단위로 줄어든다.
- 애초에 애플리케이션 쪽 커넥션 풀 상한을 DB `max_connections`보다 확실히 낮게 잡는 것도 같이 챙겨야 한다.

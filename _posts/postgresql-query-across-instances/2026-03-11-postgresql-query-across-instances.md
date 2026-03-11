---
title: PostgreSQL 서로 다른 인스턴스 조회 결과를 같이 보는 방법
date: 2026-03-11 14:30:00 +09:00
tags: [postgresql, database, dblink, postgres-fdw, dbeaver]
description: PostgreSQL에서 서로 다른 두 인스턴스의 조회 결과를 한 SQL처럼 같이 보고 싶을 때 dblink, postgres_fdw, DBeaver 비교, 외부 스크립트 병합 방식을 실무 기준으로 정리했다.
---

PostgreSQL을 쓰다 보면 "서로 다른 두 인스턴스의 테이블을 한 번에 같이 조회할 수 없을까?"라는 요구가 자주 나옵니다.
예를 들어 운영 DB와 별도 인스턴스의 분석 DB를 비교하거나, 마이그레이션 전후 데이터를 대조할 때 특히 그렇습니다.

결론부터 말하면 PostgreSQL은 기본 상태에서 서로 다른 인스턴스를 한 SQL에서 바로 조회하지 못합니다.
대신 보통 아래 네 가지 방식 중 하나를 사용합니다.

- `dblink`: 한두 번 빠르게 조회할 때
- `postgres_fdw`: 원격 테이블처럼 붙여서 계속 사용할 때
- DBeaver 결과 비교: SQL join 없이 화면에서만 비교할 때
- 외부 스크립트 병합: DB 확장 설치나 네트워크 제약이 있을 때

이번 글은 각 방법의 차이와 언제 무엇을 선택하면 되는지를 실무 기준으로 정리한 노트입니다.

## 1. 왜 기본 SQL만으로는 바로 안 되나

`SELECT * FROM instance_a.public.table1 JOIN instance_b.public.table2 ...` 같은 문법은 PostgreSQL 기본 기능에는 없습니다.
현재 접속한 세션은 하나의 PostgreSQL 인스턴스 안에서만 객체를 인식하기 때문입니다.

즉 다른 인스턴스 데이터까지 같이 보려면 다음 중 하나가 필요합니다.

- 현재 인스턴스에서 원격 인스턴스에 접속하는 기능
- 클라이언트 도구에서 결과를 나란히 비교하는 기능
- 애플리케이션 코드에서 각 결과를 받아 직접 병합하는 로직

## 2. 가장 일반적인 방법: `dblink`

`dblink`는 현재 접속한 DB에서 다른 PostgreSQL 인스턴스에 접속해 쿼리 결과를 가져오는 방식입니다.
일회성 조회나 임시 분석에는 가장 빠르게 적용할 수 있습니다.

먼저 확장을 사용할 수 있어야 합니다.

```sql
CREATE EXTENSION IF NOT EXISTS dblink;
```

### 단건 조회 예시

```sql
SELECT *
FROM dblink(
    'host=10.0.0.2 port=5432 dbname=db2 user=myuser password=mypassword',
    'SELECT id, name, created_at FROM public.sample_table'
) AS t(id int, name text, created_at timestamp);
```

이 쿼리는 현재 인스턴스에서 원격 인스턴스 `db2`의 `sample_table` 결과를 읽어옵니다.
중요한 점은 `AS t(...)` 부분에서 반환 컬럼 구조를 직접 선언해야 한다는 점입니다.

### 로컬 테이블과 join 예시

```sql
SELECT
    a.id,
    a.name AS local_name,
    b.name AS remote_name
FROM local_table a
LEFT JOIN dblink(
    'host=10.0.0.2 port=5432 dbname=db2 user=myuser password=mypassword',
    'SELECT id, name FROM public.remote_table'
) AS b(id int, name text)
ON a.id = b.id;
```

이런 식으로 현재 DB의 `local_table`과 원격 인스턴스의 `remote_table`을 한 결과셋으로 볼 수 있습니다.

### `dblink`가 잘 맞는 경우

- 한두 번 임시로 조회하고 끝나는 경우
- 빠르게 비교 쿼리를 만들어야 하는 경우
- 정식 외부 테이블 구성을 하기 전 먼저 검증해보고 싶은 경우

### `dblink`의 한계

- 컬럼 정의를 매번 직접 맞춰줘야 한다
- 재사용성과 관리성이 떨어진다
- 쿼리가 길어지고 복잡해지기 쉽다
- 비밀번호를 문자열로 직접 넣으면 보안상 좋지 않다

즉 `dblink`는 빠른 실험에는 좋지만, 운영성 있게 오래 가져갈 방식으로는 아쉬운 편입니다.

## 3. 지속적으로 쓸 거라면: `postgres_fdw`

`postgres_fdw`는 원격 PostgreSQL 테이블을 로컬의 외부 테이블처럼 다루게 해주는 기능입니다.
지속적으로 조회하거나, 여러 쿼리에서 반복해 써야 한다면 보통 `dblink`보다 이쪽이 낫습니다.

기본 설정 흐름은 아래와 같습니다.

```sql
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER remote_db
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host '10.0.0.2', port '5432', dbname 'db2');

CREATE USER MAPPING FOR current_user
SERVER remote_db
OPTIONS (user 'myuser', password 'mypassword');

CREATE FOREIGN TABLE remote_table (
    id int,
    name text,
    created_at timestamp
)
SERVER remote_db
OPTIONS (schema_name 'public', table_name 'sample_table');
```

설정이 끝나면 이후부터는 원격 테이블을 거의 로컬 테이블처럼 조회할 수 있습니다.

```sql
SELECT
    l.id,
    l.name AS local_name,
    r.name AS remote_name
FROM local_table l
LEFT JOIN remote_table r
ON l.id = r.id;
```

### `postgres_fdw`가 좋은 이유

- 원격 테이블을 재사용 가능한 객체로 관리할 수 있다
- 여러 쿼리에서 같은 방식으로 접근할 수 있다
- 쿼리 가독성이 `dblink`보다 낫다
- 운영 환경에서 반복 조회용으로 더 안정적이다

### `postgres_fdw`가 잘 맞는 경우

- 두 인스턴스 데이터를 자주 join해야 하는 경우
- 리포트나 운영 조회에서 반복적으로 사용할 경우
- 특정 원격 테이블을 로컬 테이블처럼 다루고 싶은 경우

### 참고할 점

- 확장 생성 권한이 필요할 수 있다
- DB 간 네트워크 연결이 가능해야 한다
- 원격 스키마 변경 시 외부 테이블 정의도 맞춰줘야 한다

실무에서는 "한 번 비교해보는 수준"을 넘어서면 `postgres_fdw` 쪽으로 가는 경우가 많습니다.

## 4. SQL로 합칠 필요가 없다면 DBeaver만으로도 충분하다

모든 상황에서 DB 레벨 join이 필요한 것은 아닙니다.
그냥 같은 쿼리를 두 인스턴스에 실행해서 결과를 나란히 비교하면 되는 경우도 많습니다.

이럴 때는 DBeaver에서 아래처럼 접근하면 됩니다.

1. 각 PostgreSQL 인스턴스에 연결을 만든다
2. 각각 SQL Editor를 연다
3. 같은 쿼리를 각 인스턴스에서 실행한다
4. 결과 탭을 좌우로 배치해 비교한다

이 방식의 장점은 가장 단순하다는 점입니다.
반면 SQL 하나로 합쳐진 결과셋을 만들 수는 없으므로, join이나 diff 계산이 필요한 상황에는 한계가 있습니다.

## 5. 권한이나 네트워크 제약이 있으면 외부 스크립트에서 병합한다

현실적으로는 아래 같은 이유로 `dblink`나 `postgres_fdw`를 못 쓰는 경우도 있습니다.

- 확장 설치 권한이 없음
- 보안 정책상 DB-to-DB 연결이 막혀 있음
- 중간 가공 로직이 매우 복잡함

이럴 때는 파이썬, 자바, Node.js 같은 외부 프로그램에서 해결합니다.

흐름은 단순합니다.

1. 인스턴스 A에서 조회한다
2. 인스턴스 B에서 조회한다
3. 애플리케이션 메모리에서 merge 또는 join 한다

이 방식은 유연성이 높고 후처리가 쉽지만, SQL 하나로 끝나는 단순함은 포기해야 합니다.

## 6. 무엇을 선택하면 되나

정리하면 기준은 꽤 명확합니다.

### `dblink`를 선택할 때

- 한두 번 빠르게 비교 조회하고 끝날 때
- 운영 구조를 크게 건드리지 않고 테스트성으로 확인할 때

### `postgres_fdw`를 선택할 때

- 두 인스턴스 데이터를 계속 join해서 써야 할 때
- 외부 테이블처럼 운영성 있게 관리해야 할 때

### DBeaver 비교를 선택할 때

- SQL 레벨 병합은 필요 없고 결과만 눈으로 비교하면 될 때

### 외부 스크립트를 선택할 때

- DB 확장 권한이나 네트워크 정책 때문에 DB 간 직접 연결이 어려울 때
- 복잡한 후처리와 비즈니스 로직이 같이 들어갈 때

## 7. 실무에서 같이 보는 체크포인트

방법을 정할 때는 기능만 보지 말고 아래도 같이 확인하는 편이 좋습니다.

- 현재 DB 계정에 `CREATE EXTENSION` 권한이 있는가
- 원격 인스턴스에 네트워크 접근이 가능한가
- 비밀번호를 쿼리 문자열에 직접 넣어도 되는 환경인가
- 일회성 조회인지, 반복 운영 조회인지
- 결과를 "같이 보기"만 하면 되는지, 실제 join 결과가 필요한지

이 다섯 가지를 먼저 보면 대체로 방향이 바로 정해집니다.

## 8. 정리

PostgreSQL은 기본 상태에서 서로 다른 두 인스턴스를 한 SQL로 바로 조회하지 못합니다.
그래서 보통 아래 중 하나를 선택합니다.

- 빠른 임시 조회: `dblink`
- 반복 사용과 운영성: `postgres_fdw`
- 화면 비교만 필요: DBeaver
- 권한/정책 제약 대응: 외부 스크립트 병합

질문이 "한 스크립트에서 같이 보고 싶다"에 가깝다면, 대부분의 출발점은 `dblink`입니다.
반대로 "계속 join해서 써야 한다"면 처음부터 `postgres_fdw`로 가는 편이 더 낫습니다.

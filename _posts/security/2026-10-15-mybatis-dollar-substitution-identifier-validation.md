---
title: 바인딩할 수 없는 SQL 조각, MyBatis ${} 치환을 안전하게 다루기
date: 2026-10-15 19:18:00 +09:00
categories: [Security]
tags: [security, sql-injection, mybatis, postgresql]
description: MyBatis의 ${} 치환 중 값은 #{} 바인딩으로 바꾸고, 테이블명·컬럼 별칭처럼 바인딩할 수 없는 식별자는 서비스 계층에서 허용 형식만 통과시키는 방식으로 정리했다.
---

[파라미터 바인딩 글](/posts/sql-injection-parameter-binding/)에서 "값은 언제나 바인딩으로"라고 정리했습니다. 그런데 MyBatis 매퍼를 점검하다 보면 `${}`를 쓸 수밖에 없는 자리가 남습니다. 그 자리를 어떻게 다뤘는지 정리합니다.

## `#{}`와 `${}`의 차이

```xml
WHERE id = #{id}     <!-- PreparedStatement 의 ? 로 바인딩된다 -->
WHERE id = ${id}     <!-- 문자열 그대로 SQL에 붙는다 -->
```

`${}`는 SQL 문자열 치환이라 입력이 SQL 코드가 될 수 있습니다. 점검은 매퍼 전체에서 `${`를 찾아 하나씩 "이게 값인가, 식별자인가"로 나누는 것부터 시작했습니다.

## 1단계: 값인데 `${}`인 것은 바인딩으로

가장 많았던 것은 사실 값인데 습관적으로 `${}`를 쓴 경우였습니다. 예를 들어 JSONB 키 조회입니다.

```xml
<!-- 전: 작은따옴표로 감쌌지만 입력에 ' 가 있으면 탈출한다 -->
WHEN jsonb_exists(payload::jsonb, '${key}')
THEN (payload ->> '${key}')::float8

<!-- 후: 키도 결국 값이다 -->
WHEN jsonb_exists(payload::jsonb, #{key})
THEN (payload ->> #{key})::float8
```

`->>` 연산자의 오른쪽, `jsonb_exists`의 인자, `WHERE k = '...'` 같은 자리는 모두 값이므로 바인딩이 됩니다. 따옴표로 감싸 놓은 `'${...}'`는 안전해 보이지만 입력에 작은따옴표 하나만 있으면 문자열 밖으로 나옵니다.

## 2단계: 정말 식별자인 것은 허용 형식만

테이블명과 컬럼 별칭은 JDBC 바인딩이 안 됩니다. 이 자리는 **매퍼가 아니라 서비스 계층에서** 검증하고, 검증을 통과한 값만 매퍼로 넘깁니다.

```java
public final class SqlIdentifiers {
    private static final Pattern YEARLY_TABLE = Pattern.compile("events_[0-9]{4}");

    // 테이블명: 허용 목록이 있으면 목록, 규칙이 있으면 정규식으로 전체 일치
    public static String requireYearlyTable(String name) {
        if (name == null || !YEARLY_TABLE.matcher(name).matches()) {
            throw new IllegalArgumentException("허용되지 않은 테이블명입니다.");
        }
        return name;
    }

    // "${alias}" 처럼 큰따옴표 안에 들어가는 식별자:
    // 큰따옴표·NUL·개행이 있으면 인용 식별자 밖으로 탈출할 수 있으므로 거부
    public static void requireQuotedIdentifiers(Collection<String> ids) {
        for (String id : ids) {
            if (id == null || id.isEmpty() || id.chars().anyMatch(ch -> ch == '"' || ch == 0 || ch == '\r' || ch == '\n')) {
                throw new IllegalArgumentException("허용되지 않은 식별자입니다.");
            }
        }
    }
}
```

정규식은 `find()`가 아니라 `matches()`로 전체 일치를 봐야 합니다. `find()`면 `events_2026; DROP ...`도 통과합니다.

## 3단계: 계약 테스트로 고정

새 `${}`가 슬그머니 생기지 않도록, 매퍼 XML을 읽어 `${` 사용처를 허용 목록과 비교하는 테스트를 두었습니다. 허용 목록에 있는 자리는 "어떤 검증을 거친 값이 들어오는지"를 주석으로 같이 남깁니다. 새 `${}`를 추가하려면 이 목록을 고쳐야 하므로 리뷰에서 자연스럽게 드러납니다.

## 정리

- `${}`는 하나씩 "값인가, 식별자인가"로 분류한다. 값이면 예외 없이 `#{}`로 바꾼다.
- `'${...}'`처럼 따옴표로 감싸도 안전하지 않다.
- 식별자는 서비스 계층에서 허용 목록이나 전체 일치 정규식으로 검증한 뒤에만 매퍼로 넘긴다.
- 매퍼의 `${}` 사용처를 테스트로 고정해 새로 생기는 자리를 리뷰에서 걸러낸다.

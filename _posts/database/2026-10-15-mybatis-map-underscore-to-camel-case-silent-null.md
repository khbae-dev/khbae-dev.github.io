---
title: mapUnderscoreToCamelCase를 켜는 순간 조용히 null이 되는 필드들
date: 2026-10-15 19:20:00 +09:00
categories: [Database]
tags: [mybatis, java, json, api-contract, refactoring]
description: MyBatis의 mapUnderscoreToCamelCase를 켜면 snake_case 필드를 쓰던 클래스가 예외 없이 null로 채워지는 이유와, 필드 정규화·설정 활성화·JSON 계약 고정을 한 번에 묶어 안전하게 옮긴 방법을 정리했다.
---

오래된 Spring + MyBatis 프로젝트에는 DB 컬럼명을 그대로 옮긴 `item_id` 같은 snake_case 자바 필드가 흔합니다. 이를 camelCase로 정리하면서 `mapUnderscoreToCamelCase`를 켰는데, 이 설정은 잘못 켜면 **오류 없이** 데이터를 잃습니다.

## 왜 조용히 null이 되는가

`mapUnderscoreToCamelCase=true`이면 MyBatis는 결과 컬럼 `item_id`를 프로퍼티 `itemId`에 매핑하려고 합니다. 그런데 클래스 필드가 아직 `item_id`라면 `itemId` 프로퍼티를 찾지 못합니다. MyBatis는 이때 예외를 던지지 않고 그 컬럼을 건너뜁니다. 결과는 필드가 전부 null인 객체입니다.

```xml
<settings>
  <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```

```java
public class Item {
    private String item_id;   // 설정을 켜면 이 필드는 더 이상 채워지지 않는다
}
```

## 함께 깨지는 곳: 매퍼의 파라미터 표현식

조회 결과만 문제가 아닙니다. 매퍼 XML의 `#{item.item_id}`도 Bean 프로퍼티 경로라서, 필드명을 바꾸면 함께 깨집니다.

```xml
<foreach collection="list" item="item" separator=",">
  (#{item.itemId}, #{item.name})   <!-- 필드명을 바꾸면 여기도 바꿔야 한다 -->
</foreach>
```

단, `collection`이 `HashMap` 목록인 곳은 키 이름이 그대로라 대상이 아닙니다. 어느 쪽인지 매퍼마다 확인해야 합니다.

## 나가는 JSON 계약은 그대로 두기

필드명이 바뀌면 Jackson이 내보내는 JSON 키도 `item_id`에서 `itemId`로 바뀝니다. 프런트엔드 입장에서는 API가 깨진 것입니다. 그래서 키는 `@JsonProperty`로 기존 값에 고정했습니다.

```java
public class Item {
    @JsonProperty("item_id")
    private String itemId;
}
```

## 순서가 아니라 묶음이 답이다

"필드 먼저 바꾸고 설정은 나중에", 혹은 그 반대로 나누면 그 사이 상태는 반드시 깨져 있습니다. 그래서 세 가지를 한 커밋으로 묶었습니다.

1. 매퍼가 매핑하는 클래스의 필드를 camelCase로 바꾸고 `@JsonProperty`로 JSON 키를 고정한다.
2. 매퍼 XML의 `#{item.xxx}` 경로를 함께 바꾼다.
3. `mapUnderscoreToCamelCase`를 켠다.

`callSettersOnNulls`처럼 비슷해 보이는 설정은 일부러 건드리지 않았습니다. 켜면 `HashMap` 결과에 null 필드가 새로 생겨 응답 모양이 바뀌기 때문입니다. 한 번에 하나의 계약만 바꿉니다.

## 검증

- **JSON 필드명 계약 테스트**: 각 클래스를 직렬화했을 때 나오는 키 목록을 `containsExactly`로 고정한다. 키가 하나 늘거나 바뀌어도 실패한다.
- **매핑 테스트**: 실제 `mybatis-config.xml`을 H2에 물려 snake_case 컬럼이 camelCase 필드에 채워지는지 확인한다. setter 없이 필드에 직접 주입되는 클래스와 `HashMap` 결과도 포함한다.

## 정리

- `mapUnderscoreToCamelCase`는 매핑 실패를 예외가 아니라 null로 드러낸다.
- 필드명 변경, 매퍼 파라미터 경로, 설정 활성화는 한 커밋으로 묶는다.
- 필드명을 바꿔도 JSON 키는 `@JsonProperty`로 고정하고, 키 목록을 테스트로 못 박는다.

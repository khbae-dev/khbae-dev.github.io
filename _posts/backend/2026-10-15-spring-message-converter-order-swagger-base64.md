---
title: Swagger UI가 문서를 못 그리는데 드롭다운은 멀쩡한 이유, 메시지 컨버터 순서
date: 2026-10-15 19:14:00 +09:00
categories: [Backend]
tags: [spring, http-message-converter, jackson, springdoc, swagger]
description: configureMessageConverters로 컨버터 목록을 직접 구성하면서 Jackson이 ByteArray 컨버터보다 앞에 놓여 byte[] 응답이 Base64 문자열로 나간 문제와 그 진단 과정을 정리했다.
---

springdoc으로 옮긴 직후 Swagger UI가 열리기는 하는데 API 목록이 비어 있었습니다. 상단의 그룹 드롭다운은 정상이었습니다. 원인은 HTTP 메시지 컨버터의 순서 한 줄이었습니다.

## 증상

`/v3/api-docs/{group}`을 직접 호출해 보니 JSON 문서가 아니라 따옴표로 감싼 긴 문자열 하나가 왔습니다.

```text
"eyJvcGVuYXBpIjoiMy4wLjEiLCJpbmZvIjp7InRpdGxlIjoi..."
```

디코딩하면 정상적인 OpenAPI 문서였습니다. **문서가 Base64로 한 번 더 감싸져 나가고 있었던 것**입니다.

## 원인: 먼저 `canWrite`인 컨버터가 응답을 가져간다

Spring MVC는 응답을 쓸 때 등록된 `HttpMessageConverter`를 순서대로 보면서 `canWrite(반환타입, 미디어타입)`이 처음 true인 컨버터를 씁니다.

이 프로젝트는 `WebMvcConfigurer.configureMessageConverters()`로 기본 목록을 통째로 대체하고 있었고, 그 목록에서 Jackson이 `ByteArrayHttpMessageConverter`보다 앞에 있었습니다.

```java
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    converters.add(new MappingJackson2HttpMessageConverter(objectMapper)); // 먼저
    converters.add(new ByteArrayHttpMessageConverter());                    // 나중
    converters.add(new StringHttpMessageConverter(UTF_8));
}
```

Jackson은 `byte[]`도 직렬화할 수 있습니다. JSON 규칙대로 Base64 문자열로 바꿔서요. 그래서 `canWrite(byte[].class, application/json)`이 true이고, springdoc이 이미 직렬화해서 `byte[]`로 돌려준 문서를 Jackson이 가로채 다시 직렬화했습니다.

드롭다운이 멀쩡했던 이유도 같은 원리로 설명됩니다. 드롭다운을 채우는 `/v3/api-docs/swagger-config`는 `Map`을 반환하므로 Jackson이 가져가는 게 맞습니다. 예전 springfox는 전용 직렬화기를 가진 객체를 반환해서 같은 순서에서도 문제가 드러나지 않았습니다.

## 고친 것

`ByteArrayHttpMessageConverter`를 Jackson 앞으로 옮겼습니다. 동작 변경은 이 한 줄입니다.

```java
converters.add(new ByteArrayHttpMessageConverter());                    // 먼저
converters.add(new MappingJackson2HttpMessageConverter(objectMapper));
converters.add(new StringHttpMessageConverter(UTF_8));
```

`StringHttpMessageConverter`는 일부러 Jackson 뒤에 그대로 뒀습니다. 기본 순서로 완전히 되돌리면 `ResponseEntity<String>`을 반환하던 엔드포인트의 응답이 `"..."`(JSON 문자열)에서 따옴표 없는 본문으로 바뀝니다. 그건 이미 나가 있는 API 계약을 바꾸는 일이라 이번 수정 범위에서 뺐습니다.

## 순서를 테스트로 고정하기

순서 문제는 누가 목록을 정리하다 한 줄만 옮겨도 다시 생깁니다. 그래서 실제 설정 클래스가 만드는 목록을 꺼내 순서를 검증하는 테스트를 붙였습니다.

```java
@Test
void byteArrayConverterComesBeforeJackson() {
    List<HttpMessageConverter<?>> converters = new ArrayList<>();
    config.configureMessageConverters(converters);
    int bytes = indexOf(converters, ByteArrayHttpMessageConverter.class);
    int jackson = indexOf(converters, MappingJackson2HttpMessageConverter.class);
    assertThat(bytes).isLessThan(jackson);
}
```

## 정리

- 메시지 컨버터는 "가장 잘 맞는 것"이 아니라 "먼저 `canWrite`인 것"이 이긴다.
- `configureMessageConverters`로 목록을 직접 구성하면 기본 순서가 주던 안전장치도 같이 사라진다. 필요한 것만 덧붙일 거라면 `extendMessageConverters`가 낫다.
- 순서를 고칠 때는 다른 반환 타입의 응답 형식이 바뀌지 않는지 함께 확인하고, 순서를 테스트로 고정한다.

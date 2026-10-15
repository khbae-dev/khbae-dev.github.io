---
title: springfox에서 springdoc-openapi로, pom에서 지워도 남아 있던 의존성
date: 2026-10-15 19:12:00 +09:00
categories: [Backend]
tags: [spring-boot, swagger, openapi, springdoc, maven]
description: Spring Boot 2.6 이상에서 기동이 실패하는 springfox를 springdoc-openapi로 옮기면서, 전이 의존성으로 남아 있던 springfox를 exclusion으로 걷어내고 재발 방지 테스트를 붙인 과정을 정리했다.
---

Spring Boot 업그레이드를 준비하다 보니 가장 먼저 막는 것이 API 문서 라이브러리였습니다. springfox를 springdoc-openapi로 옮긴 과정을 정리합니다.

## 왜 springfox부터인가

Spring Boot 2.6부터 MVC의 기본 경로 매칭이 `AntPathMatcher`에서 `PathPatternParser`로 바뀌었습니다. springfox는 이를 지원하지 않아 Boot 2.6 이상에서 기동 자체가 실패합니다. 사실상 유지보수도 멈춘 상태입니다. Boot 업그레이드와 Swagger 문제를 동시에 디버깅하지 않으려면 문서 라이브러리를 먼저 바꿔 두는 편이 안전합니다.

게다가 들여다보니 `springfox-swagger2 3.0.0` 백엔드에 `springfox-swagger-ui 2.9.2` UI를 섞어 쓰고 있었습니다. 세대가 다른 두 버전이 우연히 맞물려 돌던 것입니다.

## 바꾼 것

| 항목 | springfox | springdoc |
|---|---|---|
| 의존성 | `springfox-swagger2`, `springfox-swagger-ui` | `springdoc-openapi-ui` |
| 설정 | `@EnableSwagger2` + `Docket` | `OpenAPI` + `GroupedOpenApi` 빈 |
| 애너테이션 | `@ApiOperation` | `@Operation` |
| | `@ApiModel`, `@ApiModelProperty` | `@Schema` |
| | `@ApiIgnore` | `@Hidden` |
| 문서 경로 | `/v2/api-docs` | `/v3/api-docs` |
| UI 경로 | `/swagger-ui.html` | `/swagger-ui/**` |

보안 설정의 허용 경로도 새 경로로 바꾸고, springfox 때문에 직접 등록하던 webjar 리소스 핸들러는 springdoc이 알아서 등록하므로 걷어냈습니다.

## 함정: pom에서 지워도 클래스패스에 남는다

선언 두 줄을 지웠는데도 `io.swagger.annotations`가 계속 컴파일됐습니다. 사내 공통 라이브러리가 `springfox-swagger2`를 **전이 의존성**으로 끌고 오고 있었기 때문입니다.

```bash
mvn dependency:tree -Dincludes=io.springfox
```

그래서 그 라이브러리 쪽에 exclusion을 걸었습니다.

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>common-platform</artifactId>
  <exclusions>
    <exclusion>
      <groupId>io.springfox</groupId>
      <artifactId>*</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

남의 라이브러리에서 의존성을 빼는 건 위험하니 근거를 쟀습니다. jar 안의 클래스를 전수 조사해 springfox 참조가 0건인 것을 확인했고, 추출이 제대로 됐는지 보려고 대조군으로 Spring 참조가 잡히는지도 같이 봤습니다. Swagger를 언급하는 클래스는 애너테이션만 쓰고 있었는데, 애너테이션은 런타임에 클래스가 없으면 JVM이 무시하므로 안전합니다.

## 재발 방지 테스트

옛 애너테이션은 남아 있어도 컴파일이 되고 기동도 됩니다. 대신 **그 엔드포인트 설명이 문서에서 조용히 사라집니다.** 그래서 테스트로 막았습니다.

- 소스에 `io.swagger.annotations` import가 없다
- pom에 springfox 선언이 없고, exclusion이 유지된다
- 보안 설정에 새 문서 경로가 열려 있다
- 이 검사 자체가 실제로 파일을 읽고 있다 (빈 목록을 검사하고 통과하는 것을 막는다)

마지막 항목이 은근히 중요합니다. 경로가 바뀌어 검사 대상이 0개가 되면 테스트는 항상 통과합니다.

## 정리

- springfox는 Boot 2.6+에서 깨진다. Boot 업그레이드 전에 문서 라이브러리부터 바꿔 원인을 분리한다.
- pom 선언 제거만으로는 부족하다. `dependency:tree`로 전이 의존성을 확인하고 exclusion을 건다.
- 조용히 망가지는 종류의 변경에는 "검사가 실제로 무언가를 검사하는지"까지 포함한 재발 방지 테스트를 둔다.

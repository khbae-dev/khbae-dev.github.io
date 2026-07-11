---
title: XML 업로드 한 건에 서버 파일이 유출되는 XXE 인젝션
date: 2026-06-30 19:25:00 +09:00
categories: [Security]
tags: [security, xxe, xml, injection]
description: XML 파서가 외부 엔티티를 기본으로 해석하면서 생기는 XXE 취약점의 원리와, 외부 DTD·엔티티 비활성화 방어를 정리했다.
---

XML을 받아 처리하는 기능에 숨은 고전적 취약점, XXE(XML External Entity)를 공부했습니다. 파일 하나 업로드로 서버 내부 파일이 읽히는 원리가 인상 깊었습니다.

## 파서가 너무 친절해서 생기는 문제

XML 명세에는 외부 리소스를 참조하는 엔티티 기능이 있습니다. 많은 XML 파서가 이걸 기본으로 해석합니다. 공격자는 이 점을 이용합니다.

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<data>&xxe;</data>
```

파서가 `&xxe;`를 해석하며 `file:///etc/passwd`를 읽어 응답에 포함시키면, 서버 내부 파일이 그대로 유출됩니다. `file://`뿐 아니라 `http://internal-service/`로 내부망을 스캔하는 SSRF, 재귀 엔티티로 메모리를 터뜨리는 DoС(billion laughs)까지 파생됩니다. XML 기반 API, SAML, 문서 업로드(docx·svg 내부도 XML) 어디서나 나타날 수 있습니다.

## 방어: 외부 엔티티를 끈다

해결은 파서 설정 몇 줄입니다. "외부 리소스를 로드하지 않게" 만드는 겁니다.

```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
// 핵심: DOCTYPE 자체를 금지하면 XXE는 원천 차단
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
// 보조: 외부 엔티티·DTD 로딩 비활성화
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setXIncludeAware(false);
```

가장 강력한 건 `disallow-doctype-decl`입니다. DOCTYPE을 아예 금지하면 엔티티 정의가 불가능해져 XXE 계열이 통째로 막힙니다. 애플리케이션이 DTD를 정말 쓰지 않는다면 이게 최선입니다. 언어별로 설정 이름은 다르지만 원칙은 같습니다. libxml 계열은 `noent`를 켜지 않는 것, Python은 defusedxml 사용 등입니다.

## 정리

- XXE는 파서가 외부 엔티티를 "기본으로 해석"하는 친절함에서 비롯된다.
- 가능하면 DOCTYPE 선언 자체를 금지(disallow-doctype-decl)하는 게 가장 확실하다.
- XML을 다루는 모든 진입점(API, 업로드, SAML)에 같은 설정을 일관 적용한다.

---

*보안 취약점은 방어 목적으로만 다룹니다.*

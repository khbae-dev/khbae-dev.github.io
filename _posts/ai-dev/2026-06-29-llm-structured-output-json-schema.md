---
title: LLM이 준 JSON이 자꾸 파싱 실패할 때, structured output
date: 2026-06-29 19:22:00 +09:00
categories: [AI Dev]
tags: [llm, structured-output, json-schema, api]
description: 프롬프트로만 JSON 형식을 강제할 때 파싱이 깨지는 문제를, response_format의 json_schema와 스키마 검증·재시도로 안정화하는 방법을 정리했다.
---

LLM 응답을 `JSON.parse`에 넣었다가 터지는 문제를 근본적으로 없애는 방법을 공부했습니다. 프롬프트에만 의존하던 방식을 버리는 게 시작이었습니다.

## "프롬프트로만 강제"의 한계

"반드시 JSON으로만 답해"라고 아무리 강조해도 모델은 종종 마크다운 코드펜스, 설명 문장, 후행 쉼표, 잘린 응답을 섞어 냅니다. 이걸 정규식으로 걷어내고 파싱하는 땜빵 코드는 새 실패 케이스가 나올 때마다 깨집니다. 형식 보장을 모델의 "선의"에 맡기는 구조 자체가 문제입니다.

## 파서가 형식을 강제하게 만든다

요즘 API는 응답을 스키마에 맞게 제약하는 기능을 제공합니다. 모델의 디코딩 단계에서 스키마를 벗어나는 토큰을 아예 생성하지 못하게 하는 방식이라, 형식이 구조적으로 보장됩니다.

```python
response = client.chat.completions.create(
    model="...",
    messages=[...],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "extraction",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "title":    {"type": "string"},
                    "priority": {"type": "integer", "minimum": 1, "maximum": 5},
                    "tags":     {"type": "array", "items": {"type": "string"}},
                },
                "required": ["title", "priority", "tags"],
                "additionalProperties": False,
            },
        },
    },
)
```

여기에 애플리케이션 레벨 검증을 한 겹 더합니다. 받은 JSON을 Pydantic·zod 같은 스키마로 다시 검증하고, 실패하면(값 범위 위반 등) 에러 메시지를 붙여 1회 재시도합니다. 형식은 API가, 의미(값의 타당성)는 검증 라이브러리가 맡는 이중 구조입니다. 이러면 정규식 후처리가 통째로 사라지고 파싱 실패가 사실상 0이 됩니다.

## 정리

- JSON 형식을 프롬프트에만 맡기면 실패는 확률적으로 반드시 난다.
- response_format의 json_schema로 형식을 디코딩 단계에서 강제한다.
- API의 형식 보장 + 라이브러리의 값 검증 + 1회 재시도가 견고한 조합이다.

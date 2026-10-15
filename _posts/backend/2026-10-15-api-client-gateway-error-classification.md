---
title: 게이트웨이 502를 "로그인 필요"로 오진한 API 클라이언트, 오류 분기 순서
date: 2026-10-15 19:28:00 +09:00
categories: [Backend]
tags: [http, api-client, gateway, retry, python]
description: 응답이 JSON이 아니면 로그인 리다이렉트로 판단하던 클라이언트가 게이트웨이의 HTML 502 페이지를 인증 문제로 안내한 사례와, 상태 코드부터 보는 오류 분기 순서, 게이트웨이 요청 크기 제한에 맞춘 전송 속도 조절을 정리했다.
---

사내 게이트웨이 뒤에 있는 API로 데이터를 보내는 노트북 도구를 운영했습니다. 서버 설정을 바꾼 직후 도구가 "게이트웨이가 Authorization 헤더를 넘기지 않아 로그인 페이지로 보낸다"고 안내했는데, 실제로는 인증과 전혀 상관이 없었습니다.

## 오진의 구조

클라이언트는 이런 순서로 응답을 판정하고 있었습니다.

```python
resp = session.post(url, json=batch, allow_redirects=False)
if not resp.headers.get("Content-Type", "").startswith("application/json"):
    # 게이트웨이가 로그인 HTML을 돌려준 것으로 본다
    raise AuthError("로그인으로 리다이렉트됨: Authorization 전달 확인 필요")
resp.raise_for_status()
```

예전에 인증이 빠지면 게이트웨이가 로그인 HTML을 주던 경험을 그대로 규칙으로 만든 것입니다. 그런데 서버 파드가 재시작되는 동안 게이트웨이는 upstream에 닿지 못해 **기본 HTML 502 페이지**를 돌려줬습니다. HTML이니 첫 분기에 걸렸고, 사용자는 엉뚱하게 인증 설정을 뒤졌습니다.

## 고친 순서: 상태 코드 → 리다이렉트 → 본문 형식

```python
RETRYABLE = {502, 503, 504}

def classify(resp):
    if resp.status_code in RETRYABLE:
        return "upstream-unavailable"     # 서버 상태(재시작·배포 중)를 보라고 안내, 재시도 대상
    if resp.status_code in (301, 302, 303, 307, 308) and "login" in resp.headers.get("Location", ""):
        return "auth-redirect"            # 진짜 로그인 리다이렉트
    if resp.status_code in (401, 403):
        return "auth"
    if not resp.headers.get("Content-Type", "").startswith("application/json"):
        return "unexpected-body"          # 원인을 단정하지 말고 상태 코드와 본문 앞부분을 보여 준다
    return "ok" if resp.ok else "http-error"
```

원칙은 단순합니다. **가장 구체적인 신호부터 본다.** 상태 코드가 가장 구체적이고, 본문 형식은 가장 덜 구체적입니다. 본문 형식만으로 원인을 단정하는 분기는 맨 뒤에 두고, 그때도 "알 수 없음"으로 안내하며 근거(상태 코드, `Location`, 본문 앞 200자)를 같이 보여 주는 편이 낫습니다.

## 같은 게이트웨이의 또 다른 함정: 요청 크기

이 게이트웨이는 요청 본문이 일정 크기를 넘으면 파일 업로드로 보고 막았습니다. 한 번에 수천 건씩 묶어 보내던 배치가 통과하지 못했습니다. 문서화된 한도가 없어서 현장에서 값을 줄여 가며 통과하는 지점을 찾았고, 그 값을 도구의 기본값으로 박았습니다.

```python
BATCH_SIZE = 10       # 게이트웨이를 통과한 배치 크기
SEND_INTERVAL = 2.0   # 초, 배치 사이 간격

for chunk in chunks(rows, BATCH_SIZE):
    send(chunk)
    time.sleep(SEND_INTERVAL)
```

이렇게 하면 이론상 시간당 1.8만 건, 전송 시간까지 더하면 실제로는 약 1.5만 건으로 느려집니다. 그래서 문서에 "왜 이 값인지"와 예상 처리 속도를 같이 적었습니다. 근거 없이 남은 느린 기본값은 누군가 "최적화"하면서 다시 올려 버리기 쉽습니다.

## 정리

- 응답 판정은 상태 코드 → 리다이렉트 위치 → 본문 형식 순으로 한다. 본문 형식만으로 원인을 단정하지 않는다.
- 502/503/504는 인증 문제가 아니라 upstream 문제다. 재시도 대상으로 다루고 서버 상태를 보라고 안내한다.
- 중간 장비의 요청 크기·속도 제한은 실측해서 기본값으로 박고, 근거와 처리 속도를 함께 기록한다.

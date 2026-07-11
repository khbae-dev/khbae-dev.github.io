---
title: 외부 API 하나가 서버 전체를 죽이는 경로, 타임아웃 없는 HTTP 클라이언트
date: 2026-06-28 19:23:00 +09:00
categories: [Backend]
tags: [http, timeout, retry, resilience]
description: 타임아웃 없는 외부 호출이 스레드 풀 고갈로 번지는 구조와, connect 타임아웃·전체 deadline·멱등 재시도를 조합한 빠른 실패 전략을 정리했다.
---

장애 사례들을 보다 보면 놀랄 만큼 자주 나오는 패턴이 있습니다. 외부 API 하나가 느려졌을 뿐인데 우리 서버 전체가 다운되는 경우입니다.

## 고갈의 수순

HTTP 클라이언트에 타임아웃이 없으면(또는 기본값이 무한이면) 느려진 외부 API를 호출한 스레드는 응답을 영원히 기다립니다. 트래픽은 계속 들어오니 대기 스레드가 하나둘 늘고, 스레드 풀(또는 커넥션 풀)이 가득 차는 순간 외부 API와 무관한 요청까지 전부 실패합니다. 장애의 반경이 의존성 하나에서 서비스 전체로 넓어지는 겁니다.

## 타임아웃은 계층별로

타임아웃 하나로 뭉뚱그리지 말고 단계를 나눠 겁니다.

```javascript
// Node fetch 예시: 전체 deadline
const res = await fetch(url, {
  signal: AbortSignal.timeout(3000), // 전체 3초 상한
});
```

```yaml
# 일반화하면 이런 구성
connect_timeout: 1s     # TCP 연결 수립까지
read_timeout: 2s        # 응답 대기
total_deadline: 3s      # 재시도 포함 전체 상한
```

- connect 타임아웃은 짧게: 죽은 호스트는 빨리 판별된다.
- 전체 deadline은 우리 서비스의 SLA에서 역산한다. 상류가 5초를 기다려주는데 하류 호출에 30초를 주는 건 모순이다.

재시도는 멱등 요청(GET, 멱등 키 있는 PUT)에만, 지수 백오프와 함께 적용합니다. 비멱등 POST를 타임아웃 후 재시도하면 중복 처리 위험이 생깁니다.

## 정리

- 타임아웃 없는 외부 호출은 장애 전파 통로다. 기본값을 절대 믿지 않는다.
- connect/read/전체 deadline을 구분하고, 값은 SLA에서 역산한다.
- 빠르게 실패하고(fail fast), 재시도는 멱등 요청에만 백오프와 함께.

---
title: 고루틴 2만 개로 서버가 죽는 이유, 동시성에 상한 걸기
date: 2026-06-26 19:22:00 +09:00
categories: [Backend]
tags: [go, goroutine, concurrency, errgroup]
description: for 루프에서 무제한으로 go를 띄울 때 생기는 자원 고갈 문제와 errgroup.SetLimit·세마포어로 동시성 상한을 두는 패턴을 정리했다.
---

Go의 고루틴이 가볍다는 말을 "무한히 띄워도 된다"로 오해하면 생기는 일을 정리했습니다.

## 입력 크기 = 동시성이 되는 코드

```go
// 나쁨: 입력이 2만 건이면 고루틴 2만 개
for _, url := range urls {
    go fetch(url)
}
```

고루틴 자체는 KB 단위로 가볍지만, 각 고루틴이 잡는 것들이 무겁습니다. 소켓(FD), DB 커넥션, 응답 버퍼 메모리가 입력 수에 비례해 열립니다. 상대 서버 입장에선 동시 2만 요청 폭격이고, 우리 쪽은 FD 고갈·메모리 폭증으로 함께 죽습니다.

## 상한을 두는 두 가지 표준 패턴

**1. errgroup.SetLimit.** 에러 전파·대기까지 함께 처리돼 실무에서 가장 깔끔합니다.

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(100) // 동시 실행 100개 상한

for _, url := range urls {
    url := url
    g.Go(func() error {
        return fetch(ctx, url)
    })
}
if err := g.Wait(); err != nil { ... }
```

**2. 세마포어(버퍼 채널).** 의존성 없이 원시적으로 구현할 때 씁니다.

```go
sem := make(chan struct{}, 100)
for _, url := range urls {
    sem <- struct{}{}
    go func(u string) {
        defer func() { <-sem }()
        fetch(u)
    }(url)
}
```

상한값은 감이 아니라 병목 자원 기준으로 정합니다. DB 커넥션 풀이 50이면 동시성도 그 언저리, 외부 API rate limit이 있으면 그에 맞춥니다.

## 정리

- "고루틴은 싸다"는 말에는 "그 안에서 잡는 자원은 비싸다"가 생략돼 있다.
- 루프 안의 벌거벗은 go는 코드 리뷰에서 잡아야 할 신호다.
- errgroup.SetLimit 하나로 상한·에러 전파·대기가 다 해결된다.

---
title: 게이트웨이는 타임아웃인데 백엔드는 계속 도는 문제, deadline propagation
date: 2026-07-05 19:22:00 +09:00
categories: [Backend]
tags: [grpc, timeout, context, microservices]
description: 상위 호출이 취소돼도 하위 gRPC 작업이 좀비처럼 계속 실행되는 원인이 컨텍스트 미전파에 있음을 이해하고, deadline propagation으로 즉시 중단시키는 방법을 정리했다.
---

마이크로서비스에서 게이트웨이가 504를 반환했는데 그 뒤의 gRPC 서버들은 여전히 무거운 쿼리를 돌리고 있는 상황을 공부했습니다. 사용자는 이미 떠났는데 자원은 계속 타는, 좀비 작업 문제입니다.

## 원인: 컨텍스트를 끊고 새로 만들었다

gRPC(특히 Go)의 취소·타임아웃 정보는 `context.Context`에 실려 호출 체인을 따라 흐릅니다. 흔한 실수는 하위 호출 시 들어온 ctx 대신 새 컨텍스트를 만드는 겁니다.

```go
// 나쁨: 상위의 취소·데드라인과 단절된다
resp, err := client.Search(context.Background(), req)
```

이러면 상위가 타임아웃으로 취소돼도 하위는 그 사실을 모릅니다. 락, DB 커넥션, CPU를 붙잡은 채 끝까지 실행됩니다.

## 해결: 들어온 ctx를 그대로 전파

```go
func (s *Server) Handle(ctx context.Context, req *Req) (*Resp, error) {
    // 들어온 ctx를 그대로 하위 호출에 넘긴다
    resp, err := s.searchClient.Search(ctx, toSearchReq(req))
    ...
}
```

이렇게 하면 취소 신호와 잔여 시간이 gRPC 메타데이터(`grpc-timeout`)로 하위에 전달됩니다. 상위가 취소되는 순간 하위의 진행 중 RPC도 `context.Canceled`/`DeadlineExceeded`로 즉시 중단됩니다. 하위 서버 쪽에서도 긴 작업 중간중간 `ctx.Err()`를 확인해 협조적으로 멈추게 합니다.

계층별 데드라인 설계도 함께 챙깁니다. 게이트웨이 5s → 서비스 A 4s → 서비스 B 3s처럼 안쪽으로 갈수록 줄어드는 예산(budget) 구조여야 재시도할 시간이 남습니다.

## 정리

- 타임아웃은 설정이 아니라 "전파"가 본질이다. ctx를 끊는 순간 좀비가 생긴다.
- context.Background()가 핸들러 안에 보이면 일단 의심한다.
- 데드라인은 바깥에서 안쪽으로 줄어드는 예산으로 설계한다.

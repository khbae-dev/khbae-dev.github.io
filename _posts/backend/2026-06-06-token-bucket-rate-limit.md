---
title: 봇 하나가 서버 전체를 마비시키는 걸 막는 토큰버킷 rate limit
date: 2026-06-06 16:02:00 +09:00
categories: [Backend]
tags: [rate-limit, token-bucket, api, redis]
description: 속도 제한이 없을 때 단일 폭주 클라이언트가 DB를 포화시켜 정상 유저까지 마비되는 문제와, 토큰버킷 알고리즘 구현을 정리했다.
---

rate limit을 "악성 유저 차단 기능" 정도로 생각했는데, 공부해 보니 본질은 "정상 유저 보호 장치"였습니다.

## 한 명이 전체를 마비시키는 구조

속도 제한이 없는 API는 선착순입니다. 봇 하나가 초당 수백 요청을 던지면 그 요청들이 DB 커넥션과 워커를 선점하고, 정상 유저의 요청은 그 뒤에 줄을 서다 타임아웃됩니다. 장애의 원인이 "용량 부족"이 아니라 "분배 실패"인 겁니다.

## 토큰버킷: 평균과 순간을 함께 제어

토큰버킷은 버킷에 초당 r개의 토큰이 차오르고(최대 b개), 요청은 토큰을 하나 소비해야 통과하는 모델입니다. 평균 속도는 r로 묶으면서, 짧은 버스트는 b까지 허용하는 게 leaky bucket과의 차이입니다. 사용자 경험(연타, 페이지 진입 시 동시 호출)을 해치지 않으면서 지속 폭주만 걸러냅니다.

```lua
-- Redis Lua (원자적 실행): KEYS[1]=버킷 키, r=refill/s, b=용량
local tokens = tonumber(redis.call('HGET', KEYS[1], 't') or b)
local last   = tonumber(redis.call('HGET', KEYS[1], 'ts') or now)
tokens = math.min(b, tokens + (now - last) * r)
if tokens < 1 then return 0 end
redis.call('HSET', KEYS[1], 't', tokens - 1, 'ts', now)
redis.call('EXPIRE', KEYS[1], 3600)
return 1
```

초과 요청에는 429와 함께 `Retry-After` 헤더를 돌려주면, 잘 만든 클라이언트는 알아서 물러납니다. 키는 사용자 ID·API 키 기준을 기본으로 하고, IP는 NAT 뒤 다수 사용자를 오인 차단할 수 있어 보조로만 씁니다.

## 정리

- rate limit의 목적은 차단이 아니라 자원의 공정한 분배다.
- 토큰버킷은 평균 속도와 버스트 허용을 분리해 제어할 수 있어 API 기본값으로 적합하다.
- 분산 환경에서는 Redis Lua 같은 원자적 구현이어야 레이스가 없다.

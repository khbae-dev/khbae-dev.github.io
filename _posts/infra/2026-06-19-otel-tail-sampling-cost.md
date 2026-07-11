---
title: 관측 비용이 서버 비용을 넘어설 때, tail 샘플링
date: 2026-06-19 19:24:00 +09:00
categories: [Infra]
tags: [observability, tracing, sampling, opentelemetry]
description: 트레이스 전량 저장의 비용 문제를, 트레이스를 끝까지 본 뒤 에러·지연만 전부 남기고 정상은 일부만 남기는 tail 샘플링으로 해결하는 방법을 정리했다.
---

분산 트레이싱을 도입한 조직이 다음에 마주치는 문제를 공부했습니다. 관측 데이터 저장 비용이 정작 서비스 인프라 비용보다 커지는 역전 현상입니다.

## head 샘플링의 딜레마

비용을 줄이는 고전적 방법은 head 샘플링입니다. 트레이스 시작 시점에 "이 요청은 저장한다/버린다"를 확률로 결정하는 방식입니다. 문제는 결정 시점에 그 요청이 에러가 될지, 느려질지 알 수 없다는 겁니다. 5%만 남기면 비용은 줄지만, 장애 분석에 필요한 "그 느렸던 요청"이 95% 확률로 이미 버려져 있습니다.

## tail 샘플링: 다 보고 나서 고른다

tail 샘플링은 트레이스가 완결될 때까지 컬렉터가 버퍼링한 뒤, 내용을 보고 저장 여부를 결정합니다. 정책을 조합할 수 있습니다.

```yaml
# otel-collector tail_sampling processor
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors            # 에러 트레이스는 100% 보존
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow              # 느린 트레이스도 100% 보존
        type: latency
        latency: { threshold_ms: 1000 }
      - name: baseline          # 정상 트레이스는 5%만
        type: probabilistic
        probabilistic: { sampling_percentage: 5 }
```

결과적으로 디버깅 가치가 높은 데이터(에러·느린 요청)는 전부 남고, 가치가 낮은 정상 트레이스만 대표 표본으로 줄어듭니다. 저장량은 수십 퍼센트 절감되면서 장애 분석 능력은 오히려 유지됩니다.

비용은 컬렉터로 이동합니다. 결정까지 트레이스를 메모리에 들고 있어야 하므로 컬렉터 메모리와 decision_wait 튜닝이 필요하고, 트레이스의 모든 스팬이 같은 컬렉터에 모이도록 로드밸런싱(trace ID 기반) 구성도 챙겨야 합니다.

## 정리

- head 샘플링은 "무엇이 중요한지 모른 채" 버리는 방식이라 장애 분석과 상극이다.
- tail 샘플링은 에러·지연 100% + 정상 N%라는 합리적 절충을 만든다.
- 절감된 저장 비용의 일부는 컬렉터 자원으로 이동한다는 점까지 계산에 넣는다.

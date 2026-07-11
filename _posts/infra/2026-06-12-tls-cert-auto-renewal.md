---
title: 인증서 한 장 만료로 전 서비스가 멈추는 사고 예방하기
date: 2026-06-12 16:02:00 +09:00
categories: [Infra]
tags: [tls, certbot, monitoring, sre]
description: 수동 갱신에 의존하던 TLS 인증서가 만료돼 모든 HTTPS 연결이 거부되는 사고를, 자동 갱신과 만료 감시의 이중 안전망으로 막는 구성을 정리했다.
---

가장 허무하지만 여전히 흔한 장애 유형을 공부했습니다. TLS 인증서 만료입니다. 90일짜리 인증서를 손으로 갱신하는 운영은 언젠가 반드시 한 번 깜빡하게 돼 있습니다.

## 만료의 특징: 예고 없이, 전면적으로

인증서가 만료되는 순간 브라우저와 클라이언트는 연결 자체를 거부합니다. 성능 저하 같은 완충 구간 없이, 그 시각부로 모든 HTTPS 트래픽이 실패합니다. 내부 서비스 간 mTLS라면 서비스 메시 전체가 함께 멈춥니다. 예방이 전부인 장애입니다.

## 1차 방어: 자동 갱신

certbot 기준, 요즘 배포판은 systemd 타이머가 기본입니다.

```bash
# 타이머 활성 확인 (하루 2회 갱신 시도, 만료 30일 전부터 갱신됨)
systemctl list-timers | grep certbot

# 갱신 후 무중단 반영 훅
# /etc/letsencrypt/renewal-hooks/deploy/reload.sh
#!/bin/bash
nginx -s reload
```

포인트는 갱신 자체보다 갱신 "후"입니다. 새 인증서를 프로세스가 다시 읽도록 reload 훅까지 연결돼야 하고, `certbot renew --dry-run`으로 전체 경로를 미리 검증해 둡니다.

## 2차 방어: 만료 감시

자동화도 고장 납니다(DNS 변경으로 챌린지 실패, 훅 오류, 잊힌 서버). 그래서 독립적인 감시가 필요합니다. Prometheus blackbox exporter가 정석입니다.

```yaml
# 인증서 만료까지 남은 시간으로 알람
- alert: CertExpirySoon
  expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 21   # D-21
  labels: { severity: warning }
```

갱신은 D-30에 일어나야 정상이므로, D-21에도 만료가 가깝다면 자동화가 고장 났다는 뜻입니다. 알람이 "만료 임박"이 아니라 "자동화 고장 감지"로 동작하는 설계입니다.

## 정리

- 인증서 운영의 목표는 갱신이 아니라 "갱신을 사람이 기억하지 않아도 되는 상태"다.
- 자동 갱신 + reload 훅 + dry-run 검증이 1차 방어선.
- 독립적인 만료 감시(D-21 알람)가 자동화 고장까지 잡는 2차 방어선이다.

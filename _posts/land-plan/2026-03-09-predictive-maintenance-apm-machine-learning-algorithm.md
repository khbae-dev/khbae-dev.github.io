---
title: "Predictive Maintenance를 위한 APM 머신러닝 알고리즘 정리"
date: 2026-03-09 13:00:00 +09:00
tags: [apm, predictive-maintenance, machine-learning, autoencoder, rca]
description: "비지도 학습 기반 APM 모델, Health Index, Worst Parameter, RUL, 지도학습 기반 RCA Probability Model까지 예지 정비 알고리즘 구조를 기술적으로 정리한다."
---

# Predictive Maintenance를 위한 APM 머신러닝 알고리즘 정리

설비 운영 환경에서는 장비 상태를 지속적으로 모니터링하고, 고장을 사전에 감지하며, 남은 수명을 예측하는 **Predictive Maintenance**가 중요합니다.

이번 글은 설비 데이터 기반의 **APM (Asset Performance Management)** 알고리즘을 기술 블로그 스타일로 정리한 문서입니다.
핵심은 크게 두 축입니다.

- 비지도 학습 기반 설비 상태 감지 모델
- 지도 학습 기반 원인 분석 및 장애 확률 모델

즉 "이상이 있는가"를 먼저 감지하고, 그 다음 "무엇이 원인인가"를 분리해서 다루는 구조입니다.

---

## 1. APM 개요

APM 시스템의 전체 흐름은 아래처럼 정리할 수 있습니다.

```text
Normal Trace Data
  -> APM Model (Unsupervised, Autoencoder)
    -> Health Index
    -> Worst Parameter
    -> Threshold Comparison
    -> RUL Prediction
      -> RCA Model (Supervised)
        -> Failure Probability
        -> Feature Importance
```

입력 데이터는 보통 다음 세 가지를 사용합니다.

| Input | 설명 |
| --- | --- |
| Trace Data | 정상 상태 설비 시계열 데이터 |
| Parameter Weight | 센서 중요도 가중치 |
| Parameter Spec | 센서 정상 범위 및 운영 스펙 |

출력은 대체로 다음과 같습니다.

- Health Index
- Worst Parameter
- RUL (Remaining Useful Life)
- Failure Probability

APM은 단순한 이상 탐지 모델이 아니라, 설비 상태를 점수화하고 운영자가 해석 가능한 결과로 변환하는 시스템이라고 보는 편이 맞습니다.

---

## 2. 비지도 학습 기반 APM 모델

산업 현장 데이터의 현실은 대부분 비슷합니다.

- 정상 데이터는 많다
- 고장 데이터는 적다
- 고장 라벨이 있어도 품질이 들쭉날쭉하다

그래서 APM 모델은 **정상 패턴만 학습한 뒤, 정상에서 얼마나 벗어났는지**를 보는 비지도 학습 구조를 자주 사용합니다.
가장 대표적인 방법이 **Autoencoder**입니다.

### 2-1. Autoencoder 구조

```text
Input Layer -> Hidden Layer (Latent Space) -> Output Layer
```

Autoencoder는 입력을 압축한 뒤 다시 복원합니다.
정상 데이터로만 학습하면 정상 패턴은 잘 복원하지만, 이상 징후가 있는 입력은 복원 오차가 커집니다.

학습 목표는 간단합니다.

```text
input x -> reconstructed x_hat
loss = reconstruction error
```

즉 모델이 "정상이라면 어떻게 보여야 하는가"를 먼저 배우고, 실제 입력이 그 패턴에서 얼마나 벗어나는지를 이상 신호로 해석합니다.

### 2-2. 왜 비지도 학습이 유리한가

- 고장 사례가 적어도 학습 가능
- 설비별 고장 타입이 달라도 정상 기준선 학습 가능
- 초기 운영 단계에서도 적용 가능

대신 단점도 있습니다.

- 정상 데이터 품질이 낮으면 모델 기준선이 흔들린다
- 운영 조건 변경 시 기준 재학습이 필요할 수 있다
- 이상 여부는 잘 잡아도 원인을 바로 설명하지는 못한다

이 한계를 보완하기 위해 뒤에서 설명할 RCA 모델이 함께 붙습니다.

---

## 3. Health Index 계산 방식

설비 상태를 숫자로 표현하기 위해 **Health Index (HI)** 를 사용합니다.
가장 일반적인 기준은 Autoencoder의 **Reconstruction Error**입니다.

대표 구현은 MAE 기반입니다.

```text
MAE_t = (1 / n) * sum(|x_t,j - x_hat_t,j|)
```

- `x_t,j`: 시점 `t`의 실제 센서 값
- `x_hat_t,j`: 모델이 복원한 값
- `n`: 파라미터 개수

Health Index는 보통 이 재구성 오차를 그대로 쓰거나, 정규화한 값으로 변환합니다.

```text
HI_t = f(MAE_t)
```

실무적으로는 다음처럼 해석하면 됩니다.

| 상태 | Health Index |
| --- | --- |
| 정상 | 낮음 |
| 이상 | 높음 |

즉 Health Index는 "현재 상태가 정상 패턴에서 얼마나 멀어졌는가"를 나타내는 점수입니다.

### 3-1. 구현 시 자주 하는 추가 처리

- 센서별 스케일 차이 제거를 위한 정규화
- 짧은 노이즈 완화를 위한 rolling average
- 구간별 상태 비교를 위한 min-max scaling

Health Index 자체보다 중요한 것은 **일관된 계산 규칙**입니다.
같은 설비에서 시간이 지남에 따라 비교 가능해야 의미가 있습니다.

---

## 4. Worst Parameter 계산 방식

Health Index가 올라갔다면, 다음 질문은 항상 같습니다.

> 어떤 센서가 이 이상 점수에 가장 크게 기여했는가?

이를 위해 파라미터별 오차를 따로 계산합니다.

```text
error_t,k = |x_t,k - x_hat_t,k|
```

그 다음 각 파라미터의 상대 기여도를 계산합니다.

```text
score_t,k = error_t,k / max(error_t,*)
```

여기서 `max(error_t,*)`는 같은 시점에서 가장 큰 오차를 의미합니다.
즉 가장 튀는 파라미터를 1에 가깝게 두고 나머지를 상대 비율로 보는 방식입니다.

실무에서는 단발성 스파이크를 줄이기 위해 moving average를 함께 쓰는 편이 많습니다.

```text
worst_parameter_score_t,k
  = moving_average(score_t,k) * (HI_t / threshold)
```

이렇게 하면 다음 효과가 있습니다.

- 순간 노이즈보다 지속적인 이상 신호를 더 잘 본다
- Health Index가 높을수록 원인 센서 우선순위가 더 분명해진다

Worst Parameter는 유지보수 관점에서 매우 중요합니다.
운영자는 "이상이 있다"보다 "어느 센서부터 확인해야 하는가"를 먼저 알고 싶기 때문입니다.

---

## 5. Parameter Weight 적용

산업 설비에서는 모든 센서가 같은 중요도를 가지지 않습니다.
예를 들어 진동, 압력, 온도는 핵심인데 유량이나 보조 센서는 상대적으로 덜 중요할 수 있습니다.

예시:

| 센서 | 중요도 |
| --- | --- |
| 온도 | 높음 |
| 압력 | 높음 |
| 진동 | 매우 높음 |
| 유량 | 중간 |

이 차이를 반영하기 위해 **Parameter Weight**를 적용합니다.

가중치를 적용한 Health Index는 보통 아래처럼 계산합니다.

```text
weighted_HI_t = sum(w_k * |x_t,k - x_hat_t,k|) / sum(w_k)
```

- `w_k`: 센서 `k`의 중요도 가중치

이 방식의 장점은 명확합니다.

- 중요한 센서에 대한 민감도를 높일 수 있다
- 상대적으로 중요하지 않은 노이즈 변수의 영향은 줄일 수 있다
- 현장 도메인 지식을 모델 점수에 반영할 수 있다

Worst Parameter 계산에도 가중치를 함께 적용할 수 있습니다.

```text
weighted_error_t,k = w_k * |x_t,k - x_hat_t,k|
```

즉 APM은 순수 통계 모델이라기보다, 도메인 지식과 데이터 모델을 함께 쓰는 구조라고 보는 편이 맞습니다.

---

## 6. Health Index Threshold 계산

Health Index를 계산했다면, 그다음은 **어느 지점부터 이상으로 볼 것인가**를 정해야 합니다.
이때 기준선 역할을 하는 값이 Threshold입니다.

가장 흔한 방식은 정상 데이터 분포를 이용하는 통계 기반 접근입니다.

### 6-1. 3-Sigma Rule

정상 구간의 Health Index 분포를 기준으로 다음처럼 계산할 수 있습니다.

```text
UCL = mu + 3 * sigma
LCL = mu - 3 * sigma
```

- `mu`: 정상 구간 평균
- `sigma`: 정상 구간 표준편차

Health Index는 보통 음수가 아니므로 실제 구현에서는 UCL만 쓰는 경우가 많습니다.

### 6-2. 최대값 기반 Threshold

더 단순한 방식으로는 학습 구간의 최대 Health Index를 threshold로 두기도 합니다.

```text
threshold = max(HI_train)
```

이 방식은 구현이 단순하지만, 학습 데이터 품질에 민감합니다.
그래서 실무에서는 아래 두 가지를 함께 고려하는 경우가 많습니다.

- `max(HI_train)` 기반 운영 초기 기준선
- `mu + 3sigma` 기반 통계 threshold

Threshold는 너무 낮으면 오탐이 늘고, 너무 높으면 실제 이상을 늦게 잡습니다.
따라서 모델 품질만큼 중요한 것이 threshold 튜닝입니다.

---

## 7. RUL (Remaining Useful Life) 예측

RUL은 설비가 임계 상태에 도달하기까지 남은 시간을 의미합니다.
즉 현재가 이상인지 보는 것을 넘어, **언제 유지보수가 필요해질지**를 예측하는 단계입니다.

기본 아이디어는 단순합니다.

- 시간이 지날수록 Health Index가 증가한다고 가정
- 현재까지의 추세를 기반으로 threshold 도달 시점을 예측
- 남은 시간을 RUL로 계산

```text
RUL = t_threshold - t_now
```

### 7-1. 추세 모델

대표적으로 다음 모델을 사용합니다.

- Linear Regression
- Exponential / Non-linear degradation model
- Piecewise regression

예를 들어 선형 추세를 가정하면 다음처럼 볼 수 있습니다.

```text
HI(t) = a * t + b
```

Threshold와 만나는 시점은 다음과 같습니다.

```text
t_threshold = (threshold - b) / a
```

이때 주의할 점:

- 아직 열화가 충분히 나타나지 않은 초반 구간에서는 RUL 신뢰도가 낮다
- 운영 조건 변화가 있으면 과거 추세가 깨질 수 있다
- 설비별 열화 패턴이 선형이 아닐 수 있다

그래서 RUL은 단일 숫자만 내는 것보다, 예측 범위나 confidence와 함께 보는 편이 더 안전합니다.

---

## 8. 지도학습 기반 RCA Probability Model

APM 모델이 이상을 감지한다면, RCA 모델은 **왜 그런 이상이 발생했는지**를 확률적으로 설명합니다.

여기서는 지도학습 기반 분류 모델을 사용합니다.

### 8-1. 입력 데이터

보통 다음 정보를 feature로 사용합니다.

- RCA Alarm period data
- 주요 센서 파라미터 값
- 구간 통계량(평균, 최대, 최소, 변화율)
- 이벤트 발생 전후의 시계열 윈도우 feature

### 8-2. 모델 예시

대표적으로 Random Forest 같은 분류 모델을 사용할 수 있습니다.

```text
input features -> classifier -> failure probability
```

출력은 단순 라벨보다 **확률**이 더 유용합니다.

```text
P(failure = true | current features)
```

즉 운영자는 "고장이다/아니다"보다 "현재 조건에서 어떤 장애가 얼마나 유력한가"를 볼 수 있습니다.

### 8-3. SHAP 기반 원인 분석

RCA 모델 결과를 해석하기 위해 **SHAP (Shapley Additive Explanations)** 를 붙이는 경우가 많습니다.

개념적으로는 아래와 같이 이해할 수 있습니다.

```text
prediction = expected_value + sum(feature_contribution)
```

SHAP을 쓰면 다음을 설명할 수 있습니다.

- 어떤 파라미터가 확률을 올렸는가
- 어떤 파라미터가 방어적으로 작용했는가
- 동일 장애 유형에서도 설비별 원인 차이가 무엇인가

즉 APM이 "상태 이상 감지", RCA가 "원인 후보 설명"이라면, SHAP은 "모델 판단 근거 시각화"에 가깝습니다.

---

## 9. 전체 아키텍처 정리

최종적으로 시스템은 아래 구조로 볼 수 있습니다.

```text
Real-Time Sensor Data
  -> Preprocessing / Normalization
    -> APM Model (Unsupervised Autoencoder)
      -> Health Index
      -> Worst Parameter
      -> Threshold Check
      -> RUL Prediction
        -> RCA Model (Supervised Classifier)
          -> Failure Probability
          -> Parameter Importance (SHAP)
```

이 구조의 장점은 역할 분리가 분명하다는 점입니다.

- APM 모델: 이상 감지
- Health Index: 상태 수치화
- Worst Parameter: 즉시 점검 대상 제시
- RUL: 유지보수 시점 예측
- RCA Probability Model: 원인 후보 확률화
- SHAP: 모델 판단 설명력 제공

결국 Predictive Maintenance 시스템의 가치는 아래 4가지로 요약할 수 있습니다.

1. 설비 상태를 점수화한다
2. 주요 이상 원인을 빠르게 좁힌다
3. 임계 시점까지 남은 시간을 예측한다
4. 장애 발생 확률과 해석 근거를 함께 제공한다

즉 운영자는 단순 경보를 넘어서,

- 이상을 조기에 감지하고
- 원인을 분석하고
- 정비 시점을 계획하는

데이터 기반 유지보수를 수행할 수 있게 됩니다.

---

## 정리

APM 기반 Predictive Maintenance는 단일 모델 하나로 끝나지 않습니다.
정상 패턴을 학습하는 비지도 모델, 상태 점수화 로직, threshold 규칙, RUL 예측, 지도학습 기반 RCA 모델이 함께 동작해야 실제 운영 가치가 나옵니다.

특히 중요한 포인트는 아래와 같습니다.

- Autoencoder는 정상 패턴 학습에 강하다
- Health Index는 재구성 오차를 운영 가능한 점수로 바꾸는 핵심 지표다
- Worst Parameter와 Weight는 현장 해석 가능성을 높인다
- Threshold와 RUL은 운영 액션 시점을 결정한다
- RCA + SHAP은 "왜 이런 경보가 나왔는가"를 설명한다

다음 단계에서는 이 구조를 바탕으로 **Python / PyTorch 예제 코드**, **Health Index 계산 모듈**, **시계열 추세 기반 RUL 샘플 구현**까지 이어서 정리할 수 있습니다.

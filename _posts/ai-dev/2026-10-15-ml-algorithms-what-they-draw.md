---
title: 머신러닝 알고리즘 다섯 가지, 그래프 위에 무엇을 그리는가로 비교하기
date: 2026-10-15 20:00:00 +09:00
categories: [AI Dev]
tags: [machine-learning, scikit-learn, classification, regression, basics]
description: 선형 회귀, 로지스틱 회귀, 결정 트리, SVM, KNN을 각 모델이 데이터 위에 그리는 경계의 모양(직선, S자 곡선, 사각형, 최대 마진, 그리지 않음)으로 비교하고, 언제 무엇을 쓰는지 정리했다.
---

머신러닝 입문 자료는 보통 알고리즘마다 그림을 따로 보여 줍니다. 그러면 각각은 이해가 되는데 서로 뭐가 다른지는 잘 안 보입니다. 최근 본 [짧은 영상](https://www.instagram.com/reel/DegrQPzvT9z/)(@durumag.kr)이 이 문제를 깔끔하게 풀었는데, 기준을 하나로 고정하는 방식이었습니다. **"이 모델은 그래프 위에 무엇을 그리는가?"** 그 관점을 빌려 직접 정리해 봤습니다.

## 한눈에 보기

| 알고리즘 | 그리는 것 | 푸는 문제 | 학습 결과로 남는 것 |
|---|---|---|---|
| 선형 회귀 | 직선 (고차원에선 평면) | 회귀 (숫자 예측) | 기울기와 절편 |
| 로지스틱 회귀 | S자 곡선, 직선 경계 | 분류 (확률) | 가중치 |
| 결정 트리 | 축에 평행한 사각형 | 분류·회귀 | 질문(분기 규칙)의 나무 |
| SVM | 가장 넓은 틈의 가운데 선 | 분류 | 서포트 벡터 |
| KNN | 아무것도 안 그림 | 분류·회귀 | 학습 데이터 그 자체 |

## 1. 선형 회귀: 직선을 긋는다

점들 사이를 가장 잘 지나가는 직선 `y = wx + b` 하나를 찾습니다. "가장 잘"은 각 점과 직선 사이 세로 거리(오차)의 제곱합이 가장 작다는 뜻입니다.

![선형 회귀 예시](https://khbae-dev.github.io/assets/img/posts/ml-linear-regression.png)
_면적과 가격 데이터에 맞춘 직선. 회색 선이 각 점의 오차입니다_

- 출력이 연속된 숫자입니다. 면적으로 집값 예측하기 같은 문제입니다.
- 결과가 계수 몇 개라서 해석이 쉽습니다. "면적이 1㎡ 늘면 가격이 w만큼 오른다."
- 데이터가 곡선 모양이면 직선은 구조적으로 못 따라갑니다. 이때는 특성을 늘리거나(다항 특성) 다른 모델을 씁니다.

## 2. 로지스틱 회귀: S자 곡선을 그린다

이름은 회귀지만 **분류** 모델입니다. 선형 회귀처럼 `wx + b`를 계산한 뒤, 그 값을 시그모이드 함수에 넣어 0과 1 사이의 확률로 바꿉니다. 이 함수 모양이 S자입니다.

```text
p(합격) = 1 / (1 + e^-(wx + b))
```

![로지스틱 회귀 예시](https://khbae-dev.github.io/assets/img/posts/ml-logistic-regression.png)
_공부 시간에 따른 합격 확률. 확률 0.5가 되는 지점이 경계입니다_

- 확률이 0.5가 되는 지점이 분류 경계입니다. 입력 공간에서 보면 이 경계는 여전히 **직선**입니다.
- "몇 % 확률로 스팸인가"처럼 확률 자체가 필요할 때 유용합니다.
- 빠르고 해석하기 쉬워서 분류 문제의 첫 번째 기준선(baseline)으로 많이 씁니다.

## 3. 결정 트리: 사각형으로 나눈다

"x가 3보다 큰가?", "y가 5보다 작은가?" 같은 질문을 반복해 공간을 쪼갭니다. 질문 하나가 축에 평행한 선 하나라서, 결과적으로 평면이 **사각형 칸들**로 나뉘고 칸마다 답이 정해집니다.

![결정 트리 예시](https://khbae-dev.github.io/assets/img/posts/ml-decision-tree.png)
_깊이 3의 트리가 만든 경계. 모든 선이 축에 평행합니다_

- 사람이 그대로 읽을 수 있는 규칙이 나옵니다. 스케일링 같은 전처리도 거의 필요 없습니다.
- 질문을 계속 하면 점 하나하나를 감싸는 작은 칸까지 만들어 **과적합**하기 쉽습니다. 깊이(`max_depth`)를 제한하거나 가지를 칩니다.
- 대각선 경계는 계단 모양으로만 흉내 냅니다. 여러 나무를 합친 랜덤 포레스트나 그레이디언트 부스팅이 이 약점을 보완합니다.

## 4. SVM: 가장 넓은 틈을 찾는다

두 그룹을 가르는 직선은 무수히 많습니다. SVM은 그중 **양쪽 가장 가까운 점까지 거리(마진)가 가장 큰** 선을 고릅니다. 두 그룹 사이에 가장 넓은 도로를 깔고 그 중앙선을 경계로 삼는 셈입니다.

![SVM 예시](https://khbae-dev.github.io/assets/img/posts/ml-svm.png)
_가장 넓은 마진(회색 띠)의 중앙선이 경계. 빨간 원이 서포트 벡터입니다_

- 경계를 정하는 데는 도로 가장자리에 닿은 점들, 즉 **서포트 벡터**만 쓰입니다. 나머지 점은 움직여도 결과가 같습니다.
- 커널(RBF 등)을 쓰면 데이터를 더 높은 차원에 올려서 곡선 경계도 만들 수 있습니다.
- 특성 스케일에 민감해서 표준화가 사실상 필수이고, 데이터가 수십만 건을 넘으면 느려집니다.

## 5. KNN: 아무것도 그리지 않는다

KNN은 학습 단계에서 **아무것도 만들지 않습니다.** 데이터를 그대로 저장해 둘 뿐입니다. 새 점이 들어오면 가장 가까운 k개를 찾아 다수결(분류)이나 평균(회귀)으로 답합니다. 그래서 "게으른 학습(lazy learning)"이라고 부릅니다.

![KNN 예시](https://khbae-dev.github.io/assets/img/posts/ml-knn.png)
_새 점(별)에서 가장 가까운 5개가 투표합니다. 3:2로 class 0_

- 학습은 즉시 끝나지만 예측할 때마다 전체 데이터와 거리를 재야 해서 느립니다.
- k가 작으면 경계가 들쭉날쭉해지고(과적합), 크면 너무 뭉개집니다.
- 거리 기반이라 스케일링이 필수이고, 차원이 많아지면 "가깝다"는 개념 자체가 흐려집니다(차원의 저주).

## 직접 그려 보기

scikit-learn으로 같은 데이터에 분류 모델 네 개를 학습시키고 경계를 그려 보면 차이가 바로 보입니다. 선형 회귀는 분류 모델이 아니라서 뺐습니다.

```python
import matplotlib.pyplot as plt
from sklearn.datasets import make_moons
from sklearn.inspection import DecisionBoundaryDisplay
from sklearn.linear_model import LogisticRegression
from sklearn.neighbors import KNeighborsClassifier
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.svm import SVC
from sklearn.tree import DecisionTreeClassifier

X, y = make_moons(n_samples=300, noise=0.25, random_state=0)

models = {
    "Logistic Regression": LogisticRegression(),
    "Decision Tree": DecisionTreeClassifier(max_depth=4),
    "SVM (RBF)": make_pipeline(StandardScaler(), SVC(kernel="rbf")),
    "KNN (k=15)": make_pipeline(StandardScaler(), KNeighborsClassifier(n_neighbors=15)),
}

fig, axes = plt.subplots(1, 4, figsize=(16, 4))
for ax, (name, model) in zip(axes, models.items()):
    model.fit(X, y)
    DecisionBoundaryDisplay.from_estimator(model, X, ax=ax, alpha=0.3, response_method="predict")
    ax.scatter(X[:, 0], X[:, 1], c=y, s=12, edgecolor="k")
    ax.set_title(f"{name}  acc={model.score(X, y):.2f}")
plt.tight_layout()
plt.show()
```

![네 분류 모델의 결정 경계 비교](https://khbae-dev.github.io/assets/img/posts/ml-algorithms-decision-boundaries.png)
_같은 초승달 데이터에 학습시킨 네 모델의 결정 경계_

초승달 두 개가 맞물린 데이터에서 로지스틱 회귀는 직선 하나로만 가르고, 결정 트리는 계단 모양 사각형, RBF SVM은 매끄러운 곡선, KNN은 데이터 모양을 그대로 따라가는 울퉁불퉁한 경계를 만듭니다. 제목의 정확도는 학습 데이터 기준이라 실제 성능이 아니라 "이 모양으로 데이터를 얼마나 감쌀 수 있는가" 정도로 읽으면 됩니다. 직선밖에 못 긋는 로지스틱 회귀가 0.83에서 막히는 것이 그리는 모양의 한계를 그대로 보여 줍니다.

## 무엇부터 써 볼까

- **숫자 예측**이면 선형 회귀로 기준선을 잡습니다.
- **분류**면 로지스틱 회귀로 기준선을 잡고, 결과를 설명해야 하면 결정 트리를 봅니다.
- 경계가 복잡하고 데이터가 많지 않으면 SVM을 시도합니다.
- 데이터가 작고 빠르게 감을 잡고 싶으면 KNN도 괜찮습니다.
- 표 형태의 실무 데이터에서 성능이 필요하면 결국 트리 앙상블(랜덤 포레스트, XGBoost, LightGBM)로 가는 경우가 많습니다.

## 정리

- 다섯 알고리즘은 "그래프 위에 무엇을 그리는가"로 비교하면 차이가 한 번에 보인다. 직선, S자 곡선, 사각형, 가장 넓은 틈, 그리고 아무것도 그리지 않음.
- 그리는 모양이 곧 그 모델이 표현할 수 있는 패턴의 한계다. 데이터 모양과 맞지 않으면 아무리 튜닝해도 따라가지 못한다.
- 거리를 쓰는 SVM과 KNN은 스케일링이 필수이고, 트리는 과적합 제어가 핵심이다.

---
title: JSON에는 NaN이 없다, 대량 전송이 중간에 멈춘 이유
date: 2026-10-15 19:26:00 +09:00
categories: [Backend]
tags: [json, python, pandas, kafka, data-pipeline]
description: Python의 json 모듈이 기본으로 내보내는 NaN이 표준 JSON이 아니어서 서버가 거부한 문제와, 오류 위치를 정확히 찍는 방법, NaN을 null로 바꾸는 결정을 기본값이 아닌 선택지로 둔 이유를 정리했다.
---

DB에서 시계열 데이터를 뽑아 JSON으로 묶어 메시지 큐로 다시 보내는 도구를 만들었습니다. 수십 개 대상을 한 번에 뽑다가 중간에 멈췄는데, 메시지가 너무 짧아서 어디서 왜 멈췄는지 알 수 없었습니다.

```text
Stopped: NaN/Infinity found; use --nonfinite null only if intended
```

## JSON 표준에는 NaN이 없다

JSON(RFC 8259)의 숫자에는 `NaN`, `Infinity`, `-Infinity`가 없습니다. 그런데 Python의 `json.dumps`는 기본값(`allow_nan=True`)으로 이 값을 그대로 씁니다.

```python
>>> json.dumps({"v": float("nan")})
'{"v": NaN}'          # 표준 JSON이 아니다
>>> json.dumps({"v": float("nan")}, allow_nan=False)
ValueError: Out of range float values are not JSON compliant
```

받는 쪽이 Jackson처럼 엄격한 파서라면 이 메시지를 거부합니다. JavaScript의 `JSON.parse`도 마찬가지입니다. pandas로 DB 값을 읽으면 SQL의 NULL이나 결측이 `NaN`이 되는 경우가 많아서, 데이터 파이프라인에서 특히 자주 만나는 문제입니다.

## 1단계: 어디서 났는지 찍는다

원래 코드는 행 전체를 한 번에 정규화하고 있어서 어느 값이 문제인지 잃어버렸습니다. 컬럼별로 돌면서 실패한 키와 그 행의 시각을 붙여 다시 던지게 바꿨습니다.

```python
def normalize_row(row, nonfinite):
    out = {}
    for key, value in row.items():
        try:
            out[key] = normalize_value(value, nonfinite)
        except NonFiniteError as e:
            raise NonFiniteError(f"{row['source']}.{key} at {row['event_time']}: {e}") from None
    return out

def normalize_value(v, nonfinite):
    if isinstance(v, float) and not math.isfinite(v):
        if nonfinite == "null":
            return None
        raise NonFiniteError("NaN/Infinity found")
    return v
```

이제 메시지만 보고 원천 데이터를 바로 열어 볼 수 있습니다.

```text
Stopped: source-07.col_a at 2026-08-31T00:01:00Z: NaN/Infinity found
```

## 2단계: null로 바꿀지는 사람이 정한다

`NaN`을 `null`로 바꾸면 전송은 됩니다. 하지만 이건 데이터의 의미를 바꾸는 일입니다. "측정값 없음"과 "측정했는데 계산 불가"는 다를 수 있고, 받는 쪽이 null을 0으로 다루는지 건너뛰는지에 따라 결과도 달라집니다.

그래서 도구의 기본값은 계속 `error`로 두고, `--nonfinite null`은 명시적으로 고르게 했습니다. 이번 작업에서는 "그 값만 null로 내보내고 같은 행의 다른 컬럼과 나머지 행은 그대로 보낸다"고 결정했기 때문에, 이 작업용 실행 설정의 기본값만 `null`로 바꿨습니다. 도구의 기본값과 작업의 기본값을 구분한 것입니다.

이 작업에서 기본값을 바꾼 데는 운영상 이유도 있었습니다. 추출은 중간 재개가 안 돼서, 한 번 잊고 실행하면 처음부터 다시 떠야 했습니다.

## 정리

- 표준 JSON에는 NaN과 Infinity가 없다. Python은 기본으로 이를 내보내므로 `allow_nan=False`로 일찍 실패시키는 편이 안전하다.
- 변환 오류는 키와 행 식별자, 시각을 붙여 던진다. 오류 메시지가 곧 다음 행동을 정한다.
- NaN을 null로 바꾸는 것은 데이터 의미를 바꾸는 결정이다. 도구 기본값은 보수적으로 두고 작업별로 명시해서 고른다.

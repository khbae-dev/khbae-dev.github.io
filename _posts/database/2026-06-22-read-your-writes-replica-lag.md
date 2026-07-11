---
title: 방금 저장한 데이터가 사라져 보이는 이유, 복제 지연과 read-your-writes
date: 2026-06-22 19:21:00 +09:00
categories: [Database]
tags: [database, replication, consistency, read-replica]
description: 쓰기는 원본, 읽기는 복제본으로 분리한 구조에서 방금 쓴 데이터가 잠깐 안 보이는 현상과 read-your-writes 패턴을 정리했다.
---

읽기 부하 분산을 위해 리드 레플리카를 붙이면 반드시 따라오는 문제가 있다는 걸 정리해 둡니다. "방금 저장했는데 목록에 없어요"라는 바로 그 현상입니다.

## 원인은 복제 지연

쓰기는 primary로, 읽기는 replica로 가는 구조에서 복제는 비동기로 이뤄집니다. 쓰기 직후의 읽기가 아직 복제가 도달하지 않은 replica에 떨어지면, 방금 커밋한 데이터가 조회되지 않습니다. 수십 ms~수 초의 짧은 창이지만, 사용자가 저장 직후 화면을 새로고침하는 패턴과 정확히 겹치기 때문에 체감 빈도는 낮지 않습니다.

## read-your-writes 패턴

핵심 아이디어는 단순합니다. "내가 방금 쓴 것은 내가 읽을 수 있어야 한다"는 보장만 선택적으로 제공하는 겁니다.

```javascript
// 쓰기 발생 시 세션에 표시를 남기고,
// 짧은 시간 동안 그 사용자의 읽기는 primary로 보낸다
async function afterWrite(session) {
  session.readFromPrimaryUntil = Date.now() + 3000;
}

function pickPool(session) {
  return Date.now() < (session.readFromPrimaryUntil ?? 0)
    ? primaryPool
    : replicaPool;
}
```

전체 트래픽은 계속 replica로 분산하면서, 정합성이 중요한 좁은 구간(쓴 직후의 본인)만 primary로 고정하는 절충입니다. GTID나 LSN을 확인해 "복제가 내 쓰기 지점을 따라잡았는지"를 검사하는 더 정밀한 방식도 있습니다.

## 정리

- 리드 레플리카를 쓰는 순간 최종 일관성(eventual consistency)의 세계에 들어온 것이다.
- 전부 강한 일관성으로 풀면 분산의 의미가 없다. 필요한 구간만 read-your-writes로 좁혀 잡는다.
- "저장 직후 조회" 흐름이 있는 기능(글 작성, 설정 변경)부터 적용하면 된다.

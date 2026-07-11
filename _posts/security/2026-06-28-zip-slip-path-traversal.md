---
title: 업로드한 zip 하나로 서버가 장악되는 Zip Slip
date: 2026-06-28 19:21:00 +09:00
categories: [Security]
tags: [security, zip-slip, path-traversal, upload]
description: 압축 해제 시 엔트리 이름의 상위 경로 이동을 검증하지 않아 임의 파일이 덮어써지는 Zip Slip 취약점과 canonical 경로 검증 방어를 정리했다.
---

압축 파일 업로드 기능에 숨은 Zip Slip 취약점을 공부했습니다. zip을 푸는 평범한 코드가 임의 파일 쓰기와 RCE로 이어지는 경로입니다.

## `../`를 그대로 믿는 코드

zip 아카이브의 각 엔트리는 파일명을 담고 있는데, 이 이름에 상대 경로를 넣을 수 있습니다. 압축 해제 코드가 엔트리 이름을 목적지 경로에 그냥 이어붙이면 문제가 생깁니다.

```java
// 취약: entry 이름을 검증 없이 join
File out = new File(destDir, entry.getName());
// entry.getName()이 "../../../../etc/cron.d/evil" 이면?
// destDir을 벗어나 시스템 파일을 덮어쓴다
```

공격자가 `../../../`를 앞세운 엔트리명으로 zip을 만들면, 압축 해제가 목적지 디렉터리를 탈출해 임의 위치에 파일을 씁니다. cron 파일, SSH authorized_keys, 웹 루트의 스크립트를 덮어쓰면 곧 코드 실행입니다.

## 방어: 실제 경로가 목적지 안인지 확인

```java
File destDir = new File("/app/uploads").getCanonicalFile();
for (ZipEntry entry : entries) {
    File out = new File(destDir, entry.getName()).getCanonicalFile();
    // canonical(심볼릭·상대경로 해석 후) 경로가 destDir 하위인지 검증
    if (!out.toPath().startsWith(destDir.toPath())) {
        throw new SecurityException("Zip Slip 시도: " + entry.getName());
    }
    // 추가: 심볼릭 링크 엔트리 거부, 파일 개수·총 크기 상한(zip bomb 방어)
    Files.copy(zip.getInputStream(entry), out.toPath());
}
```

핵심은 문자열 비교가 아니라 canonical 경로(심볼릭 링크와 `..`를 전부 해석한 실제 경로)를 구한 뒤, 그것이 목적지 디렉터리로 시작하는지 확인하는 겁니다. 심볼릭 링크 엔트리를 따라가면 검증을 우회할 수 있으니 링크 엔트리는 거부합니다. 압축 해제는 zip bomb(작은 파일이 거대하게 풀림)도 함께 고려해 엔트리 수·총 해제 크기 상한을 두는 게 안전합니다.

## 정리

- 아카이브 엔트리 이름은 신뢰할 수 없는 입력이다. 경로 조작이 가능하다.
- 해제 경로의 canonical 값이 목적지 하위인지 검증하는 게 표준 방어다.
- 심볼릭 링크 거부와 zip bomb 상한까지 더해야 압축 해제가 안전해진다.

---

*보안 취약점은 방어 목적으로만 다룹니다.*

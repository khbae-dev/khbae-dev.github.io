---
title: 강제 종료 뒤 영원히 남는 잠금 파일, 파일 존재 대신 OS 파일 잠금
date: 2026-10-15 19:24:00 +09:00
categories: [Backend]
tags: [python, file-lock, concurrency, fcntl, windows]
description: 파일이 있는지로 중복 실행을 막던 잠금이 프로세스 강제 종료 후 남아 이후 작업을 모두 막던 문제를, 프로세스가 죽으면 OS가 풀어 주는 fcntl·msvcrt 잠금으로 바꾼 방법을 정리했다.
---

같은 데이터를 두 프로세스가 동시에 보내지 않도록, 송신 도구가 시작할 때 `.lock` 파일을 만들고 끝날 때 지우는 방식으로 막고 있었습니다. 어느 날부터 모든 송신이 "Sender lock exists"로 멈췄습니다.

## 원인: finally는 강제 종료에서 돌지 않는다

```python
@contextmanager
def sender_lock(path):
    lock = Path(f"{path}.lock")
    fd = os.open(lock, os.O_CREAT | os.O_EXCL | os.O_WRONLY)   # 있으면 실패
    try:
        os.close(fd)
        yield
    finally:
        lock.unlink()                                          # 정상 종료 때만 지워진다
```

이 도구는 Jupyter 노트북에서 서브프로세스로 실행됐습니다. 사용자가 노트북의 중단 버튼을 누르면 서브프로세스는 강제 종료되고 `finally`는 실행되지 않습니다. `kill -9`, OOM, 전원 차단도 마찬가지입니다. 남은 파일 하나가 이후 모든 실행을 막았습니다.

"오래된 잠금이면 지운다" 같은 휴리스틱은 고민할수록 함정이 많습니다. PID를 적어 두고 살아 있는지 보는 방식은 PID 재사용과 다른 호스트 문제가 있고, 시간 기준은 오래 도는 정상 작업을 죽입니다.

## 해결: 프로세스가 죽으면 OS가 푸는 잠금

파일의 **존재**가 아니라 파일에 거는 **OS 잠금**을 씁니다. 잠금은 파일 핸들에 붙어 있어서, 프로세스가 어떻게 죽든 OS가 핸들을 닫으면서 잠금도 풉니다.

```python
@contextmanager
def sender_lock(path):
    lock = Path(f"{path}.lock")
    handle = open(lock, "a+b")
    try:
        if os.name == "nt":
            import msvcrt
            handle.seek(0)
            msvcrt.locking(handle.fileno(), msvcrt.LK_NBLCK, 1)   # 첫 1바이트를 비차단으로 잠금
        else:
            import fcntl
            fcntl.flock(handle.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
    except OSError:
        handle.close()
        raise RuntimeError(f"다른 송신이 진행 중입니다 ({lock.name} 잠김).") from None
    try:
        yield
    finally:
        handle.close()   # 두 플랫폼 모두 닫으면 풀린다
```

- `LOCK_NB`/`LK_NBLCK`로 비차단 잠금을 걸어, 이미 잡혀 있으면 기다리지 않고 바로 실패합니다.
- 실제로 돌고 있는 프로세스끼리는 여전히 서로 막습니다.

## 파일은 지우지 않는다

직관과 달리 잠금 파일은 끝나도 **지우지 않습니다.** 지우면 이런 경합이 생깁니다.

1. A가 `x.lock`을 열고 잠근다.
2. B가 같은 파일을 연다. 아직 잠그기 직전이다.
3. A가 끝나면서 잠금을 풀고 파일을 지운다.
4. B가 옛 파일 핸들로 잠금을 얻는다. 지워진 파일이지만 핸들은 살아 있다.
5. C가 새로 생긴 `x.lock`을 만들어 잠근다. B와 C가 동시에 돈다.

파일은 그냥 잠금을 걸 자리로 남겨 두면 됩니다.

## 주의할 점

- `flock`은 NFS 같은 네트워크 파일시스템에서 동작이 보장되지 않습니다. 로컬 디스크에서 씁니다.
- Windows의 `msvcrt.locking`은 바이트 범위 잠금이라 같은 위치(0번 바이트)를 잠그도록 `seek(0)`을 먼저 합니다.
- 테스트는 "잠금을 쥔 서브프로세스를 강제 종료한 뒤 다시 잠글 수 있는가"를 직접 재현해서 확인합니다.

## 정리

- 파일 존재로 만든 잠금은 강제 종료 한 번에 영구 잠금이 된다.
- `fcntl.flock`/`msvcrt.locking`처럼 핸들에 붙는 OS 잠금을 쓰면 프로세스가 죽을 때 자동으로 풀린다.
- 잠금 파일은 지우지 않는다. 지우는 순간 두 프로세스가 서로 다른 파일을 잠그는 경합이 생긴다.

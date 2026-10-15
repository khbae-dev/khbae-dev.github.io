---
title: AES-GCM 복호화가 "Tag mismatch"로만 실패할 때, 원인을 좁히는 오류 메시지
date: 2026-10-15 19:30:00 +09:00
categories: [Security]
tags: [cryptography, aes-gcm, base64, kotlin, troubleshooting]
description: 외부에서 받은 AES-GCM 암호문 복호화가 실패할 때 키 인코딩·nonce 길이·base64 손상·AAD 중 무엇이 원인인지 구분할 수 있도록 단계별 검증과 오류 메시지를 설계한 과정을 정리했다.
---

외부 시스템이 메시지 큐로 보내는 암호화 메시지를 받아 복호화하는 기능을 만들었습니다. 형식은 단순했습니다.

```json
{ "nonce": "base64...", "ciphertext": "base64..." }
```

AES-GCM, nonce 12바이트, 인증 태그 16바이트는 ciphertext 끝에 붙어 있는 형태입니다. 그런데 첫 연동에서 돌아온 건 `AEADBadTagException: Tag mismatch` 한 줄이었습니다.

## Tag mismatch는 거의 아무것도 알려 주지 않는다

GCM은 인증 암호라서, 무엇이든 하나라도 틀리면 똑같이 "태그 불일치"로 실패합니다. 키가 틀려도, nonce가 틀려도, 암호문이 한 바이트 깨져도, AAD가 달라도 같은 예외입니다. 그래서 복호화를 시도하기 **전에** 확인할 수 있는 것은 전부 미리 확인해, 실패 원인을 좁혀 주는 것이 핵심이었습니다.

## 1. 키 인코딩은 추측하지 않는다

같은 32글자 문자열이라도 base64로 풀면 24바이트, hex로 풀면 16바이트, 문자열 그대로 쓰면 32바이트입니다. 셋 다 유효한 AES 키 길이라서 **틀린 방식으로 풀어도 오류 없이 다른 키가 됩니다.** "base64처럼 생겼으면 base64로" 같은 자동 판별은 이 상황을 더 헷갈리게 만듭니다.

```kotlin
fun parseKey(text: String, encoding: String = "base64"): ByteArray {
    val bytes = when (encoding) {
        "base64" -> Base64.getDecoder().decode(text.trim())
        "hex"    -> hexToBytes(text.trim())
        "text"   -> text.toByteArray(Charsets.UTF_8)
        else     -> throw IllegalArgumentException("unsupported key encoding: $encoding (base64, hex, text)")
    }
    require(bytes.size in setOf(16, 24, 32)) { "key must be 16/24/32 bytes but ${bytes.size} bytes ($encoding)" }
    return bytes
}
```

인코딩은 설정으로 명시하게 하고, 디코딩한 길이를 오류 메시지에 함께 보여 줍니다.

## 2. 길이로 알 수 있는 것은 먼저 본다

```kotlin
require(nonce.size == 12) { "nonce must be 12 bytes but ${nonce.size} bytes. nonce must be base64." }
require(cipherText.size > 16) { "ciphertext is too short (${cipherText.size} bytes). it must include 16 bytes auth tag." }
```

연동 초기에 상대가 nonce를 hex로 보낸 적이 있었습니다. 24글자 hex를 base64로 풀면 18바이트가 나옵니다. 이 검사가 없으면 Tag mismatch로 끝났겠지만, 이제는 "nonce가 18바이트"라는 메시지가 바로 인코딩 문제를 가리킵니다. 처음에는 hex도 받아 주게 만들었다가, 사양이 base64로 확정된 뒤 자동 판별을 걷어내고 base64로 고정했습니다. 받아 주는 형식이 많을수록 잘못된 입력이 조용히 통과할 여지도 커집니다.

## 3. base64 손상은 손상이라고 말한다

메시지가 중간 시스템을 거치며 줄바꿈이 끼거나 잘려서 오는 경우도 있었습니다. 공백과 줄바꿈은 걷어내고, 그래도 디코딩이 안 되면 원인 추적에 필요한 정보를 붙입니다.

```kotlin
private fun decodeField(field: String, text: String): ByteArray {
    val t = text.replace(Regex("\\s+"), "")
    return try {
        if ('-' in t || '_' in t) Base64.getUrlDecoder().decode(t) else Base64.getDecoder().decode(t)
    } catch (e: IllegalArgumentException) {
        throw IllegalArgumentException(
            "$field is not valid base64 (length: ${t.length}, length % 4: ${t.length % 4}, " +
            "head: ${t.take(16)}, tail: ${t.takeLast(16)}). value may be truncated or corrupted.", e)
    }
}
```

`length % 4`가 0이 아니면 잘렸을 가능성이 크고, 앞뒤 16글자를 보내는 쪽 로그와 맞춰 보면 어디서 깨졌는지 알 수 있습니다. 앞뒤 일부는 암호문이라 노출돼도 키 없이는 의미가 없지만, **키는 어떤 오류 메시지에도 넣지 않습니다.**

## 4. 그래도 Tag mismatch라면

여기까지 통과했다면 남은 원인은 키 값 자체, 알고리즘(AES-GCM과 ChaCha20-Poly1305 혼동), AAD 불일치뿐입니다. 메시지에 그 목록을 적어 줍니다.

```kotlin
catch (e: AEADBadTagException) {
    throw IllegalStateException("decrypt failed (Tag mismatch). check key, key encoding, cipher($cipherName), aad.", e)
}
```

## 덤: 연동 테스트용 메시지 생성기

상대 시스템 없이도 확인할 수 있도록, 같은 키로 평문을 암호화해 같은 형식의 메시지를 만드는 작은 생성기를 테스트 코드에 두었습니다. 내가 만든 메시지는 풀리는데 상대 메시지는 안 풀린다면, 문제는 내 복호화 코드가 아니라 키나 사양 쪽이라는 것이 바로 드러납니다.

## 정리

- AEAD의 Tag mismatch는 원인을 알려 주지 않는다. 복호화 전에 확인할 수 있는 것은 전부 먼저 검증한다.
- 키와 필드의 인코딩은 추측하지 말고 명시하게 한다. 잘못 풀어도 유효한 길이가 나올 수 있다.
- 오류 메시지에는 길이, `length % 4`, 앞뒤 일부처럼 원인을 좁히는 정보를 넣되 키는 절대 넣지 않는다.
- 사양이 확정되면 임시로 받아 주던 형식은 걷어낸다.

---
title: fallback은 왜 위험한가? 작은 로그 한 줄이 알려준 보안 이야기
date: 2025-03-27 21:00:00 +0900
categories: [Tech, Security]
tags: [보안, 인증, tls, 개발자성장, 실무기록]
description: 로그에서 확인된 fallback으로 해당 문제점에 대해 알아봅시다.
pin: true # make a pin
---
> 로그에서 확인된 fallback으로 해당 문제점에 대해 알아봅시다.
{: .prompt-info }

## 들어가며

사실 처음에는 별다른 생각이 없었습니다. 단순히 로그에 기록된 디버그 메시지 하나쯤으로 여겼습니다.
로그에는 `Challenge for Negotiate authentication scheme not available`라는 메시지가 남아 있었습니다.
`Challenge`라는 용어 자체는 알고 있었지만, 실제 인증 흐름 속에서 마주친 것은 이번이 처음이었습니다.

```
[클라이언트] ── 요청 ──────────────▶ [서버]
[클라이언트] ◀─ 401 Unauthorized, WWW-Authenticate: Negotiate ─ [서버] (Challenge)
[클라이언트] ── Authorization: (인증정보) ───▶ [서버]
[클라이언트] ◀─ 200 OK ────────────── [서버] (인증 성공)
```
서버가 클라이언트에게 특정 인증 방식을 요구할 때 사용하는 것이 바로 `WWW-Authenticate` 헤더입니다.
이번에는 이 `Challenge` 과정 자체가 제대로 동작하지 않았던 것입니다.

## 인증 흐름 분석

- 서버는 SPNEGO 기반의 Negotiate 인증을 요구합니다.
- 클라이언트는 Kerberos와 NTLM 인증, 그 외 인증방법을 순차적으로 시도합니다.
- 이 인증 방식들이 모두 실패하면, 최종적으로 Basic Auth로 전환합니다.

## 인증 정보 전송 구조와 보안 리스크

클라이언트에서 사용하는 username/password는  
프로퍼티 파일 내에서 **암호화 라이브러리를 사용해 암호화된 형태로 저장되고 있었습니다.**  
저장 시점에서는 평문 노출 없이 안전하게 관리되고 있었던 셈입니다.

다만 애플리케이션 실행 시 이 값은 복호화되어 사용되고,  
Basic Auth 방식으로 요청을 보낼 때는  
base64로 인코딩된 형태로 `Authorization` 헤더에 포함되어 전송됩니다.

이러한 동작은 단순한 구현 관행이 아닌, 
[HTTP Basic Authentication 공식 명세 (RFC 7617)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Authentication) 에 명시된 동작입니다. 
따라서 대부분의 언어나 HTTP 라이브러리가 기본적으로 이 방식으로 동작합니다.

문제는 **TLS가 적용되지 않은 상태라면 이 값은 네트워크 상에서 평문처럼 노출될 수 있는 구조**이고,  
특히 네트워크 장비나 클라이언트가 **Promiscuous 모드로 동작하고 있다면**,  
자신을 목적지로 하지 않는 패킷이라도 쉽게 포착하여 인증 정보를 탈취할 수 있다.

이 fallback이 단순한 예외 흐름이 아니라, **실제 보안 리스크로 이어질 수 있는 가능성**을 내포합니다.

여기서 문득 얼마 전 공부했던 POODLE 공격이 떠올랐다.
POODLE 공격도 결국은 보안성이 낮은 SSL v3로 fallback이 일어나면서 발생한 공격이었다.
'fallback'이라는 간단한 동작이 보안에 얼마나 치명적일 수 있는지, 실제 사례를 통해 다시 한번 확인할 수 있었다.

## 🧪 HTTP Basic Auth 인증정보는 네트워크에서 어떻게 보일까?

평문 HTTP 통신에서 Basic Authentication을 사용하면  
인증 정보가 어떻게 전송되고 노출되는지 직접 실험해 보았습니다.

---

### 🔧 실험 구성

- 테스트용 FastAPI 서버 (Gunicorn + HTTP, port 8000)
- 로컬 PC에서 인증 헤더를 포함한 요청 전송
- Wireshark로 요청 패킷 실시간 감시
- curl을 통한 명시적 Basic 인증 헤더 삽입

요청 명령어:

```bash
curl -H "Authorization: Basic dGVzdHVzZXI6dGVzdHBhc3M=" http://SERVER_PUBLIC_IP:8000
```

---

### 🔎 Wireshark 캡처 결과
![wireshark-result](/assets/img/posts/250327.wireShark.png)

요청 패킷에 포함된 `Authorization: Basic ...` 헤더가
Wireshark 상에서 자동 디코딩되어 `Credentials: testuser:testpass` 로 확인됩니다.

즉, 평문 HTTP 환경에서는 네트워크를 감청하면 인증정보가 그대로 노출될 수 있습니다.

### 🧪 실험 결과 요약
Basic 인증 헤더는 Base64 인코딩되어 전송되지만 암호화된 것이 아닙니다.

내부망/공용망 관계없이 HTTP는 인증 정보 탈취 위험이 존재합니다.


✅ 이번 실험을 통해 “Basic 인증은 반드시 HTTPS 위에서만 사용해야 한다”는 보안 원칙의 중요성을 실감했습니다.

## 마무리하며

처음엔 별것 아니라고 생각했던 간단한 로그 메시지에서 시작했지만,
직접 흐름을 추적해 보면서 인증 구조와 그 이면의 보안적 문제를 명확히 알 수 있었다.

그동안 책과 문서로만 이해했던 Challenge 개념이 이번 경험을 통해 비로소 실제 흐름과 연결되었다.

사실 보안이라는 분야는 개발자로서 '얼마나 깊이 알아야 할까?' 하는 고민도 있었다.
그러나 애초에 보안 공부를 시작한 건 IT 전반에 대한 더 넓고 깊은 이해를 원했기 때문이었다.

직접 로그를 분석하고 인증 흐름을 추적하면서,

>“TLS가 왜 필수인지”\
>“Fallback이 어떻게 보안 문제를 만들어내는지”

를 명확히 깨닫는 순간, 그동안의 공부가 헛되지 않았음을 느꼈다.

꼭 보안 전문가가 되지 않더라도,
이렇게 실무 경험을 글로 기록하고 정리하는 과정 자체가
개발자로서 성장하는 제 자신에게 매우 의미 있는 한 걸음이라 생각합니다.

---

이 글은 제가 실제 경험한 내용을 바탕으로 작성했습니다.
혹시 다른 시각이나 보완할 점이 있다면 언제든 편하게 알려주시기 바랍니다.
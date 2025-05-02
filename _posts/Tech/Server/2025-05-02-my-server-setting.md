---
title: 나의 Oracle Cloud Infrastructure 서버 구축기록 - 1
date: 2025-05-02 20:00:00 +0900
categories: [Tech, Server]
tags: [인프라, 개발자성장, 개인서버, ocr, cloudflare, zero trust]
description: Oracle Cloud를 이용해 도메인을 연결하고, HTTPS부터 Cloudflare 인증 게이트까지 직접 구성한 과정 정리
mermaid: true
pin: true # make a pin
---
> Oracle Cloud를 이용해 도메인을 연결하고, HTTPS부터 Cloudflare 인증 게이트까지 직접 구성한 과정을 정리해보았습니다.
{: .prompt-info }

## 시작하기 전에

도메인을 연결해서 내 클라우드 서버를 구축해봐야겠다는 생각은 전부터 있었다.  
그렇지만 도메인 비용도 있고,  
서버 유지비용까지 생각하면 선뜻 손대기 어려웠다.

그러다가 **Oracle Cloud**에서 제공하는 **Always Free** 인스턴스가 있다는 얘기를 듣고,  
일단 하나 만들어보게 됐다.  

[OCI Always Free 정보](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)

기존에 작성한 블로그글에서도,  
basic auth 실험을 위해 잠깐 써보긴 했지만  
**이번에는 아예 내 서버를 한 번 직접 구성해보자**는 생각이 들었다.  

보안 관련 공부와 인프라 실험을 병행할 수 있는 서버,  
즉 내가 직접 통제하며 마음껏 실험할 수 있는 안전한 실습 공간을 구축해보고 싶었다.  

## 내가 구성한 서버 구조

단순히 FastAPI 앱을 띄우는 수준이 아니라,  
**도메인 연결부터 HTTPS 구성, 인증 게이트 적용까지**  
실서비스 못지않은 구조로 직접 구성했다.

전체적으로는 다음과 같은 흐름이다.

| 항목 | 구성 내용 |
|:---|:---|
| **도메인** | `inye.cloud` 도메인 구입 후(1년 3,300원으로 이벤트 중인 것을 구매했다. 호스팅케이알), Cloudflare 네임서버를 호스팅케이알에 등록 |
| **서버 인프라** | Oracle Cloud에서 제공하는 **Always Free VM** 인스턴스 사용 |
| **애플리케이션** | FastAPI + Gunicorn으로 서버 실행, systemd로 등록해 재부팅 시 자동 시작 |
| **Reverse Proxy** | Nginx를 이용해 외부 요청을 내부 FastAPI로 프록시 처리 |
| **HTTPS 구성** | Let’s Encrypt 인증서 발급 + `certbot.timer`로 자동 갱신 설정 |
| **TLS 정책** | TLS 1.3 only 적용, Cloudflare와 Origin 간에도 HTTPS가 유지되는 **Full (strict) 구성** |
| **HTTP 리디렉션** | 80번 포트를 제한적으로 열어, 모든 HTTP 요청을 HTTPS로 301 리디렉션 |
| **보안 헤더** | HSTS(long duration + preload) 적용, 브라우저 수준에서 HTTPS 강제 |
| **방화벽** | iptables 및 클라우드 ingress rule을 함께 구성해 이중 방화벽 적용 |
| **Cloudflare 연동** | Cloudflare proxy를 통해 IP 비공개 |
| **Zero Trust 인증** | Cloudflare > Zero Trust > Access를 통해 **OTP 기반 이메일 로그인 제한** 적용, Policy에 등록된 이메일만 로그인 시도 가능|

[HTTP Strict-Transport-Security(HSTS)에 대해 알아보기](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security)

지금 상태만 보면 그냥 이메일 인증 후 `"hello world"`가 보이는 루트 페이지 하나뿐이지만,  
**이 구조 위에 어떤 기능이든 안전하게 얹을 수 있는 기반은 모두 갖춰졌다.**


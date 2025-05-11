---
title: 나의 Oracle Cloud Infrastructure 서버 구축기록 - 3
date: 2025-05-11 20:00:00 +0900
categories: [Tech, Server]
tags: [인프라, 개발자성장, 개인서버, oci, cloudflare, zero trust]
description: Cloudflare Zero Trust Access를 통한 OTP 인증 게이트 구성기
pin: false # make a pin
---
> Cloudflare Zero Trust Access를 통한 OTP 인증 게이트 구성기
{: .prompt-info }

## 시작하며

이전 글에서는 Nginx 리버스 프록시 설정과 HTTPS 기반 통신,  
그리고 클라우드 및 OS 방화벽 설정 과정을 다뤘다.

이번에는 Cloudflare Access를 통해  
**OTP 기반 이메일 인증 게이트**를 구성한 실습기를 정리한다. 
이 과정에서 마주했던 **SSL 모드 설정에 따른 리디렉션 문제**,  
그리고 `www.inye.cloud`와 `inye.cloud` 간 인증 정책 적용 차이도 함께 다룬다.  

## Cloudflare Access란?

Cloudflare Access는 Zero Trust 보안 모델의 일부로,  
특정 사용자나 이메일 도메인에 따라 웹 애플리케이션 접근을 제어할 수 있는 서비스다.  
기본적으로 OTP 기반의 이메일 인증, SSO 연동, IP 기반 제한 등을 설정할 수 있다.  

### 처음 내 인증 게이트 구성 목표

1. `inye.cloud`, `www.inye.cloud` 접속 시 이메일 인증 요구
2. 허용된 이메일만 접근 가능하게 구성

## 구성 과정

### 1. SSL 모드 설정 변경 이슈

초기에는 Cloudflare의 SSL 모드가 Flexible로 설정되어 있었는데,  
이 경우 **인증 게이트에서 무한 리디렉션 현상이 발생**했다.  

> → 이후 Full(strict) 모드로 변경하여 문제를 해결했다.  
> 💡 Full(strict): 클라이언트와 Cloudflare 간,  
> Cloudflare와 원본 서버 간 모두 HTTPS이며,  
> 원본 서버는 유효한 인증서를 가지고 있어야 한다.

### 2. OTP 인증 정책 적용

Cloudflare > Zero Trust > Access > Applications 메뉴에서 다음과 같이 구성했다:

1. www.inye.cloud를 애플리케이션으로 등록  
2. 이메일 정책에 개인 이메일(naver, gmail 등)을 넣음  
3. OTP 코드가 메일로 전송되며, 인증된 사용자만 접근 가능

![Zero Trust](/assets/img/posts/250511.cloudflareZeroTrust.png)  
![Access Policy](/assets/img/posts/250511.cloudflarePolicy.png)

### 3. 인증 정책이 도메인별로 다르게 작동한 이유

Cloudflare Access에서는 `www.inye.cloud`와 `inye.cloud`를 **별도의 Application**으로 인식한다.  
초기 설정 시 `www.inye.cloud`에는 인증 정책이 적용되지 않아 누구나 접속 가능했고,  
반면 `inye.cloud`에서는 이메일 인증을 요구했다.

사실 Cloudflare 인증 게이트를 설정하기 전에는,  
같은 웹페이지에 대해 `www.inye.cloud`와 `inye.cloud` 두 도메인 모두에서 접속이 가능하도록 허용해둔 상태였다.  

그러나 인증 정책뿐 아니라 **서비스 운영과 관리 측면**에서도  
**하나의 기준 도메인으로 통합하는 것이 유지 보수와 정책 적용에 훨씬 유리하다고 판단**했다.

특히 Cloudflare Access의 인증 정책이 **도메인 단위로 개별 적용**되기 때문에  
정책 설정의 중복을 피하고자 `www.inye.cloud`를 기준 도메인으로 삼았고,  
결과적으로는 모든 `inye.cloud` 요청을 `www.inye.cloud`로 리디렉션하도록 구성하였다.  

```nginx
# 리디렉션 전용 (inye.cloud → www.inye.cloud)
server {
    listen 443 ssl;
    server_name inye.cloud;

    ssl_certificate /etc/letsencrypt/live/inye.cloud/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/inye.cloud/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    return 301 https://www.inye.cloud$request_uri;
}
```

> 등록되지 않은 이메일을 입력해도, 인증게이트에서는 "등록되지 않은 이메일"임을  알려주지 않는다. (Cloudflare의 정책)

### 마무리하며

Cloudflare Access를 이용해 인증 게이트를 구성하면서,  
실제 운영 환경에서 도메인 단위로 인증 정책을 어떻게 설계하고 적용할 수 있는지 체험해볼 수 있었다.  

또한 보안 실험을 확장하기 위해 `inye-space-two`라는 별도의 VM 인스턴스도 구성하였다.  
해당 서버는 **DB 전용 서버**로 설정하여, 애플리케이션 서버와 분리된 데이터 계층 구조를 실험해볼 예정이다.  

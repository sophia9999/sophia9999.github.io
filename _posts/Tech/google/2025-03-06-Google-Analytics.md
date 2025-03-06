---
title: Google Analytics, 내 블로그에 적용해보기
date: 2025-03-06 14:00:00 +0900
categories: [Tech, Google]
tags: [구글,애널리틱스,google,google analytics]
description: [구글 애널리틱스 적용해보기 applying google analytics to your app]     
pin: true
---

> Google Analytics에 대해 알아본다. 설정을 진행해본다.
{: .prompt-info }

### 구글 애널리틱스 Google Analytics 는 무엇인가 
<https://developers.google.com/analytics?hl=ko>

간단하게 말하자면 **웹사이트나 모바일 앱의 방문자 데이터를 수집하고 분석할 수 있는 도구**입니다. 

Google Analytics 계정 생성 후 아래 스크린 샷을 따라 참고하되 자세한 체크박스는 개개인에 따라 다를 수 있습니다.

1. 계정생성\
![계정생성](assets/img/posts/250306.createAccount1.jpg)
2. 속성생성\
![속성생성](assets/img/posts/250306.createAccount2.jpg)
3. 비즈니스설정\
![비즈니스설정](assets/img/posts/250306.createAccount3.jpg)
4. 국가설정\
![국가설정](assets/img/posts/250306.createAccount4.jpg)
5. 플랫폼설정\
![플랫폼설정](assets/img/posts/250306.createAccount5.jpg)
저는 웹 플랫폼을 눌렀습니다.
6. 데이터 스트림 설정\
![데이터스트림설정](assets/img/posts/250306.setting1.jpg)
![대쉬보드추가되었음](assets/img/posts/250306.setting2.jpg)
그러면 대쉬보드에 제가 추가한 my-app 이 추가됩니다.\
추가된 row를 선택하면 웹 스트림 세부정보가 나옵니다.\
![대쉬보드추가되었음](assets/img/posts/250306.setting3.jpg)
7. 측정ID를 config.yml 의 해당하는 field에 넣어줍니다. (제가 쓰는 jekyll 블로그 테마의 config파일에서는 이미 있었네요~)\
![데이터스트림 config](assets/img/posts/250306.setting4.jpg)
이 후 push하면 Git action으로 page가 deploy 됩니다. 
8. 연필 버튼을 눌러 URL과 이름을 입력하고 스트림을 업데이트 해줍니다.
![데이터스트림 add](assets/img/posts/250306.setting5.jpg)
그럼 방문자의 데이터가 수집됩니다. 


확인은 대쉬보드 - 실시간 개요 쪽으로 가보면 통계를 확인할 수 있습니다.\
![설정 완료](assets/img/posts/250306.complete.jpg) 
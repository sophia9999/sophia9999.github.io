---
title: Hello, World!
date: 2025-03-06 00:00:00 +0900
categories: [Tech]
tags: [github.io, setting]
description: github.io 나만의 블로그 세팅하기
---

# github.io 나의 블로그 만들기

마음에 드는 jekyll template 찾기 
- <https://jekyllthemes.io/free>

Jekyll는 GitHub Pages를 기본적으로 지원하는 정적 사이트 생성기입니다. 자세한 사항은 ->[link](https://docs.github.com/ko/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll)

마음에드는 template을 찾았으면 github로 가서 확인한다.
제가 선택한 템플릿의 경우 starter를 통해 지원해주고 있었다. [link](https://chirpy.cotes.page/posts/getting-started/)

해당 포스팅에서 아주 친절히.. 다 알려주신데로 하면된다. 
자세한 사항은 해당 포스팅을 참고한다는 가정하에 아래에 기재하겠습니다.

### for the detail...

1. Windows를 사용 중인데, wsl이 개발 환경에 매우 우용하므로 cmd에서 wsl --install 을 통해 ubuntu를 설치
2. ubuntu계정에서 Docker Engine을 다운로드 <https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository> VScode의 Extention에서 Dev Container를 설치한다.
![docker](assets/img/posts/250306.docker.jpg)
3. Dev Container를 통해 프로젝트를 열어줍니다.
![devContainer](assets/img/posts/250306.devCon.jpg)
4. 완료 후 아래 커맨트 실행
```SHELL
$ bundle exec jekyll s
```

local에서 실행되는 것이 확인되고, _config.yml에 본인이 필요한 설정들을 넣어주면 됩니다. 

### 주의사항
블로그 포스팅 글 올리실 때 미래 시간이면 보이지 않습니다. (날짜를 착각하여 미래시간으로 해놓고 2시간 넘게 왜 안나오지 뻘짓해서...) 
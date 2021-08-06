---
layout: post
title: Entry Point of copyrat90::devlog
tags: [Blog]
author: copyrat90
last_modified_at: 2021-08-06T18:47:00+09:00
---


## 새로운 시작

GitHub Pages 를 이용해 기술 블로그를 열었다.

사실, 개발 공부를 하면서 배운 것들을 글로 써서 모아둬야겠다는 생각 자체는 자주 했었다.\
다만 내 **요구 사항**에 맞는 적절한 플랫폼을 찾지 못해 생각에만 그쳤었다.

> #### 요구 사항
> 3. 어두운 테마를 지원할 것.
> 1. Code snippet 을 입력하면, 자동으로 Syntax Highlighting 이 될 것.
> 2. Markdown 글 작성을 지원할 것.
> 4. Tag 기능을 지원할 것.
> 5. 군더더기 없이 간결할 것.

여러 플랫폼을 찾아 헤맸지만, 역시 `2. Code Highlighting` 이랑 `3. Markdown 글 작성 지원`하는 플랫폼이 너무 적었다.\
하기야, 개발자가 아니면 쓸 일이 없는 기능이니.

[velog](https://velog.io/) 가 제일 요구 사항에 근접했으나, `1. 어두운 테마` 미지원.\
난 어두운 테마 미지원 사이트를 무척 싫어한다.\
내 PC Chrome에는 CSS 테마를 강제로 어둡게 변환하는 플러그인까지 깔려 있다.\
하지만 모바일에서는 플러그인 설치도 못하기 때문에, 밝은 사이트는 웬만하면 PC 로 접속한다.

게다가 velog 는 개인이 운영하는 사이트라 지속 운영이 될지 여부도 불투명하다.\
사실, 한국에서 기술 블로그 서비스를 하는 곳이 저기뿐이라 금방 문 닫을 것 같지는 않지만.

그런 이유로 **GitHub Pages** 로 눈을 돌렸다.



## GitHub Pages

[GitHub Pages](https://pages.github.com/) 는 GitHub 에서 제공하는 정적 웹사이트 호스팅 플랫폼이다.

사용법은 간단하다.\
아이디가 *username* 이면, GitHub 에 이름이 *username*.github.io 인 Repository 를 하나 만들고, 최상단 디렉토리에 index.html 을 포함하면 끝이다.\
그러면 알아서 https://*username*.github.io 페이지에 호스팅이 시작된다.



## Jekyll

물론 이대로면 Markdown 지원이고 자시고 그냥 본인이 웹개발을 해야하니,
[Jekyll](https://jekyllrb.com/) 이라는 [Ruby](https://www.ruby-lang.org/en/) 기반 웹사이트 생성기를 이용한다.\
[Jekyll](https://jekyllrb.com/) 을 이용하면, Markdown 으로 작성한 문서가 정적 웹페이지로 자동 변환된다.

다만 처음 세팅하기가 상당히 귀찮다.\
`나도 GitHub Pages 블로그 운영해보고 싶다!` 하는 분들은 [이 블로그](https://devinlife.com/howto/)를 천천히 따라해보는 걸 추천한다.

난 세팅은 다 해놨으므로 (하루 종일 걸림...) 글 작성하는 법만 여기다 적어놔야겠다.

> #### 글 작성법
> Local 에서 Markdown 파일로 글을 작성한 후, `bundle exec jekyll serve` 을 돌리면 웹서버가 실행된다.\
> 이 상태에서 `127.0.0.1:4000` 에 접속하면 내용을 확인할 수가 있다.\
> `_config.yml` 파일 외에 모든 파일은 웹서버가 작동중인 상태에서도 수정하면 결과가 즉각 반영되니,\
> `Markdown 수정 => 새로고침` 을 반복하면서 글을 작성하면 된다.
>
> 글 작성이 완료됐으면 그냥 `commit` 을 하나 만들어 `push` 해주면 GitHub Pages 가 알아서 페이지를 업데이트해준다.

> `push` 하고 반영되기까지 1분정도 걸린다.

근데 Jekyll 기본 테마, 밝은 테마다...\
어두운 테마이면서 내가 원하는 기능을 지원하는 테마를 찾아다니느라 한세월.\
그걸 내 블로그에 적용하고 적절하게 세팅하느라 또 한세월.\
생각보다 손댈 게 많았다...

뭐 열심히 세팅했으니, 글 한줄이라도 더 쓰겠지.


_마지막 수정 : {{ page.last_modified_at }}_

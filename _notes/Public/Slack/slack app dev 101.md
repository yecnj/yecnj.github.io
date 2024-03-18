---
title: slack app dev 101
feed: hide
date: 23-11-2021
permalink: /slack-app-dev-101
---
#### 목적
- 슬랙 문서 여기저기에 산재된 슬랙 앱 개발 방법 사용해보기
	- [Next generation Slack platform](https://api.slack.com/start/overview#next-gen-platform) 살펴보기
	- [Bolt for Python](https://api.slack.com/start/building/bolt-python)  살펴보기
- 슬랙 앱을 생성해서 오픈소스 서비스에 토큰을 주거나 Slack REST API를 찔러서 메세지를 보내는 등의 작업은 많이 해봤지만, 직접 개발을 시도한 적은 없다.
- 어떤 아키텍쳐로 되어있는지 직접 확인해보고 싶다.

&nbsp;

#### [next-gen slack platform](https://api.slack.com/start/overview#next-gen-platform)

아마 작년 말, 올해 초 쯤에 beta를 진행했고 

&nbsp;

##### 슬랙 앱 생성

[슬랙 앱 관리 콘솔 ](https://api.slack.com/apps)

예전부터 슬랙 앱을 관리하는 웹 콘솔에서는 로그인 메뉴가 따로 없는데, 여러 로그인 세션이 같이 있어서 그렇다. **Create New App** 을 누르면 워크스페이스를 선택할 수 있다.

이렇게 생성하면 해당 워크스페이스에 등록된 계정이 작업자(Collaborator)가 된다. 각 App은 애초에 워크스페이스에 종속이기 때문에, 슬랙 계정이 여러 개라고 해서 어느 계정에 생성되는지 걱정할 필요는 없다. 불안하다면 시크릿 모드를 켜고 작업하자.
#### [[slack app scope]] 설정

**Bot Token Scope** 에 `chat:write` 권한을 추가한다.

&nbsp;
##### 


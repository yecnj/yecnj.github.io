---
title: slack app scope
feed: hide
date: 23-11-2023
permalink: /slack-app-scope
---
슬랙 앱이 할 수 있는 권한의 나열.

[슬랙 앱 관리 콘솔 ](https://api.slack.com/apps) 에서 각 슬랙 앱을 들어가면 App 메인페이지가 나오고 여기에서 **Features** - **OAuth & Permissions** 탭을 누르고 스크롤을 내려보면 **Scope** 라는 기능에서 설정 가능하다.

봇 계정으로 작업하게 되는 **Bot Token Scope**와 유저를 대신해서 작업하게 되는 **User Token Scope**가 있다.

일반적인 케이스에서는 Bot Token Scope을 사용하고, 작업자의 Status 변경 같은 특수한 케이스에만 User Token Scope를 사용하게 된다.

봇 토큰 / 권한을 잘못 관리했다가는 사칭당할 수 있으니 가능하다면 Bot Token Scope를 사용하자.
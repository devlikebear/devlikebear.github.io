---
layout: post
title: "Jekyll과 Github Page로 블로그 구축하기"
categories: jekyll github blog
---

## 1. Github에 저장소 만들기

### repository 생성

github에 "{username}.github.io"의 이름으로 public repository를 생성합니다.

예를 들어 username이 "foo"인 경우, 다음의 이름으로 생성합니다.

`foo.github.io`

### 로컬PC에 Repository Clone하기

로컬PC에서 작업디렉토리로 이동한 후 다음과 같이 Repository를 Clone합니다.

작업디렉토리가 `/Users/foo/work`인 경우 다음과 같이 진행합니다.

```bash
cd ~/work
git clone https://github.com/foo/foo.github.io
```

## 2. Jekyll 사이트 생성

### jekyll 설치

jekyll과 bundler를 설치합니다.

```bash
gem install jekyll
gem install jeykyll bundler
```

### 사이트 생성

jekyll로 사이트를 생성합니다.

```bash
cd ~/work/foo.github.io
jekyll new ./
```

### 번들 설치

bundler를 사용해 번들을 설치합니다.

```bash
bundle install
```

### 로컬PC에서 사이트 구동

bundler를 사용해 사이트를 구동합니다.

```bash
bundle exec jekyll serve
```

문서를 작성하면 사이트를 자동으로 새로고침(hot-reload)하므로, 문서를 수정하고 매번 브라우저를 새로고침할 필요가 없습니다.

단, \_config.yml을 수정한 이후에는 서버를 종료하고 재시작해야 합니다.

브라우저에서 `http://127.0.0.1:4000/`주소로 접속해서 사이트를 확인할 수 있습니다.

## 3. Github Page에 변경된 사이트 출판하기

사이트가 변경될 경우 github의 main브랜치에 push하는 것만으로 사이트를 출판할 수 있습니다.

```bash
git add .
git commit -m "커밋 메세지"
git push
```

# EK Dev Blog

## 목차

1. [사전준비](#사전준비)

- 1. [git 설치](#1-git-설치)
- 2. [golang 설치](#2-golang-설치)
- 3. [hugo 설치](#3-hugo-설치-homebrew로-설치)

2. [로컬호스트로 테스트하기](#로컬호스트로-테스트하기)
3. [PaperMod 테마를 적용한 깃헙블로그 초기환경 셋팅](#papermod-테마를-적용한-깃헙블로그-초기환경-셋팅)

---

## 사전준비

- 사용 테마: [hugo-PaperMod](https://github.com/adityatelange/hugo-PaperMod/)

- 본인 OS환경: Mac OS (Linux Terminal)
- **[homebrew](https://brew.sh/) 를 반드시 설치해주세요.**

### 1. git 설치

```shell
$ brew install git
```

### 2. golang 설치

- [Download go](https://go.dev/dl/) 링크에 접속해서 OS 사양에 맞게 go 를 설치합니다.

### 3. hugo 설치 (`homebrew`로 설치)

- 최소 Hugo Version: `v0.125.7`
- hugo는 golang 으로 만들어진 오픈소스 정적 웹사이트 생성자 입니다.
- 참고: [Hugo Installation](https://gohugo.io/installation/)

```shell
$ brew install hugo
$ hugo version
```

## 로컬호스트로 테스트하기

> [ 🖐️ 주의 ] <br><br>
> homebrew, git, golang, hugo 가 설치되어야 하므로
> "[사전준비](#사전준비)" 단계에서 설치를 미리하시길 권장드립니다.

```shell
$ hugo server
```

`localhost:1313/{사이트명}` 주소로 웹사이트를 테스트를 할 수 있습니다. 원격래포에서는 존재하지 않지만, 로컬기준으로 봤을때 래포지토리에서는 `{사이트명}`은 `blog` 이며, blog 디렉토리내에 정적 웹사이트 관련 코드들이 저장되어있습니다. 따라서 셋팅할 때 깃헙래포명과 `{사이트명}`명을 동일하게 하는게 좋습니다.

## 포스트 작성하기

```shell
$ hugo new docs/{포스트-파일명}.md
```

생성된 포스트 마크다운는 `/content/docs` 디렉토리에 생성되며
`/archetypes/default.md` 양식에 맞춰서 포스트파일이 생성됩니다.

> /archetypes/default.md

```md
---
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
date: '{{ time.Now.Format "2006-01-02" }}'
draft: true
---
```

> [ 트러블슈팅 ] Liquid Exception

- 원래는 초기에 `date: {{ .Date }}` 로 되어있습니다. 해당 날짜 포맷팅코드로 인해서 gh-pages 자동화봇의 build 단계에서 실패하여 배포가 실패되는 트러블슈팅을 겪게 되었습니다.

- Error Message

```shell
To use retry middleware with Faraday v2.0+, install `faraday-retry` gem
  Liquid Exception: Invalid Date: '"{{ .Date }}"' is not a valid datetime. in /_layouts/default.html
    ERROR: YOUR SITE COULD NOT BE BUILT:
        ------------------------------------
        Invalid Date: '"{{ .Date }}"' is not a valid datetime.
```

- hugo 에서는 해당날짜 포맷팅이 `%Y-%m-%dT%h:%m:%s%z` 로 현재 날짜 로 인식을 하지만, gh-pages는 ruby 기반의 jekyll 환경에서 실행됩니다. 안타깝게도 jekyll 환경에서는 `{{ .Date }}` 를 현재 날짜로 인식하지 못합니다. [hugo 공식문서 - Date Format](https://gohugo.io/content-management/archetypes/#date-format)을 읽어서 해결했습니다.

<br>

> (예) /content/docs/test.md

```md
---
title: "Test"
date: "2025-03-20T15:38:28+09:00"
draft: false
---

# archetypes/default.md 파일 설명

## title

- title 은 마크다운 파일명을 나타냅니다.
- 만일 생성한 마크다운파일이 test.md 라면 title 은 "Test" 로 나타냅니다.

## date

- 마크다운파일 생성시점 입니다.

## draft

- 해당 포스트가 "작성중" 혹은 "작성완료" 상태인지를 나타냅니다.
- draft: true
  "작성중" 인 상태이며, 웹사이트에 나타나지 않습니다.
- draft: false
  "작성완료" 상태이며, 웹사이트에 반영합니다.
- 게시글을 작성다하고 main에 push하는게 좋습니다.
  만일 작성중인 상태에서 게시글내용만 달라지는 경우에는 public 내부파일 변경에 반영되지 않아 빌드단계에서 실패할 확률이 높습니다.
```

<br>

## PaperMod 테마를 적용한 깃헙블로그 초기환경 셋팅

> [ 🖐️ 주의 ] <br><br>
> homebrew, git, golang, hugo 가 설치되어야 하므로
> "[사전준비](#사전준비)" 단계에서 설치를 미리하시길 권장드립니다.

- 참고자료
  - [PaperMod 탬플릿 공식 자료](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation)
  - [PaperMod 탬플릿 초기환경셋팅 유튜브영상](https://youtu.be/_QSr2_pxIJs?si=EywkWaZms0uykPfC)

### 1. 초기 README.md 없이 빈상태로 github 래포지토리를 생성합니다.

### 2. 래포지토리내에 hugo 사이트를 생성합니다.

- hugo 사이트명은 자유롭게 설정가능하나, 빠른 웹사이트 구축하기 위해서는 생성한 래포지토리명과 동일하게 합니다.

```shell
$ hugo new site {래포지토리명} --format yml
```

- 프로젝트가 생성되면 자동으로, 아래와 같은 디렉토리구조로, 하위디렉토리와 파일들이 생성됩니다. 초기 생성된 archetypes, content, public 디렉토리 안에는 파일들이 들어있습니다.

```shell

📦 ROOT
 ┗ 📂 {래포지토리명}
   ┣ 📂 archetypes
   ┣ 📂 content
   ┣ 📂 public
   ┣ 📂 themes
   ┃
   ┣ 📜.hugo_build.lock
   ┗ 📜hugo.yml
```

### 3. `{래포지토리명}` 으로 이동 후, 현위치를 git 시작점으로 지정합니다.

```shell
$ cd {래포지토리명}
$ git init
```

### 4. 테마래포를 클론 후, submodule로 적용합니다.

- `/{래포지토리명}/themes` 디렉토리 안에 테마원본 래포를 클론하여 넣습니다.

```shell
$ git clone https://github.com/adityatelange/hugo-PaperMod themes/PaperMod --depth=1
```

<br>

- 원본래포를 참조하여, Git SubModule을 적용합니다.

```shell
$ git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

### 5. hugo.yml 에 테마항목을 추가합니다.

```yaml
theme: PaperMod
```

### 6. 웹사이트를 로컬호스트로 테스트해봅니다.

- 웹사이트 로컬호스트 실행 테스트

```shell
$ hugo server
```

- 포스트 작성 테스트

```shell
$ hugo new docs/{포스트제목}.md
```

### 7. 프로젝트 내부 .git 에 원격래포지토리 정보 넣기

> [ 주의 ] <br><br>
>
> - 깃헙블로그 초기환경셋팅은 래포지토리는 래포지토리 owner만이 가능합니다.
> - 이미 github.io 도메인은 계정당 한개의 래포만 적용할 수 있습니다.

```shell
# origin 정의
$ git remote add origin https://github.com/{github-유저명}/{래포지토리명}.git

# 메인브랜치 만들기
$ git branch -M main

# 프로젝트 ROOT 디렉토리에 README.md 파일 만들기
$ echo "# {래포지토리명}" >> README.md
```

### 8. `.gitignore` 파일 생성 후, 원격래포 이력을 제외시킵니다.

- `.gitignore` 파일 생성

```shell
$ touch .gitignore
```

- `.gitignore` 파일 위치

```shell
📦blog
 ┣ 📂.github
 ┃ ┗ 📂workflows
 ┃ ┃ ┗ 📜deploy.yml
 ┣ 📂archetypes
 ┃ ┗ 📜default.md
 ┣ 📂content
 ┃ ┗ 📂docs
 ┣ 📂public
 ┣ 📂themes
 ┃ ┗ 📂PaperMod
 ┣ 📜.gitignore
 ┣ 📜.gitmodules
 ┣ 📜.hugo_build.lock
 ┣ 📜README.md
 ┗ 📜hugo.yml
```

- `.gitignore` 에 커밋반영 제외 파일 및 디렉토리 추가

```yml
# .gitignore

.hugo_build.lock
```

### 9. main 브랜치에 푸시합니다.

```shell
$ git add . && git commit -m "first commit"
$ git push -u origin main
```

### 10. 워크플로우에게 래포지토리 읽기/쓰기 접근권한을 부여합니다.

깃헙래포의 'Settings' > 'Actions' > 'General' > 'Workflow permissions' 으로 이동하게되면,
`Read and write permissions` 선택하고 저장합니다.

<br>

### 11. gh-pages 로 배포프로세스 워크플로우를 작성합니다.

```shell
$ mkdir -p .github/workflows
$ touch .github/workflows/deploy.yml
```

<br>

- main 브랜치에서 푸시이벤트가 발생하면, 워크플로우가 실행됩니다. job은 deploy한개만 존재하며, deploy에서는 5개의 step이 존재하며, 순서대로 실행됩니다.

- step `Setup hugo`
  - 최소 권장 hugo 버젼은 `v0.125.7` 입니다.
  - [github 공식문서](https://docs.github.com/ko/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#default-environment-variables)를 참고하여 `RUNNER_TEMP`는 임시디렉터리이며, 해당 step의 시작과 끝에는 비워집니다. 즉 여기서는 hugo를 설치하고 설치가 완료되면 설치실행파일을 삭제할때 사용됩니다.
- step `Deploy`
  - `GITHUB_WORKSPACE`는 러너의 작업디렉토리를 나타내는 환경변수입니다. 초기에는 빈값이지만 [`actions/checkout`](https://github.com/actions/checkout)작업은 github 래포지토리를 환경변수에 복제합니다. 여기서는 github.io 사이트와 연결된 gh-pages 브랜치가 main브랜치를 복제하여 사이트에 반영을 시키는걸 의미합니다.

```yaml
# deploy.yml

name: Publish to GH Pages
on:
  push:
    branches:
      - main
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        uses: actions/checkout@v3
        with:
          submodules: true

      - name: Checkout destination
        uses: actions/checkout@v3
        if: github.ref == 'refs/heads/main'
        with:
          ref: gh-pages
          path: built-site

      - name: Setup Hugo
        run: |
          curl -L -o /tmp/hugo.tar.gz 'https://github.com/gohugoio/hugo/releases/download/v0.125.7/hugo_extended_0.125.7_linux-amd64.tar.gz'
          tar -C ${RUNNER_TEMP} -zxvf /tmp/hugo.tar.gz hugo
      - name: Build
        run: ${RUNNER_TEMP}/hugo

      - name: Deploy
        if: github.ref == 'refs/heads/main'
        run: |
          cp -R public/* ${GITHUB_WORKSPACE}/built-site/
          cd ${GITHUB_WORKSPACE}/built-site
          git add .
          git config user.name '{github-유저이름}'
          git config user.email '{github-이메일}'
          git commit -m 'Updated site'
          git push
```

- 원격 origin/main에 푸시를 합니다.

```shell
$ git add . && git commit -m "init: define deploy.yml"
$ git push -u origin main
```

### 12. 작성후 gh-pages 브랜치를 먼저 생성하고, deploy.yml 파일을 푸시합니다.

```shell
# 브랜치 생성
$ git branch gh-pages

# 브랜치 체크아웃 및 원격래포에 gh-pages(origin/gh-pages) 브랜치 생성
$ git checkout gh-pages && git push --set-upstream origin gh-pages
```

### 13. 푸시하게되면 2개의 워크플로우가 진행됩니다.

1단계) 푸시후 바로 실행되는 deploy.yml 워크플로우
2단계) gh-pages 자동화봇 워크플로우 실행 - 단, 1단계의 결과가 성공해야 실행됩니다.

### 셋팅후 초기 프로젝트 디렉토리 구조

```shell
📦blog
 ┣ 📂.github
 ┃ ┗ 📂workflows
 ┃ ┃ ┗ 📜deploy.yml
 ┣ 📂archetypes
 ┃ ┗ 📜default.md
 ┣ 📂content
 ┃ ┗ 📂docs
 ┃   ┗ 📜test.md
 ┣ 📂public
 ┣ 📂themes
 ┃ ┗ 📂PaperMod
 ┣ 📜.gitignore
 ┣ 📜.gitmodules
 ┣ 📜.hugo_build.lock
 ┣ 📜README.md
 ┗ 📜hugo.yml
```

# Gemini CLI Documentation Site

이 디렉토리는 Gemini CLI ebook을 Hugo 정적 사이트 생성기로 빌드하고 GitHub Pages에 배포하기 위한 프로젝트입니다.

## 로컬에서 Hugo 사이트 테스트하기

### 1. Hugo 설치

#### macOS (Homebrew 사용)
```bash
brew install hugo
```

#### Linux (Snap 사용)
```bash
snap install hugo
```

#### Linux (수동 설치)
```bash
# Hugo 다운로드 및 설치
HUGO_VERSION=0.128.0
wget https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz
tar -xzf hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz
sudo mv hugo /usr/local/bin/
```

#### Windows (Chocolatey 사용)
```powershell
choco install hugo-extended
```

### 2. Hugo 설치 확인

```bash
hugo version
```

예상 출력:
```
hugo v0.128.0+extended ...
```

### 3. 로컬 개발 서버 실행

docs-site 디렉토리에서 다음 명령어를 실행합니다:

```bash
cd docs-site
hugo server
```

또는 프로젝트 루트에서:

```bash
hugo server -s docs-site
```

개발 서버가 시작되면 브라우저에서 다음 주소로 접속합니다:
```
http://localhost:1313/
```

### 4. 개발 서버 옵션

#### Draft 콘텐츠 포함하기
```bash
hugo server -D
```

#### 포트 변경
```bash
hugo server -p 8080
```

#### 자동 새로고침 비활성화
```bash
hugo server --disableLiveReload
```

#### 상세 로그 출력
```bash
hugo server --verbose
```

### 5. 프로덕션 빌드 테스트

실제 배포될 사이트를 로컬에서 빌드하려면:

```bash
cd docs-site
hugo --gc --minify
```

빌드된 파일은 `public/` 디렉토리에 생성됩니다.

빌드된 사이트를 로컬에서 확인하려면:

```bash
# Python 3 사용
cd public
python3 -m http.server 8000

# 또는 Node.js의 http-server 사용 (설치 필요)
npx http-server public -p 8000
```

브라우저에서 `http://localhost:8000` 접속

### 6. 콘텐츠 수정하기

#### 새 페이지 추가

1. `content/` 디렉토리에 Markdown 파일 생성
2. Front matter 추가:

```markdown
---
title: "페이지 제목"
weight: 10
---

페이지 내용...
```

#### 새 섹션 추가

1. `content/` 하위에 새 디렉토리 생성
2. `_index.md` 파일 생성:

```markdown
---
title: "섹션 제목"
weight: 1
---

섹션 설명...
```

### 7. 레이아웃 수정하기

- `layouts/_default/baseof.html`: 기본 HTML 템플릿
- `layouts/_default/list.html`: 목록 페이지 (섹션 페이지)
- `layouts/_default/single.html`: 단일 페이지
- `layouts/index.html`: 홈페이지

레이아웃 수정 후 Hugo 개발 서버가 자동으로 변경사항을 감지하고 페이지를 새로고침합니다.

### 8. 설정 변경하기

`hugo.toml` 파일에서 사이트 설정을 변경할 수 있습니다:

- `title`: 사이트 제목
- `baseURL`: 배포 URL
- `languageCode`: 언어 코드
- `params`: 커스텀 파라미터
- `menu`: 네비게이션 메뉴

## 프로젝트 구조

```
docs-site/
├── hugo.toml              # Hugo 설정 파일
├── content/               # Markdown 콘텐츠
│   ├── _index.md         # 홈페이지
│   ├── part1/            # 1부: 주요 구성 요소
│   ├── part2/            # 2부: LLM 인터페이스
│   └── part3/            # 3부: LLM과 Agent 사이
├── layouts/              # HTML 템플릿
│   ├── _default/
│   │   ├── baseof.html  # 기본 레이아웃
│   │   ├── list.html    # 목록 페이지
│   │   └── single.html  # 단일 페이지
│   └── index.html       # 홈페이지 템플릿
├── static/               # 정적 파일 (이미지, CSS, JS 등)
└── public/               # 빌드 출력 (git에서 제외됨)
```

## GitHub Pages 배포

이 사이트는 GitHub Actions를 통해 자동으로 배포됩니다:

1. `main` 브랜치에 변경사항을 푸시
2. `.github/workflows/hugo.yml` 워크플로우가 자동 실행
3. Hugo가 사이트를 빌드하고 GitHub Pages에 배포
4. https://thatseeai.github.io/gemini-cli/ 에서 확인 가능

## 트러블슈팅

### Hugo 명령어를 찾을 수 없음
```bash
# Hugo가 PATH에 있는지 확인
which hugo

# 없다면 다시 설치하거나 PATH에 추가
export PATH=$PATH:/usr/local/bin
```

### 빌드 오류: "module not found"
- `hugo.toml`에서 `theme` 설정을 제거하거나 주석 처리
- 이 프로젝트는 커스텀 레이아웃을 사용하며 별도의 테마가 필요 없습니다

### 스타일이 적용되지 않음
- `layouts/_default/baseof.html`의 `<style>` 태그가 올바른지 확인
- 브라우저 캐시를 지우고 다시 시도

### 페이지가 목록에 표시되지 않음
- Front matter에 `title`과 `weight`가 있는지 확인
- 파일이 올바른 `content/` 하위 디렉토리에 있는지 확인

## 참고 자료

- [Hugo 공식 문서](https://gohugo.io/documentation/)
- [Hugo 템플릿 가이드](https://gohugo.io/templates/)
- [Hugo 빠른 시작](https://gohugo.io/getting-started/quick-start/)
- [GitHub Pages 문서](https://docs.github.com/pages)

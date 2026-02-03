**MkDocs + Material for MkDocs + GitHub Actions** 조합은 기술 문서와 위키를 구축하는 데 있어 업계 표준에 가까운 매우 효율적인 구성입니다.

유지보수가 용이하고 확장이 쉬운 **환경 구성 가이드**와 **최소 템플릿(Boilerplate)**을 정리했습니다.

---

### 1. 프로젝트 디렉토리 구조 (최소 템플릿)

가장 먼저 로컬에 아래와 같은 구조로 파일과 디렉토리를 생성합니다.

```text
my-tech-docs/
├── .github/
│   └── workflows/
│       └── ci.yml          # GitHub Actions 배포 설정
├── docs/                   # 실제 문서가 저장되는 곳
│   ├── assets/             # 이미지 등 정적 파일
│   └── index.md            # 메인 페이지
├── mkdocs.yml              # MkDocs 설정 파일
└── requirements.txt        # 의존성 관리

```

---

### 2. 파일별 설정 내용

각 파일에 들어갈 핵심 코드를 작성해 두었습니다. 그대로 복사하여 사용하시면 됩니다.

#### A. `requirements.txt`

빌드 및 배포에 필요한 패키지입니다.

```text
mkdocs-material

```

#### B. `mkdocs.yml` (설정 파일)

기술 문서에 필수적인 **코드 하이라이팅**, **복사 버튼**, **검색**, **다크모드 전환** 기능을 포함한 최적화된 설정입니다.

```yaml
site_name: Tech Wiki & Manual
site_url: https://<USERNAME>.github.io/<REPO-NAME>/
site_author: <YOUR_NAME>
site_description: Technical documentation and wiki.

# 1. 테마 설정 (Material)
theme:
  name: material
  language: ko
  palette:
    # 라이트 모드
    - scheme: default
      primary: indigo
      accent: indigo
      toggle:
        icon: material/weather-sunny
        name: Switch to dark mode
    # 다크 모드
    - scheme: slate
      primary: indigo
      accent: indigo
      toggle:
        icon: material/weather-night
        name: Switch to light mode
  features:
    - navigation.tabs        # 상단 탭 네비게이션
    - navigation.sections    # 섹션별 그룹화
    - navigation.top         # 맨 위로 가기 버튼
    - search.suggest         # 검색어 제안
    - search.highlight       # 검색 결과 하이라이트
    - content.code.copy      # 코드 복사 버튼

# 2. 마크다운 확장 기능 (기술 문서용 필수 조합)
markdown_extensions:
  - admonition               # 경고/참고 박스 (!!! note 등)
  - pymdownx.details         # 접기/펼치기 기능
  - pymdownx.superfences     # 코드 블록 중첩 지원
  - pymdownx.highlight:      # 코드 하이라이팅
      anchor_linenums: true
  - pymdownx.inlinehilite
  - pymdownx.snippets        # 외부 코드 삽입 지원
  - toc:
      permalink: true        # 헤더 옆에 앵커 링크 생성

# 3. 플러그인
plugins:
  - search

```

#### C. `.github/workflows/ci.yml` (자동 배포)

`main` 브랜치에 코드가 푸시되면 자동으로 `gh-pages` 브랜치에 정적 사이트를 빌드하고 배포합니다.

```yaml
name: Publish docs via GitHub Pages

on:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: 3.x

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Configure Git Credentials
        run: |
          git config user.name github-actions[bot]
          git config user.email 41898282+github-actions[bot]@users.noreply.github.com

      - name: Deploy to gh-pages
        run: |
          mkdocs gh-deploy --force

```

#### D. `docs/index.md` (예시)

```markdown
# Welcome to Tech Wiki

이곳은 기술 문서 및 매뉴얼을 위한 공간입니다.

## 시작하기

- [ ] `mkdocs.yml` 파일에서 `site_name`과 `site_url`을 수정하세요.
- [ ] `docs/` 폴더 안에 `.md` 파일을 추가하여 문서를 작성하세요.

## 코드 블록 테스트

```bash
echo "Hello, MkDocs!"
```

---

### 3. 배포 및 설정 적용 단계 (환경 구성 가이드)

다음 순서대로 진행하면 즉시 사이트가 게시됩니다.

1. **GitHub 저장소 생성**: GitHub에서 새 리포지토리를 생성합니다. (Public/Private 무관)
2. **파일 푸시**: 위에서 작성한 파일 구조를 `main` 브랜치에 푸시합니다.
3. **Actions 확인**: GitHub 저장소의 `Actions` 탭에서 워크플로우가 성공적으로 도는(Green light)지 확인합니다.
    * 최초 실행 시 `gh-pages` 브랜치가 자동으로 생성됩니다.
4. **Pages 설정**:
    * GitHub 저장소 `Settings` &rarr; `Pages` 메뉴로 이동합니다.
    * **Build and deployment** 섹션의 Source를 **Deploy from a branch**로 선택합니다.
    * Branch를 `gh-pages` / `/(root)`로 설정하고 Save를 누릅니다.
5. **확인**: 잠시 후 상단에 표시되는 URL(`https://<ID>.github.io/<REPO>/`)로 접속하여 문서를 확인합니다.

---

### 4. 상황별 '환경 구성'

#### 1. 배포 및 운영만 하는 경우 (설치 불필요)

GitHub 저장소에 `.md` 파일만 업로드하면 됩니다.

* **작동 원리:** 코드를 Push 하면, GitHub Actions가 `requirements.txt`를 보고 가상 머신에 MkDocs를 알아서 설치하고, 빌드해서 사이트를 업데이트합니다.
* **작업 방식:**

1. VS Code 등에서 Markdown 파일 작성.
2. Git Push.
3. 끝 (약 1분 뒤 사이트 자동 갱신).

#### 2. 작성 중 화면을 미리 보고 싶은 경우 (선택적 설치)

작성한 문서를 Push 하기 전에 내 PC에서 바로 확인하고 싶다면(프리뷰), 그때는 로컬 환경 구성이 필요합니다.

* **필요한 이유:** 오타나 레이아웃 깨짐을 실시간으로 확인(`mkdocs serve`)하기 위함입니다.
* **설치 방법:**

```bash
# Python이 설치된 상태에서 단 한 줄만 실행
pip install -r requirements.txt

# 라이브 서버 실행 (http://127.0.0.1:8000 접속)
mkdocs serve

```

#### 3. "설치 없이" 미리보기도 하고 싶다면 (추천)

팀원들이 Python 설치를 번거로워한다면 **GitHub Codespaces**를 사용하세요.

1. GitHub 저장소 페이지에서 `.` 키를 누르거나, `<> Code` 버튼 → `Codespaces` 탭에서 생성.
2. 브라우저 안에서 VS Code가 열리고, 터미널에서 `pip install -r requirements.txt` 후 `mkdocs serve`를 입력하면 웹상에서 미리보기가 가능합니다.

#### 요약

* **운영/배포:** 로컬 설치 **X** (GitHub Actions가 수행)
* **로컬 미리보기:** 로컬 설치 **O** (선택 사항)

---

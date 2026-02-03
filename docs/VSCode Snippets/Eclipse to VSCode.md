Antigravity는 **VSCode 기반**의 IDE이므로, 기본적으로 **VSCode의 Java 설정 방식(Extension Pack for Java)**을 따릅니다.

Eclipse의 설정들은 VSCode에서는 주로 **`.vscode/settings.json`** 파일에 작성하거나, 일부는 **Eclipse 설정 파일(`.xml`, `.prefs`)을 직접 참조**하는 방식으로 마이그레이션합니다.

요청하신 항목별 설정 방법을 정리해 드립니다.

---

### 1. Java > Code Style

Eclipse의 강력한 포맷팅과 정리 기능을 VSCode로 가져오는 방법입니다.

#### A. Formatter (포맷터)

Eclipse에서 사용하던 포맷터 설정 파일(`formatter.xml`)을 그대로 사용할 수 있습니다.

1. Eclipse에서: `Preferences` > `Java` > `Code Style` > `Formatter` > **Export All** (XML 파일로 저장)
2. VSCode(Antigravity) 프로젝트 루트에 해당 XML 파일을 복사합니다. (예: `eclipse-formatter.xml`)
3. **`.vscode/settings.json`**에 아래 내용을 추가합니다:

```json
"java.format.settings.url": "eclipse-formatter.xml",
"java.format.settings.profile": "설정된_프로필_이름" // (선택사항: XML 안에 프로필이 여러 개일 경우)

```

#### B. Organize Imports & Clean Up (저장 시 자동 동작)

Eclipse의 `Save Actions` 기능입니다. 파일 저장 시 자동으로 import 정리와 포맷팅을 수행하게 합니다.

**`.vscode/settings.json`**:

```json
"editor.codeActionsOnSave": {
    "source.organizeImports": "explicit", // 불필요한 import 제거 및 정렬
    "source.fixAll": "explicit"           // 기타 린트 문제 자동 수정
},
// 포맷팅도 저장 시 수행하려면:
"editor.formatOnSave": true

```

#### C. Code Templates (주석 템플릿)

새 파일 생성 시 붙는 파일 헤더나 클래스 주석 등입니다. VSCode Java 확장은 Eclipse 스타일의 변수를 지원합니다.

**`.vscode/settings.json`**:

```json
"java.templates.fileHeader": [
    "/*",
    " * Copyright (c) ${year} Your Company",
    " * All rights reserved.",
    " */"
],
"java.templates.typeComment": [
    "/**",
    " * @author ${user}",
    " * @since ${date}",
    " */"
]

```

---

### 2. Java > Compiler

#### Javadoc (유효성 검증)

Eclipse의 `Compiler > Javadoc`에서 설정하던 "Malformed Javadoc comments" 같은 경고/에러 수준 제어입니다.
VSCode는 내부적으로 Eclipse JDT Language Server를 사용하므로, 프로젝트 폴더 내의 Eclipse 설정 파일을 직접 인식합니다.

1. **방법 1: `.settings` 폴더 사용 (추천)**

* 프로젝트 루트에 `.settings/org.eclipse.jdt.core.prefs` 파일이 있다면 VSCode가 이를 자동으로 읽어들입니다.
* Eclipse에서 해당 프로젝트의 설정을 완료한 후, `.settings` 폴더 전체를 VSCode 프로젝트로 가져오면 가장 완벽하게 동기화됩니다.

1. **방법 2: 직접 설정 (settings.json)**

* VSCode 설정만으로 간단히 처리하고 싶다면 아래 설정을 사용합니다. (다만 Eclipse만큼 세세하지 않습니다.)

```json
// 예: Javadoc이 없는 경우 경고 처리 여부 등
"java.compile.nullAnalysis.mode": "automatic"

```

*Tip: 컴파일러 경고 수준을 섬세하게 조정하려면 **방법 1(prefs 파일 복사)**이 가장 확실합니다.*

---

### 3. Java > Editor

#### Templates (스니펫)

Eclipse의 `sysout`, `foreach` 같은 단축키 템플릿입니다. VSCode에서는 **"User Snippets"**라고 부릅니다.

1. 명령어 팔레트(`Ctrl`+`Shift`+`P`) > **Snippets: Configure User Snippets** 선택
2. **java.json** 선택
3. Eclipse 템플릿 내용을 변환하여 등록 (앞서 답변드린 방식 활용)

**[예시: System.out.printf]**

```json
"Print Formatted": {
    "prefix": "printf",
    "body": [
        "System.out.printf(\"${1:Msg}: %s%n\", ${2:args});"
    ],
    "description": "System.out.printf shortcut"
}

```

---

### 4. Java > Installed JREs

JDK 경로 설정입니다. VSCode는 여러 버전의 JDK를 등록해두고 프로젝트별로 선택해서 사용할 수 있습니다.

**`.vscode/settings.json`** (또는 사용자 설정):

```json
"java.configuration.runtimes": [
    {
        "name": "JavaSE-1.8",
        "path": "/Library/Java/JavaVirtualMachines/jdk1.8.0_291.jdk/Contents/Home",
        "default": true  // 이 JDK를 기본으로 사용
    },
    {
        "name": "JavaSE-17",
        "path": "/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home"
    }
]

```

* **name**: Eclipse 프로젝트 설정(`.classpath`)과 매핑되는 이름입니다. (보통 `JavaSE-1.8`, `JavaSE-11` 등)
* **path**: 실제 JDK가 설치된 **절대 경로**여야 합니다.

---

### 💡 요약: 한 번에 설정하기 (settings.json 예시)

프로젝트 루트의 `.vscode/settings.json` 파일을 생성하고 아래 내용을 환경에 맞게 수정하여 붙여넣으세요.

```json
{
    // [4. Installed JREs] JDK 경로 설정
    "java.configuration.runtimes": [
        {
            "name": "JavaSE-17",
            "path": "C:\\Program Files\\Java\\jdk-17",
            "default": true
        }
    ],

    // [1-A. Formatter] Eclipse 포맷터 XML 파일 경로
    "java.format.settings.url": "eclipse-formatter.xml",

    // [1-B. Save Actions] 저장 시 자동 정리
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
        "source.organizeImports": "explicit"
    },

    // [1-C. Code Templates] 파일/클래스 주석
    "java.templates.fileHeader": [
        "/* Copyright (c) ${year} ... */"
    ],
    "java.templates.typeComment": [
        "/**",
        " * @author ${user}",
        " */"
    ]
}

```

추가적으로 Antigravity(VSCode) 내에서 **Eclipse 단축키 키맵(Keymap)** 확장을 설치하시면 `Ctrl+D` (라인 삭제), `Ctrl+Shift+F` (포맷팅) 등 익숙한 단축키를 그대로 사용하실 수 있습니다.

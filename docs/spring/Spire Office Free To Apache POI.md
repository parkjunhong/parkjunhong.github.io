# Spire Office Free To Apache POI

```
기존 기능을 Spire 기준으로 먼저 분해하고, 그다음 Apache POI에서 대체 가능한 영역과 불가능한 영역을 나눠서 마이그레이션 절차로 정리해보겠습니다. PDF 변환·렌더링처럼 POI가 직접 못 하는 부분이 있는지도 같이 체크하겠습니다.

핵심 차이점이 하나 보입니다. Apache POI는 Excel 쪽이 가장 성숙하고, Word/PowerPoint도 지원하지만 Spire처럼 “통합형 상용 API 느낌의 1:1 대체”는 아닙니다. 특히 PDF 변환·이미지 렌더링·고급 레이아웃 쪽은 사전에 분리해서 보셔야 합니다.
```

**Spire.Office.Free → Apache POI 마이그레이션은 “라이브러리 교체”가 아니라 “기능별 재설계”에 가깝습니다.** Apache POI는 Excel 지원이 가장 성숙하고, Word/PowerPoint도 지원하지만 Spire처럼 모든 Office 기능을 하나의 통합 API 감성으로 1:1 치환해 주지는 않습니다. 또 POI의 주된 범위는 **MS Office 문서 포맷 조작**이며, PDF 변환 같은 영역은 별도 판단이 필요합니다. 현재 최신 안정 버전은 **Apache POI 5.5.1**입니다. ([Apache POI][1])

## 1. 가장 먼저 할 판단

먼저 지금 Spring Boot 서비스에서 Spire로 무엇을 하는지 기능을 쪼개야 합니다.

이 분류가 제일 중요합니다.

* **A. 직접 이전 쉬움**

  * Excel 읽기/쓰기
  * 셀 값, 스타일, 병합, 수식, 시트 복사 수준
  * DOCX 텍스트/문단/표 생성
  * PPTX 슬라이드/텍스트/이미지 기본 조작

* **B. 조건부 이전**

  * 대용량 Excel 생성
  * 암호 문서 처리
  * 차트/도형/이미지 포함 문서
  * 오래된 binary 포맷(.xls/.doc/.ppt) 지원

* **C. 별도 검토 필요**

  * PDF 변환/렌더링
  * 높은 수준의 문서 레이아웃 fidelity
  * Spire 전용 편의 API에 많이 의존한 부분

Apache POI는 Excel(HSSF/XSSF/SXSSF)이 가장 성숙하고, Word(HWPF/XWPF), PowerPoint(HSLF/XSLF)도 지원합니다. 다만 POI 자체 소개에서도 Excel 쪽이 가장 발전되어 있고 Word/PowerPoint는 계속 발전 중이라고 설명합니다. ([Apache POI][2])

---

## 2. 마이그레이션 전체 절차

### 2-1. 현재 Spire 사용처 전수조사

가장 먼저 해야 할 일은 코드에서 `com.spire` import를 전부 수집하는 것입니다.

조사 결과를 이런 식으로 정리하세요.

* 패키지/클래스명
* 파일 종류: xlsx / xls / docx / doc / pptx / ppt / pdf
* 동작: 생성 / 읽기 / 수정 / 변환 / 이미지 추출 / 텍스트 추출 / 암호
* 입력/출력 크기
* HTTP 응답 다운로드인지, 배치 생성인지
* 장애 허용 수준: 서식 100% 동일해야 하는지, 내용만 같으면 되는지

예시 분류:

* `Workbook`, `Worksheet`, `CellRange` 사용 → Excel 영역
* `Document`, `Section`, `Paragraph`, `Table` 사용 → Word 영역
* `Presentation`, `Slide`, `Shape` 사용 → PowerPoint 영역
* `SaveToFile(... PDF ...)`, `SaveToStream(... PDF ...)` → **POI 단독 대체 불가 가능성 높음**

이 단계에서 반드시 **“Spire API 사용 목록”과 “대체 목표 API 목록”**을 1:N으로 매핑해야 합니다.

---

## 3. Apache POI 대상 기술 매핑

POI는 파일 종류별로 모듈이 나뉩니다.

* **Excel**

  * `.xls` → `HSSF`
  * `.xlsx` → `XSSF`
  * 대용량 `.xlsx` 쓰기 → `SXSSF`
  * 공통 인터페이스 → `org.apache.poi.ss.usermodel.*`

* **Word**

  * `.doc` → `HWPF`
  * `.docx` → `XWPF`

* **PowerPoint**

  * `.ppt` → `HSLF`
  * `.pptx` → `XSLF`

* **OLE2/문서 속성**

  * `POIFS`, `HPSF`

이 구조는 POI 공식 컴포넌트 문서에 정리되어 있습니다. Excel은 공통 SS API가 있고, Word/PowerPoint는 포맷별 구현이 나뉩니다. ([Apache POI][3])

실무적으로는 보통 이렇게 대응합니다.

* Spire Excel Workbook → `Workbook`, `XSSFWorkbook`, `HSSFWorkbook`, `SXSSFWorkbook`
* Spire Worksheet → `Sheet`
* Spire Row/Cell → `Row`, `Cell`
* Spire Range → `CellRangeAddress`
* Spire Word Document → `XWPFDocument` 또는 `HWPFDocument`
* Spire PowerPoint Presentation → `XMLSlideShow` 또는 `HSLFSlideShow`

---

## 4. 의존성 교체

Spring Boot 서비스에서는 보통 Maven 기준으로 먼저 의존성을 분리합니다.

```xml
<properties>
    <poi.version>5.5.1</poi.version>
</properties>

<dependencies>
    <!-- Excel / 공통 -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi</artifactId>
        <version>${poi.version}</version>
    </dependency>

    <!-- OOXML: xlsx / docx / pptx -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>${poi.version}</version>
    </dependency>

    <!-- 구형 포맷 doc / ppt 등 필요 시 -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-scratchpad</artifactId>
        <version>${poi.version}</version>
    </dependency>
</dependencies>
```

주의할 점이 하나 있습니다. OOXML 스키마가 많이 필요한 고급 기능을 쓰면 `poi-ooxml-lite`로 부족할 수 있고, 이 경우 FAQ에서 말하는 **full schema** 계열을 검토해야 합니다. POI FAQ는 `poi-ooxml-lite`와 `poi-ooxml-full`의 차이를 안내합니다. ([Apache POI][1])

또 운영 환경에 **옛날 POI jar가 숨어 있으면** `MethodNotFoundException`, `IncompatibleClassChangeError` 같은 충돌이 날 수 있다고 FAQ가 경고합니다. 기존 WAS/lib, 공통 모듈, 사내 공통 라이브러리에 오래된 POI가 섞여 있지 않은지 반드시 확인해야 합니다. ([Apache POI][4])

---

## 5. 추천하는 전환 방식

### 5-1. 한 번에 갈아엎지 말고 어댑터 계층을 먼저 만드세요

가장 안전한 방식은 서비스 코드가 Spire/POI를 직접 모르도록 만드는 것입니다.

예를 들면:

```java
public interface ExcelDocumentPort {

    byte[] createReport(ReportData data) throws Exception;

    ParsedExcel read(InputStream inputStream) throws Exception;
}
```

그다음 구현을 두 개 둡니다.

```java
public class SpireExcelDocumentPort implements ExcelDocumentPort {
    @Override
    public byte[] createReport(ReportData data) throws Exception {
        // 기존 Spire 구현
        return null;
    }

    @Override
    public ParsedExcel read(InputStream inputStream) throws Exception {
        // 기존 Spire 구현
        return null;
    }
}
```

```java
public class PoiExcelDocumentPort implements ExcelDocumentPort {
    @Override
    public byte[] createReport(ReportData data) throws Exception {
        // Apache POI 구현
        return null;
    }

    @Override
    public ParsedExcel read(InputStream inputStream) throws Exception {
        // Apache POI 구현
        return null;
    }
}
```

그리고 서비스는 포트만 보게 합니다.

이렇게 해야:

* 파일 종류별 순차 전환 가능
* A/B 테스트 가능
* 장애 시 롤백 쉬움
* 테스트 코드 재사용 가능

---

## 6. 기능별 상세 마이그레이션 방법

# 6-1. Excel 먼저 옮기기

실무적으로 가장 먼저 옮기기 좋은 것이 Excel입니다. POI는 Excel 지원이 가장 성숙하고, 공통 SS API와 대용량 쓰기용 SXSSF도 제공합니다. ([Apache POI][2])

### 절차

1. **파일 포맷 결정**

   * `.xlsx` 중심이면 `XSSFWorkbook`
   * 매우 큰 파일 쓰기면 `SXSSFWorkbook`
   * `.xls` 유지 필요하면 `HSSFWorkbook`

2. **기본 API 치환**

   * Workbook 생성
   * Sheet 생성
   * Row / Cell 생성
   * CellStyle, Font, DataFormat 이식
   * 병합 셀, 너비, 높이, 정렬 이식

3. **수식 처리 이식**

   * `setCellFormula(...)`
   * 필요하면 `FormulaEvaluator`
   * 일부 계산은 Excel 열기 시 재계산 유도도 가능

4. **대용량 처리**

   * 생성 건수가 많으면 `SXSSFWorkbook`
   * 메모리 튜닝
   * HTTP 응답 스트리밍 방식 검토

### 기본 예시

```java
Workbook workbook = new XSSFWorkbook();
Sheet sheet = workbook.createSheet("report");

Row header = sheet.createRow(0);
Cell headerCell = header.createCell(0);
headerCell.setCellValue("이름");

CellStyle style = workbook.createCellStyle();
Font font = workbook.createFont();
font.setBold(true);
style.setFont(font);
headerCell.setCellStyle(style);

Row row = sheet.createRow(1);
row.createCell(0).setCellValue("홍길동");

try (ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
    workbook.write(bos);
    workbook.close();
    byte[] bytes = bos.toByteArray();
}
```

### 읽기 쪽은 가능하면 `WorkbookFactory` 우선

포맷 자동 판별이 필요하면 `WorkbookFactory`가 편합니다. 텍스트 추출도 문서 타입별 팩토리 계열이 있어 Spire의 “한 API로 여러 형식 처리” 감각을 일부 흉내 낼 수 있습니다. ([Apache POI][5])

### 대용량 Excel 주의

POI 공식 문서는 **대용량 쓰기에는 SXSSF**, 매우 큰 파일 읽기에는 스트리밍 성격의 접근을 보라고 안내합니다. SXSSF는 sliding window 방식이라 오래된 row는 다시 접근이 제한됩니다. 즉, Spire처럼 마음대로 뒤돌아가 수정하던 코드는 구조를 바꿔야 합니다. ([Apache POI][6])

---

# 6-2. Word 마이그레이션

Word는 `.docx`면 `XWPF`, `.doc`면 `HWPF`입니다. 다만 Word 쪽은 Excel만큼 매끄럽지 않을 수 있으니, 문단/표/이미지/헤더푸터 단위로 기능을 나눠서 옮기는 것이 좋습니다. `HWPF`는 공식 컴포넌트 설명에서도 초기 단계 성격이 남아 있고, `.docx` 중심이면 `XWPF` 우선 전략이 좋습니다. ([Apache POI][3])

### 절차

1. 현재 문서가 `.docx`인지 `.doc`인지 분리
2. `Paragraph`, `Run`, `Table`, `Header/Footer` 사용처 분리
3. 템플릿 기반 생성인지, 코드 생성인지 구분
4. 서식 fidelity 테스트를 별도 수행

### 기본 예시

```java
XWPFDocument document = new XWPFDocument();

XWPFParagraph paragraph = document.createParagraph();
XWPFRun run = paragraph.createRun();
run.setText("안녕하세요.");
run.setBold(true);

try (ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
    document.write(bos);
    document.close();
    byte[] bytes = bos.toByteArray();
}
```

### Word에서 꼭 확인할 것

Spire에서 아래를 많이 썼다면 난이도가 올라갑니다.

* 문서 레이아웃을 정밀하게 맞추는 기능
* 복잡한 표 병합
* 이미지 배치 정밀 제어
* PDF 변환

POI 공식 문서는 `HWPF` 쪽에 Word-to-HTML, Word-to-FO 변환기를 언급하고, `.doc`에 대해 FOP와 연계해 PDF 생성이 가능하다고 설명합니다. 다만 이것은 **일반적인 DOCX/PDF 대체 솔루션으로 보기는 어렵고**, 적어도 “Spire의 SaveToPdf를 그대로 치환”하는 방식은 아니라는 뜻으로 받아들이는 것이 안전합니다. 이 부분은 기능 요구를 따로 떼어 설계하는 것이 좋습니다. ([Apache POI][7])

---

# 6-3. PowerPoint 마이그레이션

PowerPoint는 `.pptx`면 `XSLF`, `.ppt`면 `HSLF`입니다. 슬라이드 생성, 텍스트, 도형, 이미지, 기본 차트 정도는 가능하지만, PowerPoint도 Excel보다 검증 범위를 더 넓게 잡아야 합니다. ([Apache POI][8])

### 절차

1. 슬라이드 추가/복제/삭제
2. 텍스트 상자, 이미지, 도형 추가
3. 기존 템플릿 import 여부 확인
4. 썸네일/이미지 렌더링 필요 여부 확인

### 기본 예시

```java
XMLSlideShow ppt = new XMLSlideShow();
XSLFSlide slide = ppt.createSlide();

XSLFTextBox textBox = slide.createTextBox();
textBox.setAnchor(new java.awt.Rectangle(50, 50, 400, 80));
textBox.setText("제목");

try (ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
    ppt.write(bos);
    ppt.close();
    byte[] bytes = bos.toByteArray();
}
```

POI는 공식적으로 슬라이드를 이미지로 렌더링하는 `PPTX2PNG` 유틸리티와 `slide.draw(graphics)` 기반 렌더링 경로를 제공합니다. 따라서 **PPT/PPTX → 이미지 썸네일** 수준은 POI로 접근 가능합니다. ([Apache POI][9])

---

## 7. PDF 관련 판단

이 부분이 제일 중요합니다.

Apache POI의 주력 범위는 Office 문서 포맷 조작입니다. 따라서 현재 Spire에서 하고 있는 것이 아래라면 별도 설계가 필요합니다.

* Word/Excel/PPT를 PDF로 직접 변환
* PDF를 읽고 수정
* PDF를 이미지로 변환
* PDF 텍스트 추출

Free Spire.Office는 공식 다운로드 페이지에서도 Word/PowerPoint/PDF에 페이지 제한이 있다고 안내합니다. 그래서 Spire Free에서 벗어나려는 이유가 “제한/라이선스/경량화”라면 이해되지만, **POI만으로 모든 PDF 시나리오를 대체한다고 가정하면 중간에 막힐 가능성이 큽니다.** ([E-ICEBLUE][10])

즉, 현재 코드에 이런 것이 있으면 먼저 표시해 두셔야 합니다.

```java
document.saveToFile("x.pdf", FileFormat.PDF);
workbook.saveToFile("x.pdf", FileFormat.PDF);
presentation.saveToFile("x.pdf", FileFormat.PDF);
```

이건 “POI 전환” 대상이 아니라 **“POI + 별도 변환 전략”** 대상입니다.

---

## 8. 보안/운영 관점에서 꼭 넣어야 할 절차

POI 공식 보안 문서는 **신뢰할 수 없는 문서 업로드를 처리할 때 추가 보호 계층을 두라**고 안내합니다. 업로드 문서를 다루는 Spring Boot 서비스라면 이건 꼭 포함해야 합니다. ([Apache POI][11])

권장 절차는 이렇습니다.

1. 업로드 파일 크기 제한
2. 확장자만 보지 말고 MIME/시그니처 검사
3. 파싱 전 임시 저장소 격리
4. 처리 시간 제한
5. 대용량 문서는 별도 워커로 분리
6. 실패 시 원본 보존 및 오류 로그 표준화
7. 암호 문서는 별도 처리 경로

POI는 암호화된 Office 파일 일부를 읽고 처리하는 기능도 제공합니다. 암호 문서를 현재 Spire에서 처리 중이라면 이 기능은 별도 테스트 항목으로 빼야 합니다. ([Apache POI][12])

---

## 9. 테스트 절차

마이그레이션에서 제일 많이 실패하는 지점은 “파일은 열리는데 모양이 달라짐”입니다.

그래서 테스트를 3단계로 하셔야 합니다.

### 9-1. 구조 테스트

* 시트 수 / 슬라이드 수 / 문단 수
* 셀 값 / 텍스트 값
* 병합 범위
* 헤더/푸터 존재 여부
* 이미지 개수

### 9-2. 서식 테스트

* 폰트
* 굵기
* 배경색
* 테두리
* 정렬
* 열 너비 / 행 높이
* 표 구조

### 9-3. 사용자 체감 테스트

* 실제 Office에서 열리는지
* 깨짐이 없는지
* 인쇄/PDF 시 모양이 허용 범위인지

가장 좋은 방식은 **Golden file 테스트**입니다.

* 기존 Spire 결과물 저장
* POI 결과물 저장
* 내용 비교
* 사람이 보는 대표 샘플 시각 비교

바이너리 파일은 내부 메타데이터 차이 때문에 byte-by-byte 비교는 부정확할 수 있어서, **구조 비교 + 눈으로 확인할 샘플 비교**를 같이 두는 것이 좋습니다.

---

## 10. 실무형 추천 마이그레이션 순서

제가 추천하는 순서는 이렇습니다.

### 1단계

**Excel부터 이전**

* POI가 가장 강함
* 성공 경험 만들기 좋음
* 보고서/다운로드 기능 먼저 안정화 가능

### 2단계

**DOCX 생성 기능 이전**

* 단순 문단/표 중심부터
* 템플릿 복잡도 낮은 문서 우선

### 3단계

**PPTX 이전**

* 생성 기능부터
* 렌더링/썸네일은 별도 검증

### 4단계

**PDF 관련 기능 분리**

* “POI로 대체”가 아니라 “독립 기능”으로 재설계

### 5단계

**구형 포맷(.doc/.ppt/.xls) 지원 여부 최종 결정**

* 꼭 필요한 것만 유지
* 가능하면 OOXML(.docx/.xlsx/.pptx)로 표준화

---

## 11. 실제 작업 항목 체크리스트

바로 착수용으로 정리하면 아래 순서가 좋습니다.

### 준비

* `com.spire` import 전수조사
* 기능 목록 작성
* 파일 타입별 분류
* PDF 의존 기능 표시

### 기술 설계

* Office 포트 인터페이스 작성
* Spire 구현 / POI 구현 분리
* 공통 DTO 설계
* 예외 체계 통일

### 개발

* Maven 의존성 추가
* Excel 기능 이전
* Word 기능 이전
* PowerPoint 기능 이전
* 대용량 처리 최적화

### 검증

* 단위 테스트
* Golden file 비교
* 실제 Office 열기 테스트
* 운영 샘플 데이터 검증

### 전환

* feature flag로 병행 운영
* 일부 API만 POI 전환
* 로그/에러율 비교
* Spire 제거

---

## 12. 진행 중 자주 만나는 문제

### 문제 1. Spire는 몇 줄인데 POI는 코드가 길다

정상입니다.
Spire는 고수준 편의 API가 많고, POI는 상대적으로 저수준 조합이 많습니다.

### 문제 2. Word/PPT가 100% 동일하게 안 나온다

가능성이 있습니다.
특히 정교한 레이아웃 문서는 사전 허용 오차를 정해야 합니다. ([Apache POI][2])

### 문제 3. 대용량 Excel에서 메모리가 많이 든다

`SXSSFWorkbook`로 바꾸고, “한 번 쓴 row를 다시 수정하는 방식”을 버려야 합니다. ([Apache POI][6])

### 문제 4. 배포 후 클래스 충돌이 난다

오래된 POI jar 중복 가능성이 큽니다. 공식 FAQ가 바로 이 케이스를 다룹니다. ([Apache POI][4])

---

## 13. 제일 현실적인 결론

현실적으로는 이렇게 보시면 됩니다.

* **Excel 중심 서비스**라면 Apache POI 전환은 매우 현실적입니다. ([Apache POI][13])
* **DOCX/PPTX 기본 생성/수정**도 충분히 가능합니다. ([Apache POI][7])
* **PDF 변환/고정밀 렌더링까지 Spire처럼 한 번에 대체**하려고 하면 설계가 틀어질 수 있습니다. 이 부분은 별도 기능으로 분리해서 봐야 합니다. ([Apache POI][2])

원하시면 다음 단계로 이어서 **“Spire 클래스/메서드 → Apache POI 클래스/코드” 매핑표**를 사용 중인 코드 기준으로 상세하게 작성해드리겠습니다.

[1]: https://poi.apache.org/download.html?utm_source=chatgpt.com "Apache POI™ - Download Release Artifacts"
[2]: https://poi.apache.org/?utm_source=chatgpt.com "Apache POI™ - the Java API for Microsoft Documents"
[3]: https://poi.apache.org/components/?utm_source=chatgpt.com "Apache POI™ - Component Overview"
[4]: https://poi.apache.org/help/faq.html?utm_source=chatgpt.com "Frequently Asked Questions"
[5]: https://poi.apache.org/text-extraction.html?utm_source=chatgpt.com "Apache POI™ - Text Extraction"
[6]: https://poi.apache.org/components/spreadsheet/how-to.html?utm_source=chatgpt.com "The New Halloween Document - Apache POI"
[7]: https://poi.apache.org/components/document/?utm_source=chatgpt.com "HWPF and XWPF - Java API to Handle Microsoft Word Files"
[8]: https://poi.apache.org/components/slideshow/?utm_source=chatgpt.com "Java API To Access Microsoft Powerpoint Format Files"
[9]: https://poi.apache.org/components/slideshow/ppt-wmf-emf-renderer.html?utm_source=chatgpt.com "Rendering slideshows, WMF, EMF and EMF+ - Apache POI"
[10]: https://www.e-iceblue.com/Download/office-for-java-free.html?utm_source=chatgpt.com "Downloads - Free Spire.Office for Java"
[11]: https://poi.apache.org/security.html?utm_source=chatgpt.com "Apache POI™ - Security guidance"
[12]: https://poi.apache.org/encryption.html?utm_source=chatgpt.com "Apache POI™ - Encryption support"
[13]: https://poi.apache.org/components/spreadsheet/?utm_source=chatgpt.com "POI-HSSF and POI-XSSF/SXSSF - Java API To Access ..."

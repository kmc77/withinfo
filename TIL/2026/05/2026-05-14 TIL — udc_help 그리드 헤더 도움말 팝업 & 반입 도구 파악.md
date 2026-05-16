# 2026-05-14 TIL — udc_help 그리드 헤더 도움말 팝업 & 반입 도구 파악

---

## 1. udc_help 그리드 헤더 도움말 팝업 적용 (미완)

### 목표
그리드 헤더 아이콘 컬럼(`column49`) 클릭 시 `udc_help` 말풍선 팝업으로 특정 텍스트 노출.

---

### udc_help.xml 구조 파악

컴포넌트 팔레트 등록 파일(`<components>` 구조)과 실제 UDV 파일(`/svc/cm/udv/udc_help.xml`)은 **별개**.

실제 UDV 내부 DOM 구조:
```
w2:anchor#btn_tipOpen.btn_tip        ← 도움말 열기 아이콘
xf:group#grp_help.help_box           ← 팝업 컨테이너
  └ w2:anchor#btn_tipClose.help_close
  └ xf:group.info_list (ul)
      └ w2:generator#gen_infoList
          └ w2:textbox#tbx_info.tit
```

`setLabel` 내부 로직 (UDC 내부 정의):
```js
udc_obj.setLabel = function(value){
    if(nGenCnt == 0) gen_infoList.removeAll();
    nGenCnt = gen_infoList.insertChild();
    var objTextBox = gen_infoList.getChild(nGenCnt, "tbx_info");
    objTextBox.setLabel(value);
    nGenCnt++;
};
```

팝업 열기/닫기 로직 (UDC 내부):
```js
// btn_tipOpen onclick
grp_help.toggleClass("on");
this.toggleClass("on");

// btn_tipClose onclick
btn_tipOpen.toggleClass("on");
grp_help.toggleClass("on");
```

→ `btn_tipOpen`과 `grp_help` **둘 다** `on` 클래스 토글되어야 CSS 조건 충족.

---

### UDV 자식 컴포넌트 접근 경로

외부 화면에서 UDV 내부 자식 컴포넌트 접근:
```js
udc_help_grd.scope.btn_tipOpen   // w2:anchor 타입
udc_help_grd.scope.grp_help      // xf:group 타입
```

`udc_help_grd.btn_tipOpen` (직속) 접근은 불가 — **반드시 `.scope.` 경유**.

---

### setLabel 주의사항

**`setLabel`은 반드시 `onpageload`에서 1회만 호출.**

`onheaderclick`에서 호출하면 헤더 클릭마다 `gen_infoList`에 중복 삽입 → `firstChild` 에러 발생.

```js
// 올바른 위치
scwin.onpageload = function() {
    udc_help_grd.setLabel(`[중소기업확인서 발급 요건] <br>
        - 대상: 운수 및 창고업(H) 영위 기업 <br>
        - 기준: ① 연 매출액 1,000억 원 이하 & (2) 자산 총액 5,000억 원 미만 <br>
        (직전 사업연도 재무제표 기준) <br>
        (기준 초과 시에도 사유에 따라 향후 5년간 지위 유지 가능)`);
};
```

---

### 그리드 헤더 UDV 삽입 방법

Studio 디자이너에서 헤더 더블클릭은 `value` 속성만 편집 가능 → **XML 소스 탭에서 직접 수정**.

```xml
<!-- value 속성 방식 (불가) -->
<w2:column id="columnXX" value="선사분류"/>

<!-- w2:header 자식 구조 (정상) -->
<w2:column id="columnXX" ...>
    <w2:header>
        선사분류
        <w2:udc_help id="udc_help_01" style="width:16px;height:16px;"></w2:udc_help>
    </w2:header>
</w2:column>
```

헤더 raw HTML 직접 삽입(`<div>`, `<img>`)은 이스케이프 처리되어 텍스트로 찍힘 → UDV 컴포넌트로만 가능.

---

### 시도한 방법 및 결과

| 시도 | 결과 |
|------|------|
| `scwin.udc_help_grd_onUserClick_1()` | `is not a function` |
| `WebSquare.util.getComponentById("udc_help_grd:btn_tipOpen").trigger("click")` | `trigger` 메서드 없음 |
| `udc_help_grd.btn_tipOpen.toggleClass("on")` | btn_tipOpen이 직속에 없음 |
| `udc_help_grd.scope.btn_tipOpen.toggleClass("on")` | `Cannot read properties of undefined (reading 'toggleClass')` |
| `udc_help_grd.scope.grp_help.toggleClass("on")` | 에러 없으나 팝업 안 뜸 |
| `udc_help_grd.scope.btn_tipOpen.dom.anchor.click()` | 검증 불충분 |
| jQuery `$(...).toggleClass("on")` | min 거부 — WebSquare 컴포넌트 메서드 방식 유지 원칙 |

**현재 상태:** 미해결. 다음 세션에서 계속.

---

### 다음 세션 진입점

```js
// 콘솔에서 확인
console.log(udc_help_grd.scope.btn_tipOpen);
// → 정상 객체면 toggleClass 메서드 존재 여부 확인
// → undefined면 scope 접근 경로 재검토
```

---

### XML 현재 상태

```xml
<!-- 그리드 밖 UDV 선언 -->
<w2:udc_help id="udc_help_grd" style=""></w2:udc_help>

<!-- 그리드 이벤트 바인딩 -->
ev:onheaderclick="scwin.grd_loanIntSprtBizListVO_onheaderclick"

<!-- 아이콘 컬럼 -->
<w2:column width="0" inputType="image" style="height:28px;" id="column49"
    value="&lt;- 아이콘" displayMode="label" class="udc_help_grd"
    sortable="false" fixColumnWidth="true"></w2:column>
```

---

## 2. 반입 허가 프로그램 파악

| 프로그램 | 용도 | 활용도 |
|---|---|---|
| **Affinity** | Adobe 대체 디자인 툴 | 화면설계서 이미지 편집 |
| **ColorPicker** | 화면 색상 코드 추출 | UI 색상 맞출 때 |
| **Git for Windows** | Git CLI + Git Bash | GitHub Desktop 보조 |
| **Lunacy** | Figma 대체 UI 디자인 툴 | 오프라인 UI 설계 |
| **NET Framework 4.6.1** | Windows 앱 런타임 | 다른 프로그램 의존성 |
| **Photopea** | 포토샵 대체 이미지 편집 | 화면설계서 이미지 작업 |
| **Visual C++ Redistributable 2015** | C++ 런타임 | 다른 프로그램 의존성 |
| **VSCode** | 코드 에디터 | Eclipse 보조 XML/JS 편집 |
| **WinMerge** | 파일/폴더 diff 비교 | 쿼리·XML 변경 전후 비교 |

---

## 3. WinMerge 사용법

### 기본 조작
- `File > Open` → 왼쪽/오른쪽 파일 또는 폴더 지정
- 차이 나는 줄 노란색/빨간색 하이라이트
- `F4` / `Shift+F4` — 다음/이전 diff 이동
- 우클릭 → `Copy to Right/Left` — 한쪽 내용을 반대쪽으로 복사
- 폴더 비교 시 변경된 파일만 필터링 가능

### 폐쇄망 환경 활용 포인트
- **쿼리 수정 전후 비교** — 원본 SQL vs 수정 SQL diff로 실수 방지
- **Jenkins 배포 전 XML 검증** — 로컬 vs 의도치 않은 변경 사전 차단
- **Git 충돌 해결** — 3-way merge 시각적으로 처리
- **폴더 단위 비교** — 어떤 파일 건드렸는지 한눈에 확인
- **화면설계서 버전 비교** — txt 추출 후 버전 간 변경 이력 추적

---

## 요약

| 구분 | 내용 |
|------|------|
| 핵심 학습 | UDV 자식 컴포넌트는 `.scope.` 경유 접근 |
| 핵심 학습 | `setLabel`은 반복 호출 금지 — onpageload 1회 |
| 핵심 학습 | 그리드 헤더 컴포넌트 삽입은 `w2:header` 자식 구조 필수 |
| 미해결 | `btn_tipOpen.toggleClass("on")` 외부 호출 방법 |
| 신규 도구 | WinMerge, VSCode, Git for Windows 등 반입 완료 |

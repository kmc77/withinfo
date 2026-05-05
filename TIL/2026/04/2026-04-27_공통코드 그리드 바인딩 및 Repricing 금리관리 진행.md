# 2026-04-27_공통코드 그리드 바인딩 및 Repricing 금리관리 진행

> Repricing 금리관리 화면에서 setCommonCode 그리드 컬럼 바인딩, expression 컬럼, oneditend 이벤트, 공통버튼 제어 등을 학습. 국가필수선대는 등록자명 표시를 위한 TSM_USER_I JOIN 추가.

---

## 1. 공통버튼 영역(NGS_PGTITLE wframe) 제어

화면 최상단의 공통 버튼 영역은 wframe으로 끌어다 사용 중. 화면별로 보일 버튼/숨길 버튼을 제어하는 방법.

### 패턴 1 — commonSf.js의 switch case에 분기 추가 (다른 화면들 방식)

`sfcom.win.gubtnControl` 함수 내부 switch case에 화면 ID별 분기:

```js
} else if ( vScrnId == "CWMI_SFCM201M" ) {    // Repricing 금리관리
    com.util.showBtnById("btn_save");
    com.util.setBtnDisableById(true, "btn_save");   // 비활성화
}
```

### 패턴 2 — 화면 onpageload에서 직접 호출 (commonSf.js 미수정)

```js
sfcom.win.gubtnControl(btnOptions);

com.util.showBtnById("btn_save");
com.util.setBtnDisableById(true, "btn_save");
```

**선택 기준:** 공통파일(`commonSf.js`) 미수정 원칙을 따르려면 패턴 2. Git pull 시 충돌 회피에도 유리.

### 사용 가능한 버튼 ID 목록 (commonSf.js 주석 기준)

```
btn_approval     // 전자결재
btn_mail         // 메일전송
btn_formDown     // 양식다운로드
btn_print        // 출력
btn_printReport  // 보고서출력
btn_new          // 신규
btn_import       // 불러오기
btn_copy         // 복사
btn_tempSave     // 임시저장
btn_save         // 등록
btn_modify       // 수정
btn_remove       // 삭제
```

---

## 2. DataMap set 키 - 컴포넌트 id vs VO 필드명 ⚠️

### 문제

화면 진입 시 부서코드(`deptCd`) 값이 안 들어옴. 콘솔에는 사용자 정보 정상 출력.

```js
// ❌ 안 되는 코드
var userInfo = com.data.getUserInfr();
dma_rclcInrtMngReqVO.set("ibx_deptCd", userInfo.deptCd);
```

### 원인

`ibx_deptCd`는 화면의 인풋박스 **컴포넌트 id**. DataMap의 필드명이 아님.

### 해결

VO에 정의된 실제 필드명으로 변경:

```js
// ✅ 정상 동작
dma_rclcInrtMngReqVO.set("deptCd", userInfo.deptCd);
```

### 핵심 학습

- DataMap `set(key, value)`의 key는 **DataMap에 정의된 필드명** (= VO 필드명)
- 컴포넌트 id (`ibx_xxx`, `sbx_xxx`)와 혼동 시 **에러 없이 무시되어 디버깅 어려움**
- 인풋박스 컴포넌트의 `ref` 속성이 `data:dma_xxx.fieldNm` 형태로 VO 필드를 가리키는 구조 → 컴포넌트와 데이터는 분리

---

## 3. setCommonCode 그리드 컬럼 바인딩 ⭐

### 핵심 발견 — 공식 지원 문법

`commonScope.js` 함수 정의 주석(line 515)에서 발견:

```js
{ grpCd : "00024", compID : "grd_CommCodeSample:JOB_CD" }
```

**`compID`에 `그리드ID:컬럼ID` 형태로 지정하면 그리드 컬럼에 자동 바인딩됨.**

### Repricing 금리관리 적용

```js
var codeOptions = [
    { grpCd : "8905", compID : "grd_list:flctnInrtRclcYmdClssCd" }                    // 룩백데이트
  , { grpCd : "0001", compID : "grd_list:crtrInrt", sysCd : gcm.SYS_CD.TST }          // 기준금리구분
];
com.data.setCommonCode(codeOptions);
```

### 그리드 컬럼 정의 — displayMode 필수

```xml
<w2:column id="flctnInrtRclcYmdClssCd"
           inputType="select"
           displayMode="label"/>
```

`displayMode="label"` 빠지면 코드값(`0001`)이 그대로 노출됨. 코드명(`직전적용일`)을 표시하려면 필수.

### 다중 매핑 패턴 — 한 grpCd로 여러 컴포넌트 동시 바인딩

```js
{ grpCd : "0001", compID : "sbx_chgAfCrtrInrtClssCd2,grdResult:inrtClssCd", sysCd : gcm.SYS_CD.TST }
```

콤마 구분으로 셀렉트박스 + 그리드 컬럼 동시 바인딩 가능.

### 핵심 학습

- grpCd 하나만 세팅하면 해당 그룹 전체 코드가 자동 로드됨 (개별 코드값 일일이 안 넘겨도 됨)
- 단순 코드 변환이 아닌 별도 기준 적용이 필요한 경우(예: 선종 코드)는 codeOptions로 처리 불가 → 별도 변환 로직 필요

---

## 4. xf:select1 + compID 매핑 패턴

setCommonCode를 단독 select1에 적용하는 표준 패턴 (다른 화면 참고):

```xml
<xf:select1 id="sbx_micoClssCd"
            ref="data:adl_dma_clcmVO.micoClssCd"
            allOption=""
            appearance="minimal"
            chooseOption="true"
            disabled="true"
            disabledClass="w2selectbox_disabled"
            style="width:100%;"
            submenuSize="auto">
</xf:select1>
```

```js
var codeOptions = [
    { grpCd : "0018", compID : "sbx_micoClssCd" }
];
com.data.setCommonCode(codeOptions);
```

**핵심:** `xf:select1`은 itemset 직접 정의 없이도 compID 지정만으로 자동 주입됨. `ref`는 DataMap 필드와 연결, `compID`는 setCommonCode가 코드 리스트를 주입할 대상.

---

## 5. 그리드 expression 컬럼 — 행 내 연산

기준금리 + 조정금리 = Spread 자동 계산.

### 잘못된 시도들

```xml
<!-- ❌ 컬럼 id 직접 참조 -->
expression="crtrInrt + inrtSprdItrt"

<!-- ❌ {} 또는 #{} 래핑 -->
expression="{crtrInrt} + {inrtSprdItrt}"
expression="#{crtrInrt} + #{inrtSprdItrt}"

<!-- ❌ SUM 사용 (footer 전용) -->
expression="SUM(crtrInrt, inrtSprdItrt)"
```

전부 `crtrInrt is not defined` 에러 또는 미작동.

### 정답 — display() 함수 사용

```xml
<w2:column id="spread"
           inputType="expression"
           expression="display('crtrInrt') + display('inrtSprdItrt')"
           dataType="number"
           displayFormat="#,##0.00000%"/>
```

### 핵심 학습

- **body column에서 다른 컬럼 참조는 `display('컬럼ID')` 함수 필수**
- `SUM()` / `MIN()` / `MAX()` / `COUNT()` 는 footer/subtotal 전용 — body에서 미작동
- `datalist('컬럼ID')` — gridView와 매핑되지 않은 dataList 컬럼 값도 가져올 수 있음
- 소수점 정밀도가 중요한 경우 `dataType="bigDecimal"` 고려

---

## 6. 그리드 셀 변경 이벤트 — oneditend

캘린더 값 변경 시 조회 쿼리 호출.

### 잘못된 선택

```xml
<!-- ❌ onmodelchange는 DataMap/DataList 전체 변경 감지 → 너무 자주 발생 -->
ev:onmodelchange="..."
```

### 정답

```xml
<w2:gridView id="grd_list" ev:oneditend="scwin.grd_list_oneditend" ...>
```

```js
scwin.grd_list_oneditend = function(row, col, colId, value) {
    if (colId == "inrtCmptYmd") {  // 캘린더 컬럼만
        dma_xxxReqVO.set("inrtCmptYmd", value);
        sbm_selectXxx.submit();
    }
};
```

### 핵심 학습

- 파라미터 순서: `row`, `col`(인덱스), `colId`, `value`(새 값)
- 같은 그리드에 여러 편집 컬럼이 있으면 **`colId` 분기 필수**
- 편집 후 값이 변하지 않은 경우 호출 차단하려면 기존 값과 비교

---

## 7. 수정 버튼 동작 변경 — 즉시 저장 방식

기존 설계는 "수정 버튼 → 편집 모드 → 사용자 입력 → 별도 저장"이었으나, 사수 확인 결과 "수정 버튼 클릭 = 즉시 저장 실행"이 맞음.

```js
scwin.grd_list_oncellclick = function(row, col, colId) {
    if (colId != "intAmt") return;
    
    var today = com.date.getServerDateTime().substring(0, 8);
    var dl = $p.getComponentById("dataList1");
    var aplcnYmd = dl.getCellData(row, "aplcnYmd");
    
    if (today <= aplcnYmd) return;  // 미래 건 차단
    
    // 필수값 검증
    if (com.util.isEmpty(dl.getCellData(row, "inrtCmptYmd"))) {
        com.win._message(gcm.MESSAGE_TYPE.INFO, "금리산출일자는 필수입력입니다.");
        return;
    }
    if (com.util.isEmpty(dl.getCellData(row, "inrtSprdItrt"))) {
        com.win._message(gcm.MESSAGE_TYPE.INFO, "조정금리는 필수입력입니다.");
        return;
    }
    if (com.util.isEmpty(dl.getCellData(row, "mdfcnRsn"))) {
        com.win._message(gcm.MESSAGE_TYPE.INFO, "수정사유는 필수입력입니다.");
        return;
    }
    
    com.win._confirm("수정하시겠습니까?", function(yn) {
        if (yn) {
            scwin.selectedIndex = row;
            sbm_saveRepricingAplcn.action = "/api/sf/cs/cmsf/saveRepricingAplcn.do";
            sbm_saveRepricingAplcn.submit();
        }
    });
};
```

---

## 8. 회의 메모 — 금리/코드 표시 규칙

### 퍼센트 + 소수점 5자리

기준금리(`crtrInrt`), 조정금리(`inrtSprdItrt`), Spread(`spread`) 전부:

```xml
displayFormat="#,##0.00000%"
dataType="number"
```

### 조달번호/계약번호/유가증권번호 = 동일 컬럼

`bizClssCd` 변경 시 헤더명 자동 반영:
- `10340005` → 조달번호
- `10340001` → 유가증권번호
- `10340002` → 계약번호

이미 `submitdone`에서 `setHeaderValue("column75", headerNm)` 처리 완료.

### 선종 코드 — 별도 처리 필요

단순 코드 변환이 아니고 별도 기준에 따라 처리해야 함. 사수에게 기준 받은 후 작업 예정.

---

## 9. 국가필수선대 — 등록자명 표시 (TSM_USER_I JOIN)

### 문제

`FRST_REG_USER_ID`만 SELECT해서 화면에 사용자 ID(`026006`)가 그대로 표시됨.

### 해결 — 쿼리에 사용자 테이블 outer join

`selectNtnEsntlDsgnBrkd` 쿼리 (`NtnEsntlGrshSql.xml`):

```sql
SELECT B.NTN_ESNTL_GRSH_NO
     , A.BIZ_YR
     , A.GIVE_DSPR
     , B.DSGN_YMD
     , B.RMV_YMD
     , B.FRST_REG_DT
     , B.FRST_REG_USER_ID
     , U.USER_NM AS FRST_REG_USER_NM      -- ★ 추가
     , B.EXNT_CN
  FROM TPS_NTN_ESNTL_GRSH_H B
     , TPS_NTN_ESNTL_GRSH_SBSD_GIVE_B A
     , TSM_USER_I U                        -- ★ 추가
 WHERE 1=1
   AND A.NTN_ESNTL_GRSH_NO(+) = B.NTN_ESNTL_GRSH_NO
   AND SUBSTR(B.DSGN_YMD, 1,4) = A.BIZ_YR(+)
   AND U.USER_ID(+) = B.FRST_REG_USER_ID   -- ★ 추가 (outer join)
   AND B.NTN_ESNTL_GRSH_NO = #{ntnEsntlGrshNo}
   ...
```

### 추가 작업

- `NtnEsntlDsgnHistVO`에 `String frstRegUserNm` 필드 + getter/setter 추가
- 화면 그리드 컬럼 바인딩: `frstRegUserId` → `frstRegUserNm`
- 헤더 라벨: "등록자ID" → "등록자명"

### 핵심 학습

- 사용자 ID → 이름 변환은 **쿼리 JOIN 방식이 표준** (이 프로젝트 기준)
- 화면단에 별도 변환 헬퍼 함수는 없음
- outer join `(+)` 사용 — 사용자 정보 없는 경우에도 행 보존

---

## 10. Git pull 충돌 처리

`commonSf.js` 같은 공통파일을 로컬 수정한 상태에서 pull 시도하면:

```
Unable to pull when changes are present on your branch. 
The following files would be overwritten: WebContent/cm/js/commonSf.js
```

### GitHub Desktop 처리

해당 파일만 로컬 변경 무시하고 pull:

1. Changes 탭에서 해당 파일 우클릭 → **Discard changes...**
2. 확인 팝업에서 **Discard Changes** 클릭
3. 상단 **Pull origin** 클릭

다른 파일의 변경분은 유지된 채로 pull됨.

### Git Bash 명령어

```bash
git checkout -- WebContent/cm/js/commonSf.js
git pull
```

---

## 트러블슈팅 — 그리드 컬럼 width 안 먹는 이슈 (미해결)

공통 CSS `.wq_gvw .gridBodyDefault`가 디자인탭에서 설정한 width를 덮어씀.

### 시도한 것
- inline `style="width:150px"` → 적용 안 됨
- `style="width:150px !important;"` → 적용 안 됨

### 다음 시도 예정
- `width="150"` 속성(style 아님) 사용 — colgroup col에 직접 박힘
- 헤더/바디 컬럼 width 일치 여부 확인
- `grd_list.setColumnWidth("colId", 150)` JS 호출
- F12로 적용된 selector 및 computed 값 확인

---

## 핵심 교훈 요약

| 주제              | 교훈                                                                |
|-----------------|-------------------------------------------------------------------|
| DataMap set 키   | 컴포넌트 id(`ibx_xxx`) 아님. **VO 필드명** 사용. 잘못 쓰면 에러 없이 무시됨               |
| 공통코드 그리드 바인딩    | `compID: "그리드ID:컬럼ID"` 패턴. grpCd 하나로 그룹 전체 자동 로드                  |
| 공통코드 다중 매핑      | `compID: "sbx_xxx,grd_list:colId"` 콤마 구분으로 여러 컴포넌트 동시 바인딩          |
| displayMode     | `"label"` 필수. 빠지면 코드값 그대로 노출                                       |
| 그리드 expression  | body column에서는 `display('컬럼ID')` 사용. SUM/MIN/MAX는 footer 전용         |
| 셀 변경 이벤트        | `oneditend` (편집 완료 시점). `onmodelchange`는 부적합                       |
| 사용자 ID → 이름     | 쿼리 JOIN 방식이 표준. 화면단 변환 헬퍼 없음                                       |
| 공통버튼 제어         | `com.util.showBtnById()` / `setBtnDisableById()`. 공통파일 미수정 원칙       |
| Git pull 충돌     | GitHub Desktop에서 해당 파일 Discard 후 pull (다른 변경분 유지)                   |
| 회의 메모           | 금리값 퍼센트 + 소수점 5자리. 선종코드는 별도 기준 필요                                  |

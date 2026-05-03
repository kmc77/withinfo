# 2026-04-23_Repricing 수정버튼 활성화 제어 및 편집모드 구현

---

## 1. 샘플 화면 패턴 분석 (국가필수선대 등록)

다른 화면(`SFPS_CSBS007M.xml`)에서 `setScreenDisable` + `grp_right.setDisabled` 패턴으로 읽기전용/수정가능 상태를 제어하는 방식을 참고함.

```js
scwin.setScreenDisable = function(status) {
    switch (status) {
        case "load":
        case "select":
            grp_right.setDisabled(true);
            break;
        case "modify":
            grp_right.setDisabled(false);
            break;
    }
};
```

**Repricing과의 구조 차이:**
- 샘플: 폼 그룹 컨테이너(`grp_right`) 전체 단위 제어
- Repricing: 그리드 행 기반 → 행별 개별 셀 단위 제어 필요
- → `setDisabled("cell", row, colId, bool)` 사용

---

## 2. setDisabled API 시그니처

```js
// 그리드 전체
gridView1.setDisabled("grid", true);

// 특정 행 전체
gridView1.setDisabled("row", 0, true);

// 특정 컬럼 전체
gridView1.setDisabled("column", "colId", true);

// 특정 셀
gridView1.setDisabled("cell", rowIndex, "colId", true);
```

---

## 3. 수정 버튼 활성/비활성 제어 구조 확정

### 흐름
```
onpageload → grd_list.setDisabled("grid", true)
    ↓
조회(search) → sbm execute
    ↓
submitdone → setDisabled("grid", false)
           → 3개 편집 컬럼 setCellInputType readOnly: true (전체 행)
           → fn_setRowButtonState() → intAmt 날짜 조건 분기
    ↓
수정버튼 클릭(oncellclick) → 해당 행 3개 컬럼 readOnly: false
```

### 핵심: setDisabled vs setCellInputType readOnly 분리 이유
- `setDisabled`는 button 타입 컬럼 제어에 효과적 (회색 처리)
- text/calendar 컬럼은 `setCellInputType {readOnly: true/false}`로 제어
- `setDisabled("grid", false)` 호출 시 셀/컬럼 단위 disabled 설정도 전부 리셋됨 → 이후 컬럼 재잠금 필수

### CSS 확인
```css
/* disabled와 readonly 모두 동일한 회색 스타일 적용 */
.wq_gvw .gridBodyDefault.w2grid_default_disabled button,
.wq_gvw .gridBodyDefault.w2grid_default_readonly button {
    border: 1px solid #cdd0d3;
    cursor: auto;
    color: #999;
    background: #e7e7e7;
}
```

---

## 4. 최종 확정 코드

```js
// 페이지 최초 진입
scwin.onpageload = function() {
    grd_list.setDisabled("grid", true);

    scwin.sysdate = com.data.getPrcsCrtrYmd();
    dma_rclcInrtMngReqVO.set("rpcgAplcnEndYmd", scwin.sysdate);
    dma_rclcInrtMngReqVO.set("rpcgAplcnBgngYmd", com.date.addMonth(scwin.sysdate, -1));

    var codeOptions = [
        { grpCd: "8905", compID: "sbx_flctnInrtRclcYmdClssCd" },
        { grpCd: "0001", compID: "sbx_inrtClssCd", sysCd: gcm.SYS_CD.TST }
    ];
    com.data.setCommonCode(codeOptions);
};

// 조회 완료
scwin.sbm_selectRprcInrtList_submitdone = function(e) {
    grd_list.setDisabled("grid", false);

    // 3개 편집 컬럼 전체 행 잠금
    var cnt = grd_list.getRowCount();
    for (var i = 0; i < cnt; i++) {
        grd_list.setCellInputType(i, "inrtCmptYmd", {inputType: "calendar", readOnly: true});
        grd_list.setCellInputType(i, "inrtSprdItrt", {inputType: "text", readOnly: true});
        grd_list.setCellInputType(i, "mdfcnRsn", {inputType: "text", readOnly: true});
    }

    // 헤더명 변경 (조회 완료 후)
    var cd = dma_rclcInrtMngReqVO.get("bizClssCd");
    var headerNm = "";
    if (cd == "10340005")      { headerNm = "조달번호"; }
    else if (cd == "10340001") { headerNm = "유가증권번호"; }
    else if (cd == "10340002") { headerNm = "계약번호"; }
    grd_list.setHeaderValue("column75", headerNm);

    scwin.fn_setRowButtonState();

    if (com.util.isEmpty(scwin.selectedIndex)) {
        scwin.selectedIndex = 0;
    }
    dataList1.setRowPosition(scwin.selectedIndex);
};

// 날짜 조건에 따라 행별 수정버튼 활성/비활성
scwin.fn_setRowButtonState = function() {
    var today = com.date.getServerDateTime().substring(0, 8);
    var dl = $p.getComponentById("dataList1");
    var cnt = grd_list.getRowCount();
    for (var i = 0; i < cnt; i++) {
        var aplcnYmd = dl.getCellData(i, "aplcnYmd");
        if (today > aplcnYmd) {  // 현재 > 적용일자 → 수정버튼 활성
            grd_list.setDisabled("cell", i, "intAmt", false);
        } else {                  // 미래 → 비활성 (회색)
            grd_list.setDisabled("cell", i, "intAmt", true);
        }
    }
};

// 수정 버튼 클릭 (ev:oncellclick)
scwin.grd_list_oncellclick = function(row, col, colId) {
    if (colId == "intAmt") {
        var today = com.date.getServerDateTime().substring(0, 8);
        var dl = $p.getComponentById("dataList1");
        var aplcnYmd = dl.getCellData(row, "aplcnYmd");
        if (today <= aplcnYmd) {
            return; // 미래 날짜 → 차단
        }
        grd_list.setCellInputType(row, "inrtCmptYmd", {inputType: "calendar", readOnly: false});
        grd_list.setCellInputType(row, "inrtSprdItrt", {inputType: "text", readOnly: false});
        grd_list.setCellInputType(row, "mdfcnRsn", {inputType: "text", readOnly: false});
        grd_list.setFocusedCell(row, "inrtCmptYmd", true);
    }
};
```

---

## 5. 초기 조회 조건 변경

- 기존: 당월 1일 ~ 당일
- 변경: **1개월 전 당일 ~ 당일**

```js
// 오늘 2026-04-23 → 시작일 2026-03-23
dma_rclcInrtMngReqVO.set("rpcgAplcnBgngYmd", com.date.addMonth(scwin.sysdate, -1));
```

- `com.date.addMonth(date, n)`: 날짜 문자열(yyyyMMdd)에 n개월 가감
- 당월 01일이 아닌 동일 일자 기준으로 1개월 전 계산됨

---

## 6. 헤더명 변경 위치 이동

- 기존: `onmodelchange` (조회조건 변경 시마다 동작 → 조회 전에도 헤더 바뀜)
- 변경: `submitdone` (조회 완료 후 정확한 타이밍에 1회 동작)

---

## 7. 국가필수선대 해제일자 자동 세팅 보완 (SFPS_CSBS007M.xml)

### 기존
```js
if(dsgnYmd && rmvYmd.length > 4) {
    rmvYmd = dsgnYmd.substr(0,4) + "1231";
}
```

### 변경 - 해제일자가 비어있을 때도 말일 세팅
```js
if(dsgnYmd && (rmvYmd.length > 4 || com.util.isEmpty(rmvYmd))) {
    rmvYmd = dsgnYmd.substr(0, 4) + "1231";
}
```

---

## 8. 지정일자 유효성 실패 시 자동 세팅된 해제일자도 초기화

지정일자 유효성 실패(당해년도 외 입력) 시, 자동으로 세팅된 해제일자도 같이 초기화해야 함.

```js
if(!com.util.isEmpty(dsgnYmd) && dsgnYmd.substr(0,4) != thisYear) {
    com.win._message(gcm.MESSAGE_TYPE.INFO, "지정일자는 당해년도만 입력가능합니다.", function() {
        boxName.setValue("");
        dma_NtnEsntlGrshVO.set("rmvYmd", ""); // 해제일자 초기화
        ica_rmvYmd.setValue("");               // 화면 컴포넌트도 초기화
    });
    return;
}
```

---

## 9. Git Push 이슈

**증상:** Push 시 "Newer commits on remote" 팝업  
**원인:** 리모트 브랜치에 로컬에 없는 커밋 존재 (사수 or 타인이 같은 브랜치에 푸시)  
**해결:** Fetch → Pull로 리모트 커밋 병합 → Push

---

## 10. 미해결 이슈

- **수정 모드 시각적 표시**: 수정버튼 클릭 시 3개 컬럼이 동시에 편집 가능함을 명확히 보여주는 방법 미확정
  - `displayMode="always"` 시도 → 미적용
  - `setCellStyle` border 처리 시도 예정
- **국가필수선대 gHeader `req` dot 미표시**: 속성 설정은 확인됐으나 렌더링 안 됨 → 캐시 또는 다른 원인 추가 확인 필요

---

## 핵심 교훈 요약

| 구분 | 내용 |
|------|------|
| setDisabled 단위 | `grid` / `row` / `column` / `cell` 4가지 |
| button vs text/calendar 제어 분리 | button → setDisabled, text/calendar → setCellInputType readOnly |
| setDisabled("grid", false) 주의 | 하위 cell/column disabled 전부 리셋 → 이후 재잠금 필수 |
| 날짜 조건 | `today > aplcnYmd` = 현재가 적용일자보다 미래 = 수정 가능 |
| addMonth | `com.date.addMonth(yyyyMMdd, n)` — 동일 일자 기준 n개월 가감 |
| 헤더 변경 타이밍 | onmodelchange(X) → submitdone(O) |
| 콜백 내 초기화 | 얼럿 콜백 내에서 VO.set + 컴포넌트.setValue 동시에 처리 |

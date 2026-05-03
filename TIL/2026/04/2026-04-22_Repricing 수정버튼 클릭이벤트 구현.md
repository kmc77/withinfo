# 2026-04-22_Repricing 수정버튼 클릭이벤트 구현

## 오늘 한 것
- WebSquare 그리드 컴포넌트 접근 방법 확립
- 수정 버튼 클릭 이벤트(`ev:onrightbuttonclick`) 연결
- 적용일자 조건 기반 버튼 활성화/비활성화 로직 구현
- 수정 가능 컬럼 3개 편집 모드 전환

---

## WebSquare 컴포넌트 접근

```js
// ❌ 직접 접근 불가
grd_list.getCount(); // is not a function

// ✅ $p.getComponentById 사용
var grid = $p.getComponentById("grd_list");
var dl = $p.getComponentById("dataList1");
var cnt = grid.getRowCount();
```

- dataList id 확인 방법: 그리드뷰 태그의 `dataList="data:dataList1"` 속성 확인

---

## dataList 셀 값 접근

```js
// ❌ getValue 미지원
dl.getValue(i, "aplcnYmd");

// ✅ getCellData 사용
var aplcnYmd = dl.getCellData(i, "aplcnYmd");
```

---

## 서버 날짜 취득 (yyyyMMdd)

```js
var today = com.date.getServerDateTime().substring(0, 8);
// new Date() 사용 금지 → Date 객체라 문자열 비교 불가
```

---

## 그리드 버튼 클릭 이벤트

### XML 설정
```xml
<!-- 그리드뷰 태그에 이벤트 추가 -->
<w2:gridView id="grd_list" ev:onrightbuttonclick="scwin.grd_list_onrightbuttonclick" ...>

<!-- gBody 버튼 컬럼 설정 -->
<w2:column id="intAmt" readOnly="" textAlign="right" value="수정"
    displayMode="edit" width="85" inputType="button" style="height:28px;">
</w2:column>
```

**주의사항:**
- `displayMode="label"` → 버튼 렌더링 안 됨. 반드시 `displayMode="edit"`
- `setCellInputTypeCustom="true"` → 빨간 표시 뜨고 동작 안 함 → 미적용

### JS 핸들러
```js
scwin.grd_list_onrightbuttonclick = function(row, col, colId) {
    if (colId == "intAmt") {
        var today = com.date.getServerDateTime().substring(0, 8);
        var grid = $p.getComponentById("grd_list");
        var dl = $p.getComponentById("dataList1");
        var aplcnYmd = dl.getCellData(row, "aplcnYmd");

        // 적용일자가 오늘 이전이면 수정 불가
        if (aplcnYmd <= today) {
            return;
        }

        // 수정 가능 컬럼 3개 활성화
        grid.setCellInputType(row, "inrtCmptYmd", {inputType: "text"});
        grid.setCellInputType(row, "inrtSprdItrt", {inputType: "text"});
        grid.setCellInputType(row, "mdfcnRsn", {inputType: "text"});
        grid.setFocusedCell(row, "inrtCmptYmd", true);
    }
};
```

---

## submitdone 날짜 조건 활성화/비활성화

```js
scwin.sbm_selectRprcInrtList_submitdone = function(e) {
    var today = com.date.getServerDateTime().substring(0, 8);
    var grid = $p.getComponentById("grd_list");
    var dl = $p.getComponentById("dataList1");
    var cnt = grid.getRowCount();

    for (var i = 0; i < cnt; i++) {
        var aplcnYmd = dl.getCellData(i, "aplcnYmd");
        if (aplcnYmd > today) {
            grid.setCellInputType(i, "intAmt", {inputType: "button"});
        } else {
            grid.setCellInputType(i, "intAmt", {inputType: "readonly"});
        }
    }

    if (com.util.isEmpty(scwin.selectedIndex)) {
        scwin.selectedIndex = 0;
    }
    dataList1.setRowPosition(scwin.selectedIndex);
};
```

---

## setCellInputType 동작 특이사항

- 파라미터: `(rowIndex, colId, {inputType: "text"|"readonly"|"button"})` 객체 형태
- 같은 row에 연속 3번 호출 시 마지막만 적용되는 케이스 존재 → `setTimeout` 분리 시도 필요
- XML에서 해당 컬럼 `readOnly="true"` 있으면 JS에서 바꿔도 안 먹힘 → 제거 필요

---

## 날짜 문자열 비교

- `yyyyMMdd` 형태 문자열은 대소비교 바로 가능
- `"20260626" > "20260422"` → `true` ✅

---

## 미완료 (다음 세션)

- `setCellInputType` 3개 동시 적용 문제 최종 확인
- submitdone 버튼 활성화/비활성화 동작 검증
- 저장 로직 구현 (`saveRepricingAplcn`)
- Git 커밋/푸시

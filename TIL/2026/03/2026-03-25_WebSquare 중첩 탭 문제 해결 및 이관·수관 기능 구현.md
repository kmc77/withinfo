# 2026-03-25_WebSquare 중첩 탭 문제 해결 및 이관·수관 기능 구현

## 개요
관리자 이수관(SP800M01) 화면에서 이관/수관 탭 UI 구현 및 권한 이관 기능 전체 개발.  
WebSquare 중첩 탭 렌더링 버그를 우회하는 방법을 찾아내고, 체크박스 다중 선택 이관까지 완성.

---

## 1. 문제: WebSquare 중첩 tabControl 렌더링 버그

### 증상
- WebSquare 디자이너에서는 탭 안에 좌우 레이아웃이 정상 배치
- 실제 브라우저에서는 이관/수관 패널이 탭 **밖(위)**에 렌더링
- 탭 버튼만 화면 하단에 떠 있는 구조

### 원인
페이지 자체가 `tac_layout`이라는 최상위 `w2:tabControl`로 동작하는 구조인데,  
그 안에 `w2:tabControl`을 또 넣으면 **중첩 tabControl** 상태가 됨.  
WebSquare가 내부 탭의 content 위치를 외부 탭 기준으로 계산해버려서 content가 탈출함.

```
tac_layout (최상위 페이지 탭)
  └─ SP800M01.xml 화면
       └─ tac_transfer (내부 탭) ← 탭 안에 탭 → 렌더링 버그 발생
```

### 해결
`w2:tabControl` 완전 제거 후 **버튼 + show/hide로 탭 직접 구현**

```xml
<!-- 탭 버튼 직접 구현 -->
<xf:group id="grp_tab_btns" style="margin-bottom:5px;">
    <xf:trigger id="btn_tab_from" class="btn_cm" type="button" ev:onclick="scwin.showTab('from')" style="width:70px;">
        <xf:label><![CDATA[이관]]></xf:label>
    </xf:trigger>
    <xf:trigger id="btn_tab_to" class="btn_cm" type="button" ev:onclick="scwin.showTab('to')" style="width:70px;">
        <xf:label><![CDATA[수관]]></xf:label>
    </xf:trigger>
</xf:group>

<!-- 패널 show/hide로 탭 전환 -->
<xf:group id="grp_from_panel" class="lybox" style="overflow:hidden; width:100%;">
    <!-- 이관/수관 패널 내용 -->
</xf:group>
<xf:group id="grp_to_panel" style="display:none;">
</xf:group>
```

```javascript
scwin.showTab = function(tab) {
    if (tab === 'from') {
        grp_from_panel.show();
        grp_to_panel.hide();
        btn_tab_from.setStyle("background-color", "#336699");
        btn_tab_from.setStyle("color", "#ffffff");
        btn_tab_to.setStyle("background-color", "");
        btn_tab_to.setStyle("color", "");
    } else {
        grp_from_panel.hide();
        grp_to_panel.show();
        btn_tab_to.setStyle("background-color", "#336699");
        btn_tab_to.setStyle("color", "#ffffff");
        btn_tab_from.setStyle("background-color", "");
        btn_tab_from.setStyle("color", "");
    }
    // show() 후 display:block → flex 복원 (lybox가 flex 기반이라 필요)
    setTimeout(function() {
        $("[id$='grp_from_panel']").css("display", "flex");
        dlt_box.resize();
        grd_gridView1.resize();
    }, 50);
};
```

> **핵심 교훈**: WebSquare에서 페이지 자체가 tabControl일 때 내부에 tabControl을 중첩하면 렌더링이 깨짐. show/hide 방식으로 우회.

---

## 2. 문제: lybox show() 후 좌우 레이아웃 깨짐

### 원인
`ly_column col_5`가 **flex 기반** CSS 클래스인데,  
WebSquare의 `show()` 메서드가 `display:block`으로 덮어써버림 → flex 레이아웃 소멸

### 해결
`show()` 후 setTimeout으로 `display:flex` 강제 복원 + 그리드 resize 트리거

```javascript
setTimeout(function() {
    $("[id$='grp_from_panel']").css("display", "flex");
    dlt_box.resize();
    grd_gridView1.resize();
}, 50);
```

---

## 3. 그리드 체크박스 추가 (다중 선택 이관)

### 문제: 그리드 전체 readOnly → 체크박스 disabled
그리드에 `readOnly="true"` 설정 시 WebSquare가 체크박스를 `disabled` 처리함  
→ `input[type="checkbox"]:disabled + .w2checkbox_label:before` CSS 적용되어 비활성 스타일

### 해결
그리드 레벨 `readOnly` 제거, 컬럼 레벨에서 개별 제어

```xml
<w2:gridView autoFit="allColumn" class="wq_gvw" dataList="data:dlt_from_auth"
    defaultCellHeight="20" focusMode="row" id="dlt_box"
    rowNumHeaderValue="No" rowNumVisible="true" rowNumWidth="50"
    rowStatusHeaderValue="상태" rowStatusVisible="true" rowStatusWidth="50"
    style="height:150px;" visibleRowNum="all">
    <w2:header id="hdr_from" style="">
        <w2:row id="row_from_h" style="">
            <w2:column displayMode="label" id="col_from_cd" value="화면코드" width="82"></w2:column>
            <w2:column displayMode="label" id="col_from_nm" value="화면명" width="69"></w2:column>
            <!-- header 체크박스: displayMode="label" -->
            <w2:column inputType="checkbox" id="chk" value="" displayMode="label"
                checkboxLabel="선택" fixColumnWidth="true" width="50"></w2:column>
        </w2:row>
    </w2:header>
    <w2:gBody id="gbody_from" style="">
        <w2:row id="row_from_b" style="">
            <w2:column displayMode="label" headers="col_from_cd" id="PROGRAM_CD" inputType="text" width="82"></w2:column>
            <w2:column displayMode="label" headers="col_from_nm" id="PROGRAM_NM" inputType="text" width="69"></w2:column>
            <!-- body 체크박스: displayMode="edit" 이어야 클릭 가능 -->
            <w2:column headers="chk" id="CHK_YN" inputType="checkbox"
                displayMode="edit" blockSelect="false" fixColumnWidth="true" width="50"></w2:column>
        </w2:row>
    </w2:gBody>
</w2:gridView>
```

> **핵심**: header 체크박스는 `displayMode="label"`, body 체크박스는 `displayMode="edit"`

dataList에도 CHK_YN 컬럼 추가 필요:
```xml
<w2:column id="CHK_YN" name="선택" dataType="text"></w2:column>
```

---

## 4. 이관 기능 구현 (다중 선택)

```javascript
scwin.btn_transfer_onclick = function(e) {
    var checkedRows = dlt_box.getCheckedRowIndexes();

    from_emp = dm_from_emp.get("EMP_NO");
    if (!from_emp || from_emp == "") { alert("이관자를 선택해주세요"); return; }

    to_emp = dm_to_emp.get("EMP_NO");
    if (!to_emp || to_emp == "") { alert("수관자를 선택해주세요"); return; }

    if (from_emp == to_emp) { alert("이관자와 수관자가 같습니다"); return; }

    if (checkedRows.length === 0) { alert("이관할 권한을 선택해주세요."); return; }

    var nmList = [];
    for (var i = 0; i < checkedRows.length; i++) {
        nmList.push(dlt_from_auth.getCellData(checkedRows[i], "PROGRAM_NM"));
    }

    if (confirm("권한을 이관하시겠습니까?\n" + nmList.join(", ") + "\n" + from_emp + " -> " + to_emp)) {
        for (var k = 0; k < checkedRows.length; k++) {
            dm_transfer.set("PROGRAM_CD", dlt_from_auth.getCellData(checkedRows[k], "PROGRAM_CD"));
            dm_transfer.set("PROGRAM_NM", dlt_from_auth.getCellData(checkedRows[k], "PROGRAM_NM"));
            dm_transfer.set("FROM_EMP_NO", from_emp);
            dm_transfer.set("TO_EMP_NO", to_emp);
            com.sbm.execute(sbm_transfer);
        }
    }
};
```

> **수관 그리드는 선택 불필요** — 수관자가 현재 보유한 권한 확인용이지 선택용이 아님

---

## 5. 서버단 구현

### Controller
```java
@RequestMapping("/with/sample/transfer")
public @ResponseBody Map<String, Object> transfer_Auth(@RequestBody Map<String, Object> param) {
    Result result = new Result();
    try {
        Map dmTransfer = (Map) param.get("dm_transfer");

        int cnt = transfer.checkAuth(dmTransfer);
        if (cnt > 0) {
            result.setMsg(result.STATUS_ERROR, "이미 보유한 권한입니다.");
            return result.getResult();
        }

        transfer.transfer_Auth(dmTransfer);
        transfer.addHistory(dmTransfer);
        result.setMsg(result.STATUS_SUCESS, "이관되었습니다.");
    } catch (Exception ex) {
        ex.printStackTrace();
        result.setMsg(result.STATUS_ERROR, "이관 실패");
    }
    return result.getResult();
}
```

### Mapper
```xml
<!-- 중복 체크 -->
<select id="checkAuth" parameterType="map" resultType="int">
    SELECT COUNT(*)
    FROM EMP_PROGRAM_AUTH
    WHERE EMP_NO = #{TO_EMP_NO}
    AND PROGRAM_CD = #{PROGRAM_CD}
</select>

<!-- 권한 INSERT -->
<insert id="transfer_Auth" parameterType="map">
    INSERT INTO EMP_PROGRAM_AUTH (EMP_NO, PROGRAM_CD, PROGRAM_NM)
    VALUES(#{TO_EMP_NO}, #{PROGRAM_CD}, #{PROGRAM_NM})
</insert>

<!-- 이력 조회 (서브쿼리로 이름 변환) -->
<select id="searchHistory" parameterType="map" resultType="map">
    SELECT
        TRANSFER_NO,
        (SELECT EMP_NM FROM EMPLOYEE WHERE EMP_NO = H.FROM_EMP_NO) AS FROM_EMP_NM,
        (SELECT EMP_NM FROM EMPLOYEE WHERE EMP_NO = H.TO_EMP_NO) AS TO_EMP_NM,
        PROGRAM_CD,
        PROGRAM_NM,
        TRANSFER_DT
    FROM EMP_AUTH_TRANSFER H
</select>
```

> **중복 처리 전략**: INSERT IGNORE 대신 서버단에서 COUNT 후 에러 메시지 반환.  
> 프론트는 우회 가능하므로 서버단 체크가 필수.

---

## 6. 이력 조회 — 페이지 진입 시 자동 조회

```xml
<xf:submission id="sbm_searchHistory" ref='data:json,[]'
    target='data:json,[{"id":"dlt_transfer_history"}]'
    action="/with/sample/searchHistory" method="post" ...>
</xf:submission>
```

```javascript
scwin.onpageload = function() {
    scwin.showTab('from');
    com.sbm.execute(sbm_searchHistory);
};
```

`ref='data:json,[]'` → body가 없으므로 Controller에서 `@RequestBody` 제거 필요

```java
@RequestMapping("/with/sample/searchHistory")
public @ResponseBody Map<String, Object> searchHistory() { // @RequestBody 없음
    ...
    result.getResult().put("dlt_transfer_history", list); // key가 target id와 일치해야 바인딩
    ...
}
```

---

## 핵심 정리

| 상황 | 원인 | 해결 |
|------|------|------|
| 탭 안 콘텐츠가 탭 밖에 렌더링 | 페이지가 이미 tabControl → 중첩 tabControl 버그 | w2:tabControl 제거, show/hide로 탭 직접 구현 |
| show() 후 좌우 레이아웃 깨짐 | show()가 display:block으로 덮어씀 | setTimeout으로 display:flex 복원 |
| 체크박스 disabled | 그리드 readOnly="true" | 그리드 readOnly 제거, 컬럼별 displayMode="label"로 제어 |
| body 체크박스 클릭 안 됨 | displayMode="label" | displayMode="edit"으로 변경 |
| submission 400 에러 | ref=[] → body 없는데 @RequestBody 사용 | @RequestBody 제거 |
| 이관 이력 이름 코드로 표시 | FROM/TO_EMP_NO 그대로 출력 | 서브쿼리로 EMP_NM 조회 후 AS 별칭 사용 |

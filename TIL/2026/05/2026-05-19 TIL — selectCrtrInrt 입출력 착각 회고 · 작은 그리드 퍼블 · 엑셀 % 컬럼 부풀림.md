# 2026-05-19 TIL — selectCrtrInrt 입출력 착각 회고 · 작은 그리드 퍼블 · 엑셀 % 컬럼 부풀림

## 오늘 한 작업
1. `saveRepricingAplcn` 저장 로직 접근 방향 확정
2. `selectCrtrInrt` 입출력 방향 착각 회고 (4/29~5/18 디버깅 회고)
3. OB 날짜 변경 시 기준금리 부풀림 버그 원인 추정 (미해결)
4. 두 번째 탭 작은 그리드 퍼블 적용 (방향 확정, 적용 미완)
5. 엑셀 다운로드 % 컬럼 부풀림 (미해결, 퍼블리셔 문의)

---

## 1. saveRepricingAplcn 저장 로직 접근

### 결론
기존 INSERT용 저장 로직(`saveRepricingAplcn`)이 Controller → Service → Mapper 체인까지 깔려 있음. **Mapper 호출만 INSERT → UPDATE(`updateRepricingAplcn`)로 교체하면 됨.** 새로 연결할 필요 없음.

### 확인 필요 사항 (실제 작업 진입 시)
- 기존 Service가 단건 INSERT 루프인지, `foreach` 다건인지
- 변경분 추출 방식 (`getModifiedData()` / `getSaveData()` 등)
- VO 타입이 `RclcInrtMngListVO` 단건인지 리스트인지
- 5/18에 만든 `updateRepricingAplcn`은 `parameterType="...RclcInrtMngListVO"` 단건 → 기존이 다건이면 시그니처 조정 필요

### 메모
- 5/18 `updateRepricingAplcn` SET절: `INRT_CRTR_YMD`, `INRT_CMPT_YMD`, `FLCTN_INRT_RCLC_YMD_CLSS_CD`, `CMPT_INRT_CLSS_CD`, `APLCN_INRT`, `ADJ_INRT`, `MDFCN_RSN_CN`, `CHRG_DEPT_CD`, `LAST_CHG_USER_ID`, `LAST_CHG_DT`
- 화면에서 편집 가능한 3개 컬럼: `inrtCmptYmd`, `inrtSprdItrt`, `mdfcnRsn`
- 화면 컬럼과 SET절 컬럼명 매핑 차이(예: `inrtSprdItrt` → `ADJ_INRT`?) 발생 시 검증 필요. 단, 4/23 → 5/18로 작업이 얹어진 결과로 컨벤션상 의도된 매핑일 가능성도 있음 — 단정 X

---

## 2. selectCrtrInrt 입출력 방향 착각 회고

### 헤맨 원인

**(1) 비즈니스 로직 미숙지**

`selectCrtrInrt`의 입출력 방향을 거꾸로 잡았음.

| 구분 | 실제 | 착각 |
|---|---|---|
| 입력 | Observation Date(`inrtCmptYmd`) | 출력으로 착각 |
| 출력 | 기준금리(`inrt`) | 입력으로 착각 |

`resVO`에서 `inrtCmptYmd`를 꺼내려 했고, `CrtrInrtResVO`에 필드를 추가하고, `selectCrtrInrt` 내부에서 `result.setInrtCmptYmd()`를 세팅하려고 했음.

**(2) 잘못된 진단**

로직을 모르니까 "리턴 VO에 필드가 없어서 null이다" → "필드 추가하면 된다"는 방향으로 흘러감. 실제 문제는 애초에 꺼내려는 값이 틀렸던 것 — VO엔 문제가 없었음.

### 실제 정답
```
Observation Date(inrtCmptYmd) → selectCrtrInrt 입력값
기준금리(inrt)               → selectCrtrInrt 리턴값
```

서버 루프(`selectRepricingTrgt`)에서 세팅할 코드는 한 줄:
```java
vo.setCrtrInrt(resVO.getInrt());
```

### 교훈
**검증된 기존 호출부가 함수의 입출력 계약을 다 보여준다.**

`oneditend`에 이미 `selectCrtrInrt`(또는 `sbm_selectObservationDateInrt`) 호출 코드가 있었음. 거기서 무엇을 넣고 리턴에서 무엇을 쓰는지만 봤어도 방향이 바로 잡혔음.

> **같은 함수의 동작 사례가 코드에 이미 있을 땐 추론하지 말고 그걸 스펙으로 삼는다.**

증상 대응 함정도 같이 정리:
- "VO에 필드 없네 → 추가하자" 식 증상 대응 금지
- 입출력 방향부터 검증

---

## 3. OB 날짜 변경 시 기준금리 부풀림 버그 (미해결)

### 증상
- Compounded SOFR 행: OB 날짜 변경 시 기준금리 재계산 정상
- **그 외 기준금리(Term SOFR 등) 행: OB 날짜를 동일/다른 값으로 변경하면 기준금리가 부풀려져서(오른 값으로) 표시됨**

### grd_list_oneditend 실제 코드 (CWMI_SFCM201M.xml 274~292줄)
```js
scwin._isSettingCellData = false;
scwin.grd_list_oneditend = function(row, col, value, colId) {
    if(scwin._isSettingCellData) return;
    if(col != grd_list.getColumnIndex("inrtCmptYmd", col)) return;
    scwin._isSettingCellData = true;

    dma_rclcInrtMngReqVO.set("inrtCmptYmd", dataList1.getCellData(row, "aplcnYmd"));
    dma_rclcInrtMngReqVO.set("rpmtYmd",     dataList1.getCellData(row, "inrtCrtrYmd"));
    dma_rclcInrtMngReqVO.set("inrtClssCd",  dataList1.getCellData(row, "cmptInrtClssCd"));
    
    dma_rclcInrtMngReqVO.set("flctnInrtRclcYmdClssCd", dataList1.getCellData(row, "flctnInrtRclcYmdClssCd"));
    dma_rclcInrtMngReqVO.set("intAplcnWaysClssCd", "89030003");
    dma_rclcInrtMngReqVO.set("adltPrcsMethClssCd", "11020002");
    dma_rclcInrtMngReqVO.set("ntnAreaCd", "KR");
    
    scwin._isSettingCellData = false;
    
    scwin.selectedIndex = row;
    com.sbm.execute(sbm_selectObservationDateInrt);
};
```

### 의심 라인 (281줄)
```js
dma_rclcInrtMngReqVO.set("inrtClssCd", dataList1.getCellData(row, "cmptInrtClssCd"));
```

`inrtClssCd`(이자구분코드)에 `cmptInrtClssCd`(산출이자구분코드 — Compounded/Term SOFR 구분값)를 넣고 있음. 핸드오버 서버 코드 `reqVO.setInrtClssCd(vo.getCmptInrtClssCd())`와 동일 패턴.

- Compounded SOFR 행: 매핑이 우연히 맞아서 정상
- 그 외 기준금리: `inrtClssCd`에 엉뚱한 코드가 들어가 `selectCrtrInrt`가 다른 산출식 타고 값 부풀림 → "오른 값으로 이상하게"의 직접 원인 후보

### 의심 라인 (279줄)
```js
dma_rclcInrtMngReqVO.set("inrtCmptYmd", dataList1.getCellData(row, "aplcnYmd"));
```

이름은 `inrtCmptYmd`(Observation Date)인데 값은 `aplcnYmd`(적용일자)를 꺼냄. 사용자가 셀에서 방금 바꾼 OB 날짜가 아니라 적용일자를 보내고 있음. 의도면 주석이 있어야 하는데 없음.

### sbm 콜백 (298~307줄)
```js
scwin.sbm_selectObservationDateInrt_submitdone = function(e) {
    var inrt = e.responseJSON.inrt;
    if(com.util.isEmpty(inrt)) {
        //com.win.message("EZZ0001", null);
        return;
    }
    dataList1.setCellData(scwin.selectedIndex, "crtrInrt", inrt);
};
```

### 확정 전 확인 필요
1. `selectCrtrInrt` 서버/매퍼 SQL에서 `inrtClssCd`로 어떻게 분기 타는지 확인 → 281줄이 범인인지 확정
2. Term SOFR 행의 실제 `cmptInrtClssCd` vs 정상 `inrtClssCd`에 들어가야 할 값 확인

### 수정 방향 (가설 확정 시)
```js
dma_rclcInrtMngReqVO.set("inrtClssCd", dataList1.getCellData(row, "inrtClssCd"));
```

단, `dataList1`에 `inrtClssCd` 컬럼이 실제 있는지 확인 필요. 없으면 hidden 컬럼 추가 필요.

---

## 4. 두 번째 탭 작은 그리드 퍼블 적용

### 현재 화면 구조 (CWMI_SFCM201M.xml)
```
lybox > ly_column
 ├─ dfbox (타이틀 + 엑셀다운로드)
 └─ tabControl
     ├─ tabs2 "이차보전금상세"
     ├─ tabs3 "선사별 이차보전금"
     ├─ content2 → grd_IdstAmtCmpt    (큰 그리드, class="wq_gvw")
     └─ content3 → grd_IdsAmtCmptC    (작은 그리드 적용 대상)
```

- `tabControl`이 panel을 **직계 자식 인덱스로 매칭**함 → `content3`은 반드시 `tabControl` 직계 자식 유지해야 함
- 탭 전환 동작 자체는 정상

### 시도와 실패
**시도 A: `content3`을 `<xf:group class="gvwbox">`로 외부 래핑**
→ 탭 클릭 시 `Cannot read properties of null (reading 'style')`
→ 원인: `gvwbox` 래퍼가 `tabControl` 직계 관계 깨뜨려서 panel 매칭 실패

**시도 B: `w2:content`에 `class="gvwbox"` 직접 추가**
→ 시각 변화 0

**시도 C: `w2:gridView`에 class 추가 검토**
→ 이미 `class="wq_gvw"` 존재. `gvwbox`는 grid에 박는 class 아님

### 결정적 발견 — CSS 분석

**test3.css 3줄:**
```css
.gvwbox{}
```

**`gvwbox`는 선언만 있고 속성이 비어 있음. 시각 효과 0.** 어떤 위치에 박든 영향 없음.

### 진짜 메커니즘 — contents.css 342~352줄
```css
.lybox      { display: table; width: 100%; table-layout: fixed; }
.ly_column  { display: table-cell; padding: 0 10px; ... }
.lybox.col2 .ly_column { width: 50%; }
.lybox.col3 .ly_column { width: 33.333%; }
.lybox.col4 .ly_column { width: 25%; }
```

추가로 `.col_2 ~ .col_8 { width: 20% ~ 80% !important; }`

**CSS table-cell 레이아웃.** `lybox` 안에 `ly_column` 형제가 N개 있으면 자동 N등분. 빈 `ly_column` 형제가 좌우 여백 역할.

### 적용 방향 (확정)
`content3`을 `tabControl` 직계 유지하면서 **안쪽에서** `lybox` 레이아웃 구성:

```xml
<w2:content id="content3" ...>
    <xf:group class="lybox" id="">
        <xf:group class="ly_column" id="">
            <w2:gridView id="grd_IdsAmtCmptC" ...>...</w2:gridView>
        </xf:group>
        <xf:group class="ly_column" id="">
            <!-- 빈 여백 -->
        </xf:group>
    </xf:group>
</w2:content>
```

### 교훈
- WebSquare `tabControl`은 panel을 **직계 자식 인덱스**로 매칭. 외부 래퍼 추가 시 깨짐
- CSS 디버깅은 추측 금지. 실제 CSS 파일 까서 정의 확인 (`.gvwbox{}` 비어 있음 = 효과 0)
- 폐쇄망 검색: Eclipse Ctrl+H File Search (workspace `*.css` 범위)

---

## 5. 엑셀 다운로드 % 컬럼 부풀림 (미해결)

### 화면
이차보전 금액 검증내역 (30401) / `CWMI_SFCM201M.xml` / 그리드 `grd_IdstAmtCmpt`

### 증상
같은 그리드 안의 % 컬럼 2개 중 한쪽만 엑셀 다운로드 시 부풀림 발생.

| 컬럼 | 의미 | DB raw | 화면 표시 | 엑셀 표시 |
|---|---|---|---|---|
| `assrRat` | 보증요율 | 0.013, 0.015 (소수) | 0.01%, 0.01% | 정상 |
| `intRto` | 이차보전율 | 9.96, 6, 2 (정수대) | 9.96%, 6.00%, 2.00% | **996.00%, 600.00% 등 ×100 부풀림** |

### 컬럼 속성 비교 (둘 다 동일)
```
dataType="number"  displayFormat="##.00%"  inputType="text"
readOnly="true"    displayMode="label"
```

차이는 `textAlign`/`width`/`style` 정도. `textAlign="right"` 제거 후에도 동일 → 시각 속성은 원인 아님.

### displayFormat 동작 (확인됨)
`##.00%`는 **raw 값에 `%` 기호만 붙임 (×100 없음)**.
- raw 0.013 → 화면 `0.01%` (소수점 2자리 절삭)
- raw 9.96 → 화면 `9.96%`

화면은 둘 다 일관되게 정상.

### 엑셀 다운로드 옵션 (현재)
```js
var option = {
    type : 1,                  // 보이는 값
    autoSizeColumn : "true",
    useDataFormat : "false"
};
com.data.downloadGridViewExcel(targetGrid, option);
```

### 시도한 방향과 결과

| 시도 | 결과 |
|---|---|
| `type:1` + `useDataFormat:false` (현재) | `intRto`만 부풀림 |
| `type:0` + `useDataFormat:true` | `%` 기호 사라짐, 숫자만 |
| `textAlign="right"` 제거 | 동일 부풀림 |
| dataType `number` ↔ `bigDecimal` 변경 | 동일 |
| 다운로드 직전 해당 컬럼 ÷ 100 가공 | 라이브 그리드 흔드는 억지 방법, 비동기 타이밍 위험 — 미적용 |

### 근본 원인 (추정)
**DB raw 데이터 스케일이 두 컬럼 간 비일관:**
- `assrRat` = 소수형 (0.013 = 1.3%)
- `intRto` = 정수대형 (9.96 = 9.96%)

WebSquare 화면 렌더링은 `##.00%`가 raw에 `%`만 붙이므로 두 케이스 모두 우연히 정상 표시. 하지만 엑셀 다운로드(`type:1`)에서 `intRto` raw 9.96이 엑셀 백분율 셀 서식과 만나면서 ×100되어 996.00%로 표시됨.

### 미확인 사항
- 엑셀 셀 클릭 시 수식 입력줄의 raw 값이 무엇인지 미확인 (셀 표시값 말고 실제 저장된 값)
- 이게 확정되면 원인 분기 가능:
  - 셀 raw `9.96` + 백분율 서식 → 표시 996% : 엑셀 셀 서식 문제
  - 셀 raw `996` → WebSquare 다운로드 로직 자체가 ×100 처리

### 가능한 해결 방향
1. **DB 데이터 정합화 (근본)** — `intRto`도 소수형(0.0996)으로 통일. 영향 범위 큼
2. **displayFormat 변경 (현실)** — `##.00%` → `##.00`으로 바꾸고 컬럼 헤더에 `(%)` 표기. 화면도 엑셀도 숫자만 일관 표시
3. **다운로드 직전 가공 (억지)** — 그리드 셀 흔드는 방식, 비추
4. **퍼블리셔 컨펌 (보고 완료)** — 같은 displayFormat인데 한 컬럼만 엑셀에서 부풀림 발생 원인 문의 중

### 사수/기획 컨펌 필요
- 엑셀 다운로드 시 `intRto` 컬럼이 화면과 동일하게 `9.96%`로 떨어져야 하는지, 숫자만 `9.96`이어도 되는지
- DB raw 스케일 비일관(`assrRat` 소수 vs `intRto` 정수대)이 의도된 것인지, 마이그레이션 영역인지

---

## 잔여 작업 (다음 세션)
1. `saveRepricingAplcn` Mapper 호출 INSERT → UPDATE 교체, 시그니처 정합성 확인
2. 저장 후 재조회 or 행 갱신, 수정모드 시각 표시
3. OB 날짜 변경 시 기준금리 부풀림 — `selectCrtrInrt` 매퍼 SQL 확인, 281줄/279줄 매핑 수정
4. 두 번째 탭 작은 그리드 — `content3` 내부 `lybox > [ly_column, ly_column]` 구조 적용, 비율 결정
5. 엑셀 % 부풀림 — 퍼블리셔 답변 대기, 엑셀 셀 raw 값 확인

---

## 핵심 교훈 정리

| 구분 | 내용 |
|------|------|
| 함수 입출력 | 사례 코드(`oneditend`)가 곧 스펙. 추론 금지 |
| 증상 대응 | "VO에 필드 없네 → 추가" 식 금지. 입출력 방향부터 검증 |
| 시간순 작업 | 4/23 → 5/18 작업이 얹어진 것을 모순으로 단정 금지 |
| CSS 디버깅 | 추측 금지. CSS 파일 까서 정의 직접 확인 |
| WebSquare tabControl | panel 매칭은 **직계 자식 인덱스**. 외부 래퍼 추가 시 깨짐 |
| `.gvwbox{}` | 빈 선언이라 효과 0. 작은 그리드 효과는 `lybox` table-cell + 빈 `ly_column` 형제 |
| 폐쇄망 검색 | Eclipse Ctrl+H File Search (workspace `*.css`) |
| 데이터 스케일 | 같은 displayFormat이라도 DB raw 스케일이 다르면 엑셀에서 다른 결과. 화면 정상 != 엑셀 정상 |
| 이미지 의존 | 캡처 의존한 추측은 신뢰도 낮음. 사용자가 직접 사실관계 확인해주는 단계 필수 |

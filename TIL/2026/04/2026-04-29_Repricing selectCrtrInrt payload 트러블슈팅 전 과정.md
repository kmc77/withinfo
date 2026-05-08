# 2026-04-29_Repricing selectCrtrInrt payload 트러블슈팅 전 과정

## 오늘 한 것 요약
- 핸드오버 문서에서 누락된 CF 필드 (`flctnInrtRclcYmdClssCd`) 발견
- `CrtrInrtReqVO.java` 필드 전체 확인
- `dataList1` 컬럼 목록 확인 (DataCollection Editor)
- `dma_rclcInrtMngReqVO` keyInfo 구조 확인 및 키 추가
- `oneditend` 무한루프 원인 분석 및 플래그 패턴으로 해결
- WebSquare Submission payload 비어있는 문제 트러블슈팅 전 과정
- `hidden="true"` 컬럼이 필요한 이유 이해
- DataMap `getValue` / `set` vs DataList `getCellData` / `setCellData` 구분
- WebSquare Submission ref 배열 구조 파악
- `pniRpmtDd` NPE 발생 (151라인) — 사수 확인 필요

---

## 1. 핸드오버에서 누락된 CF 파라미터 발견

`CashFlowCmptImpl.java` 2041~2042 라인 실제 호출:
```java
result.setInrt(selectCrtrInrt(
    vo.getBgngYmd(),
    vo.getEndYmd(),
    vo.getInrtClssCd(),
    vo.getFlctnInrtRclcYmdClssCd(),   // ← 핸드오버에 없었음
    vo.getIntAplcnWaysClssCd(),
    getRoundingMode(vo.getAdltPrcsMethClssCd()),
    vo.getRpmtYmd(),
    vo.getInrtCmptYmd(),
    vo.getNtnAreaCd(),
    vo.getBgngYmdFxtnYn()
));
```

---

## 2. CrtrInrtReqVO.java 필드 전체 목록

```java
private String bgngYmd;                // 계약기간시작일자
private String endYmd;                 // 계약기간종료일자
private String ntnAreaCd;              // 국가지역코드
private String inrtClssCd;             // 금리구분코드
private String flctnInrtRclcYmdClssCd; // 변동금리재산출(Repricing) 일자구분코드
private String intAplcnWaysClssCd;     // 이자적용방식코드
private String adltPrcsMethClssCd;     // 단수처리방법구분코드
private String rpmtYmd;                // 상환일자
private String inrtCmptYmd;            // 금리산출일자 (이전상환일자)
private String bgngYmdFxtnYn;          // 시작일자고정여부
```

**스펠링 주의:** `flctnInrtR**clc**YmdClssCd` — 중간에 `Relc`로 혼동했으나 `Rclc`가 맞음

---

## 3. dataList1 컬럼 목록 확인 (DataCollection Editor)

| No | id | name |
|---|---|---|
| 1 | _chk | check |
| 2 | ctrtNo | 계약번호 |
| 3 | bizNm | 사업번호 |
| 4 | deptNm | 담당부서 |
| 5 | aplcnYmd | 적용일자 |
| 6 | inrtCtrYmd | Repricing기준일자 |
| 7 | flctnInrtRclcYmdClssCd | Look-back-date |
| 8 | inrtCmptYmd | Observation Date |
| 9 | cmptInrtClssCd | 기준금리 |
| 10 | crtrInrt | 기준금리값(A) |
| 11 | inrtSprldtrt | 조정금리(B) |
| 12 | spread | 적용기준금리(A+B) |
| 13 | mdfcnRsn | 수정사유 |
| 14 | intAmt | 수정 |
| 15 | rsngNm | 사업명 |
| 16~24 | hidden 컬럼들 | rpmtYmd, inrtClssCd, bgngYmd, endYmd, intAplcnWaysClssCd, adltPrcsMethClssCd, ntnAreaCd, bgngYmdFxtnYn, pniRpmtDb |

---

## 4. dataList1 컬럼명 vs CF 필드명 최종 매핑

| dataList1 컬럼 | CF 필드명 | 처리 방식 |
|---|---|---|
| `aplcnYmd` | `rpmtYmd` | dma.set 매핑 |
| `cmptInrtClssCd` | `inrtClssCd` | dma.set 매핑 |
| `flctnInrtRclcYmdClssCd` | `flctnInrtRclcYmdClssCd` | 이름 동일 |
| `inrtCmptYmd` | `inrtCmptYmd` | 이름 동일 |
| `rpcgAplcnBgngYmd` (dma) | `bgngYmd` | `dma.getValue()`로 가져옴 |
| `rpcgAplcnEndYmd` (dma) | `endYmd` | `dma.getValue()`로 가져옴 |
| 없음 | `intAplcnWaysClssCd` | 하드코딩 `"89030003"` |
| 없음 | `adltPrcsMethClssCd` | 하드코딩 `"11020002"` |
| 없음 | `ntnAreaCd` | 하드코딩 `"KR"` |

**bgngYmd / endYmd 출처:** 화면 상단 `dma_rclcInrtMngReqVO`의 `rpcgAplcnBgngYmd`, `rpcgAplcnEndYmd`  
→ 화면 dataList1에 해당 컬럼 없음 → dma에서 꺼내서 세팅

---

## 5. hidden 컬럼이 필요한 이유

WebSquare Submission은 DataCollection에 정의된 컬럼/키 기준으로 JSON payload를 만든다.  
**컬럼 정의 없이 `setCellData` 호출 → 값 저장 안 됨 → payload에 미포함 → 서버 null**

```
setCellData(row, "rpmtYmd", "20240101")
→ dataList1에 rpmtYmd 컬럼 정의 없음
→ 무시됨
→ payload: { data: {} }   ← 비어있음
```

해결: `hidden="true"` 속성으로라도 컬럼 정의 필수  
→ 화면에 안 보이지만 DataCollection에 값 저장 + 서버 전송 가능

```xml
<w2:column id="rpmtYmd"                dataType="text" hidden="true"></w2:column>
<w2:column id="inrtClssCd"             dataType="text" hidden="true"></w2:column>
<w2:column id="bgngYmd"                dataType="text" hidden="true"></w2:column>
<w2:column id="endYmd"                 dataType="text" hidden="true"></w2:column>
<w2:column id="intAplcnWaysClssCd"     dataType="text" hidden="true"></w2:column>
<w2:column id="adltPrcsMethClssCd"     dataType="text" hidden="true"></w2:column>
<w2:column id="ntnAreaCd"              dataType="text" hidden="true"></w2:column>
<w2:column id="bgngYmdFxtnYn"          dataType="text" hidden="true"></w2:column>
<w2:column id="pniRpmtDb"              dataType="text" hidden="true"></w2:column>
```

---

## 6. 컬럼 이름만 바꿔서 보낼 수 없는 이유

WebSquare Submission은 DataCollection 컬럼 ID를 그대로 JSON 키로 전송한다:
```json
{ "aplcnYmd": "20240101", "cmptInrtClssCd": "001" }
```
서버 VO 매핑 시 **JSON 키 = VO 필드명** 이어야 한다.  
`rpmtYmd` 필드에 값을 넣으려면 JSON에 `"rpmtYmd"` 키가 있어야 하므로  
→ dataList1에 `rpmtYmd` 컬럼 정의가 필수다.

VO에 `@JsonProperty("aplcnYmd")` 붙이는 우회도 가능하지만 서버단 CF 로직 수정이므로 금지.

---

## 7. oneditend 무한루프 원인 및 해결

### 원인
```
setCellData 호출
→ WebSquare 내부: $W._s._22.handleEndEdit 발동
→ $W._s._29.notifyBeforeSettedCellData
→ oneditend 재발동
→ 다시 setCellData → 무한루프
```

콘솔에서 확인: `scwin.grd_list_oneditend` 스택이 수천 번 반복

### 초기 시도 (실패)
`setCellData(row, "inrtCmptYmd", value)` 제거 → 다른 setCellData들도 동일하게 재발동

### 최종 해결: 플래그 패턴
```js
scwin._isSettingCellData = false;

scwin.grd_list_oneditend = function(row, col, value, colId) {
    if(scwin._isSettingCellData) return;   // 재진입 차단
    if(col != grd_list.getColumnIndex("inrtCmptYmd")) return;
    if(com.util.isEmpty(value)) return;

    scwin._isSettingCellData = true;       // 진입 플래그 ON

    // ... setCellData 로직 ...

    scwin._isSettingCellData = false;      // 플래그 OFF

    scwin.selectedIndex = row;
    com.sbm.execute(...);
};
```

---

## 8. WebSquare Submission payload 비어있는 문제 전 과정

### 시도 1: ref index 표현식 (실패)
```xml
ref='data:json,{"id":"dataList1","index":"{scwin.selectedIndex}","key":"crtrInrtReqVO"}'
```
→ `{scwin.selectedIndex}` 런타임 평가 안 됨 → payload: `{ data: {} }`

따옴표 제거도 시도:
```xml
ref='data:json,{"id":"dataList1","index":{scwin.selectedIndex},"key":"crtrInrtReqVO"}'
```
→ 동일하게 비어있음

### 시도 2: JS setAttribute로 동적 변경 (동작하나 억지)
```js
var sbm = $p.getComponentById("sbm_selectObservationDateInrt");
sbm.setAttribute("ref", 'data:json,{"id":"dataList1","index":' + row + ',"key":"crtrInrtReqVO"}');
```
→ 구조상 억지스러워 채택 안 함

### 시도 3: dma.setValue() (실패)
```js
var dma = $p.getComponentById("dma_rclcInrtMngReqVO");
dma.setValue("rpmtYmd", dl.getCellData(row, "aplcnYmd"));
```
→ keyInfo에 해당 키 없으면 payload에 미포함 → header 값만 나옴

### 시도 4: dma.set() (keyInfo 추가 후 해결)
```js
dma_rclcInrtMngReqVO.set("rpmtYmd", dl.getCellData(row, "aplcnYmd"));
```
→ `dma_rclcInrtMngReqVO` DataCollection Editor에서 keyInfo 키 추가 후 정상 동작

---

## 9. dma_rclcInrtMngReqVO keyInfo 추가 목록

DataCollection Editor → dma_rclcInrtMngReqVO 선택 후 Key 탭에서 추가:

| No | id | name |
|---|---|---|
| 기존 1 | rpcgAplcnBgngYmd | 적용시작일자 |
| 기존 2 | rpcgAplcnEndYmd | 적용종료일자 |
| 기존 3 | bizClssCd | 사업구분 |
| 기존 4 | deptCd | 담당부서코드 |
| 기존 5 | deptNm | 부서명 |
| **추가 6** | rpmtYmd | 상환일자 |
| **추가 7** | inrtClssCd | 금리구분코드 |
| **추가 8** | bgngYmd | 계약기간시작일자 |
| **추가 9** | endYmd | 계약기간종료일자 |
| **추가 10** | intAplcnWaysClssCd | 이자적용방식코드 |
| **추가 11** | adltPrcsMethClssCd | 단수처리방법구분코드 |
| **추가 12** | ntnAreaCd | 국가지역코드 |
| **추가 13** | bgngYmdFxtnYn | 시작일자고정여부 |
| **추가 14** | pniRpmtDb | name24 |

---

## 10. WebSquare DataMap vs DataList API

| | DataMap (dma) | DataList (dl) |
|---|---|---|
| 읽기 | `getValue("키")` | `getCellData(row, "컬럼")` |
| 쓰기 | `setValue("키", 값)` 또는 `set("키", 값)` | `setCellData(row, "컬럼", 값)` |
| 오류 | `getCellData` 호출 시 → `is not a function` | `getValue` 호출 시 → `is not a function` |

**혼동 주의:** `$p.getComponentById("dma_...")` 로 가져와도  
DataMap이면 `getCellData` 사용 불가 → `getValue` 사용

---

## 11. WebSquare Submission ref 배열 구조

여러 DataCollection을 동시에 서버로 전송 가능:
```xml
ref='data:json,[
  {"id":"dlt_cwBgtRqareRegListVO","key":"cwBgtRqareRegListVO"},
  {"id":"dma_cwBgtRqareRegInfoVO","key":"cwBgtRqareRegInfoVO"},
  {"id":"dma_cwBgtRqareRegReqVO","key":"cwBgtRqareRegReqVO"}
]'
```
→ 실제 타 화면에서 이 방식 사용 확인  
단, VO 필드명 = DataCollection 키/컬럼 id 일치 필수

---

## 12. pniRpmtDd NPE (미해결)

`CashFlowCmptImpl.java` 151라인:
```java
pniRpmtDd = Integer.parseInt(reqVO.getPniRpmtDd());
```
`getPniRpmtDd()` null → `Integer.parseInt(null)` → NPE

- `pniRpmtDd`는 화면에서 보내는 값이 아님
- CF 내부 로직에서 채워야 하는 값으로 추정
- **사수 확인 필요**
- 임시 테스트용: `pniRpmtDb` 컬럼 추가 + 하드코딩 값 세팅 시도 중

---

## 13. 현재 oneditend 최종 코드 (진행 중)

```js
scwin._isSettingCellData = false;

scwin.grd_list_oneditend = function(row, col, value, colId) {
    if(scwin._isSettingCellData) return;
    if(col != grd_list.getColumnIndex("inrtCmptYmd")) return;
    if(com.util.isEmpty(value)) return;

    scwin._isSettingCellData = true;

    var dl  = $p.getComponentById("dataList1");
    var dma = $p.getComponentById("dma_rclcInrtMngReqVO");

    dma_rclcInrtMngReqVO.set("rpmtYmd",                dl.getCellData(row, "aplcnYmd"));
    dma_rclcInrtMngReqVO.set("inrtClssCd",             dl.getCellData(row, "cmptInrtClssCd"));
    dma_rclcInrtMngReqVO.set("inrtCmptYmd",            dl.getCellData(row, "inrtCmptYmd"));
    dma_rclcInrtMngReqVO.set("flctnInrtRclcYmdClssCd", dl.getCellData(row, "flctnInrtRclcYmdClssCd"));
    dma_rclcInrtMngReqVO.set("bgngYmd",                dma.getValue("rpcgAplcnBgngYmd"));
    dma_rclcInrtMngReqVO.set("endYmd",                 dma.getValue("rpcgAplcnEndYmd"));
    dma_rclcInrtMngReqVO.set("bgngYmdFxtnYn",          "Y");
    dma_rclcInrtMngReqVO.set("intAplcnWaysClssCd",     "89030003");
    dma_rclcInrtMngReqVO.set("adltPrcsMethClssCd",     "11020002");
    dma_rclcInrtMngReqVO.set("ntnAreaCd",              "KR");

    scwin._isSettingCellData = false;

    scwin.selectedIndex = row;
    com.sbm.execute("sbm_selectObservationDateInrt", scwin.sbm_selectObservationDateInrt_submitdone);
};

scwin.sbm_selectObservationDateInrt_submitdone = function(e) {
    // dataList1 crtrInrt 자동 업데이트 예정
};
```

Submission:
```xml
<xf:submission id="sbm_selectObservationDateInrt"
    ref='data:json,{"id":"dma_rclcInrtMngReqVO","key":"crtrInrtReqVO"}'
    action="/api/sf/cs/cmsf/selectCrtrInrt"
    method="post"
    mediatype="application/json"
    mode="asynchronous"
    ev:submitdone="scwin.sbm_selectObservationDateInrt_submitdone">
</xf:submission>
```

---

## 14. 잔여 작업

- `pniRpmtDd` NPE 사수 확인
- payload 정상 전송 확인
- `crtrInrt` submitdone 후 그리드 반영 확인
- `saveRepricingAplcn` 백엔드 구현
- 선종코드 처리 (사수 확인)
- Git 커밋/푸시

---

## 오늘 배운 핵심 한 줄 정리

| 구분 | 핵심 |
|---|---|
| oneditend 무한루프 | `setCellData` → `handleEndEdit` → `oneditend` 재발동 → 플래그로 차단 |
| hidden 컬럼 필수 이유 | 컬럼 정의 없으면 `setCellData` 무시 → payload 미포함 |
| DataMap API | `getValue` / `set` 사용. `getCellData` 호출 시 타입에러 |
| dma keyInfo | `set()` 써도 keyInfo에 키 없으면 payload 미포함 |
| Submission ref 표현식 | `{scwin.selectedIndex}` 런타임 평가 안 됨 |
| Submission ref 배열 | `[{...}, {...}]` 로 여러 DataCollection 동시 전송 가능 |
| 컬럼명 다를 때 | JSON 키 = VO 필드명 일치 필수 → 화면단에서 매핑 처리 |

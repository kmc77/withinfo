# 2026-04-30_Repricing 금리관리 Observation Date 응답 바인딩 디버깅

## 작업 개요

| 항목 | 내용 |
|------|------|
| 대상 화면 | CWMI_SFCM201M.xml (Repricing 금리관리) |
| 패키지 | kobc.ngs.sf.cs.cmsf |
| 세션 주제 | Observation Date 변경 → 기준금리 조회 → crtrInrt 셀 바인딩 |
| 상태 | submitdone 진입 직전 / 응답 구조 확인 완료 / 바인딩 미완 |
| API | `/api/sf/cs/cmsf/selectCrtrInrt` |

---

## 1. 세션 시작 시점 (이전 핸드오버 인계)

이전 세션(2026-04-29)에서 백엔드 selectCrtrInrt 호출 시 NPE 발생. 다음 항목 사수 확인 대기 중이었음:
- nextRpmtYmd null 처리 — 백엔드에서 `substring(0,6)` 호출 시 NPE
- intAplcnWaysClssCd / adltPrcsMethClssCd 하드코딩 여부
- ntnAreaCd `"KR"` 동적 처리 여부
- 선종코드 처리 로직

**사수 답변 결과**: nextRpmtYmd 무시 가능 → 이번 세션은 그 기준으로 진행

---

## 2. 진행 흐름

### 2.1 dataList1 매핑 컬럼 검증

**페이로드 빈값 다수 확인 → 그리드 SELECT 쿼리(쿼리 엑셀 191~236라인) 분석**

UNION ALL의 두 번째 SELECT 매핑 관계:
| 쿼리 컬럼 | dataList1 컬럼 |
|---|---|
| `F.INVS_PRTM_BGNG_YMD` | bgngYmd |
| `F.INVS_PRTM_END_YMD` | endYmd |
| `'89030003' AS INT_APLCN_WAYS_CLSS_CD` | intAplcnWaysClssCd |
| `'11020002' AS ADLT_PRCS_METH_CLSS_CD` | adltPrcsMethClssCd |
| `'KR' AS NTN_AREA_CD` | ntnAreaCd |

**조건**: `A.CMPT_INRT_CLSS_CD = '900000013'` (특수 케이스만)

**현재 테스트 행**: `cmptInrtClssCd: "900000002"` → 첫 번째 SELECT 결과
→ DBeaver 단독 실행 시는 정상 조회됨. 즉 매핑/Mapper 로직 단계 의심.

### 2.2 oneditend 핸들러 보강

```javascript
scwin._isSettingCellData = false;
scwin.grd_list_oneditend = function(row, col, value, colId) {
    if(scwin._isSettingCellData) return;
    if(col != grd_list.getColumnIndex("inrtCmptYmd")) return;

    scwin._isSettingCellData = true;
    var dl = $p.getComponentById("dataList1");

    dma_rclcInrtMngReqVO.set("bgngYmd", dl.getCellData(row, "invsPrtmBgngYmd"));
    dma_rclcInrtMngReqVO.set("endYmd",  dl.getCellData(row, "invsPrtmEndYmd"));
    dma_rclcInrtMngReqVO.set("inrtCmptYmd", dl.getCellData(row, "inrtCmptYmd"));
    dma_rclcInrtMngReqVO.set("rpmtYmd", dl.getCellData(row, "aplcnYmd"));
    dma_rclcInrtMngReqVO.set("inrtClssCd", dl.getCellData(row, "cmptInrtClssCd"));

    dma_rclcInrtMngReqVO.set("flctnInrtRclcYmdClssCd", '89050001'); // 직전적용일 고정
    dma_rclcInrtMngReqVO.set("intAplcnWaysClssCd", "89030003");
    dma_rclcInrtMngReqVO.set("adltPrcsMethClssCd", "11020002");
    dma_rclcInrtMngReqVO.set("ntnAreaCd", "KR");

    scwin._isSettingCellData = false;
    scwin.selectedIndex = row;
    com.sbm.execute("sbm_selectObservationDateInrt",
                    scwin.sbm_selectObservationDateInrt_submitdone);
};
```

### 2.3 [이슈 #1] getColumnIndex 회귀

- **증상**: `getColumnIndex("inrtCmptYmd", col)` — 두번째 인자가 코드에 다시 들어가 있음 (이전 세션에서 수정했던 회귀)
- **조치**: `getColumnIndex("inrtCmptYmd")` 단일 인자

### 2.4 [이슈 #2] dataList1 컬럼 값 비어있음

DataCollection Editor에서 dataList1 검증 결과 `invsPrtmBgngYmd` / `invsPrtmEndYmd` 컬럼 존재 확인. 그러나 viewCollection 결과 첫 행 데이터:
- **값 있음**: ctrtNo, bizNm, deptNm, aplcnYmd, inrtCrtrYmd, flctnInrtRclcYmdClssCd, inrtCmptYmd, cmptInrtClssCd, inrtSprdItrt
- **빈값**: bgngYmd, endYmd, rpmtYmd, inrtClssCd, intAplcnWaysClssCd, adltPrcsMethClssCd, ntnAreaCd, bgngYmdFxtnYn, nextRpmtYmd, invsPrtmBgngYmd, invsPrtmEndYmd

→ DBeaver 직접 실행 시는 정상. 매핑 단계 문제 가능성. 이번 세션에서는 우회 진행.

### 2.5 [이슈 #3] submitdone is not a function

WebSquare logMsg.html 로그 (`localhost:8080/svc/websquare/message/logMsg.html`):
```
Event error [...sbm_selectObservationDateInrt, ev:xforms-submit-done]:
mf_wdc_main_subWindow3_wframe_sbm_selectObservationDateInrt_submitdone
is not a function
```

**원인**: submission XML의 `ev:submitdone` 속성 값과 실제 등록된 함수 명세 불일치. 같은 화면 정상 submission(`sbm_selectRprcInrtList`) 패턴 그대로 따라가야 함.

### 2.6 [이슈 #4] setJSON data is null

```
[WebSquare.DataCollection.dataController.setJSON]
data is null[mf_wdc_main_subWindow3_wframe_dataList1]
```

submission target이 dataList1을 잡고 있는데 응답 매핑 데이터가 null. target ref 구조 재검토 필요.

### 2.7 [이슈 #5] 응답 구조 최종 확인

DevTools 콘솔 e 인자 출력 결과 — 응답 정상 도달. 핵심 구조:

```json
{
    "checkErrors": null,
    "frstRegDt": null,
    "frstRegUserId": null,
    "header": {
        "dlngId": "BUI260430135800800127000000000173",
        "srvcUriAddr": "/api/sf/cs/cmsf/selectCrtrInrt",
        "prgrsSn": 1
    },
    "inrt": 4.26531,
    "lastChgDt": null,
    "lastChgUserId": null,
    "prcsCrtrYmd": null
}
```

**핵심**: `inrt`가 최상위에 평면으로 옴 (data 래핑 없음). `responseStatusCode: 200`.

### 2.8 [이슈 #6] crtrInrt is not defined

```
ReferenceError: crtrInrt is not defined
at scwin.sbm_selectObservationDateInrt_submitdone
```

submitdone 코드에서 `crtrInrt`를 변수로 참조함. setCellData의 컬럼 ID 인자는 **문자열** `"crtrInrt"`로 전달 필요.

---

## 3. 다음 세션 즉시 작업

### 3.1 submission XML 정정

같은 화면 정상 동작 submission(`sbm_selectRprcInrtList`) 패턴 따라가기. 권장 형태:

```xml
<xf:submission id="sbm_selectObservationDateInrt"
    ref='data:json,{"id":"dma_rclcInrtMngReqVO"}'
    target='data:json,{"id":"dma_rclcInrtMngReqVO"}'
    action="/api/sf/cs/cmsf/selectCrtrInrt"
    method="post"
    mediatype="application/json"
    encoding="UTF-8"
    mode="asynchronous"
    ev:submitdone="scwin.sbm_selectObservationDateInrt_submitdone">
</xf:submission>
```

> ⚠️ `ev:submitdone` 값은 정상 submission 패턴 확인 후 일치 필요 (`scwin.` 접두사 유무 등).

### 3.2 submitdone 콜백 — e 인자에서 직접 추출

응답이 평면 구조이므로 dma 매핑 의존하지 않고 `e.responseJSON`에서 직접 추출:

```javascript
scwin.sbm_selectObservationDateInrt_submitdone = function(e) {
    var inrt = e.responseJSON.inrt;
    var dl = $p.getComponentById("dataList1");

    scwin._isSettingCellData = true;
    dl.setCellData(scwin.selectedIndex, "crtrInrt", inrt);
    scwin._isSettingCellData = false;
};
```

**핵심 포인트**:
- `"crtrInrt"` 문자열 — 변수 아닌 문자열로 전달 (이전 ReferenceError 원인)
- `e.responseJSON.inrt` — submitdone 첫 인자에서 직접. dma에 inrt 컬럼 추가 불필요
- `scwin.selectedIndex` — oneditend에서 이미 세팅됨
- `_isSettingCellData` 플래그 — setCellData가 oneditend 재진입 안 하도록 보호

### 3.3 검증 순서

1. submission `ev:submitdone` 명세를 정상 submission 패턴과 일치
2. submitdone 함수 진입 확인 (console.log)
3. `e.responseJSON.inrt` 값 확인
4. `dataList1[selectedIndex].crtrInrt`에 정상 바인딩 확인
5. displayFormat `#,##0.00000%` 적용 화면 표시 확인

---

## 4. 보류 사항

- **dataList1 빈값 컬럼**: bgngYmd, endYmd, invsPrtmBgngYmd 등이 SELECT 쿼리 결과에 안 채워지는 케이스. DBeaver 단독 실행 시는 정상 → 매핑/Mapper 로직 확인 필요
- **nextRpmtYmd 처리**: 사수 확인 결과 '무시'. 백엔드 null safe 처리 여부 확인 필요
- **선종코드 처리 로직**: 이전 세션부터 사수 확인 대기
- **saveRepricingAplcn 백엔드 구현**: 별도 후속 이슈
- **submission target 정리**: ref/target 같은 dma 사용 시 응답이 요청 덮어씀. 응답 전용 dma 분리 검토

---

## 5. 학습 포인트

### 5.1 WebSquare submission 디버깅

- **응답 구조는 `e.responseJSON`에서 raw 확인** — submitdone 첫 인자가 가장 정확
- **"결과 상태 메시지가 없습니다" 알림** = 표준 응답 wrapper 검증 실패. submitdone 진입 자체가 안 됨
- **logMsg.html 로그 뷰어** (`localhost:8080/svc/websquare/message/logMsg.html`) — submission 라이프사이클 추적용. start/complete/event error 시간 정보 모두 확인 가능
- **Event error log** — submitdone 함수 미인식 케이스 진단에 결정적

### 5.2 setCellData / 컬럼 ID

- `setCellData(row, colId, value)` — colId는 반드시 문자열. 변수처럼 쓰면 ReferenceError
- 재진입 방지 플래그 — `_isSettingCellData = true`로 oneditend ↔ setCellData 무한루프 차단

### 5.3 submission 속성 역할

| 속성 | 역할 |
|------|------|
| `id` | JS에서 `com.sbm.execute("id")`로 호출 |
| `action` | 서버 엔드포인트 URL |
| `method` | HTTP 메서드 (보통 post) |
| `ref` | 요청 데이터 소스 (`data:json,{"id":"dataMap_id","key":"래핑키"}`) |
| `target` | 응답 데이터 매핑 대상 |
| `mediatype` / `encoding` | application/json / UTF-8 |
| `mode` | asynchronous (기본) / synchronous |
| `ev:submitdone` | 응답 표준 검증 통과 후 호출 콜백 |
| `ev:submiterror` | 에러 시 호출 콜백 |
| `customHandler` | 표준 wrapper 우회 응답 처리용 |

### 5.4 디버깅 우선순위

1. logMsg.html 로그 → submission 라이프사이클 확인
2. DevTools Network → 응답 본문 직접 확인 (Content-Length, JSON 구조)
3. submitdone 인자 e 콘솔 출력 → responseJSON 구조 파악
4. DataCollection Editor → 컬럼 명세 확인
5. 같은 화면 정상 submission → 패턴 비교

---

## 6. 환경 정보

- 프론트: WebSquare Studio (CWMI_SFCM201M.xml)
- 백엔드: Eclipse + eGovFrame, Tomcat 9.0 로컬
- 주요 파일: `CashFlowCmptContoroller.java`, `CashFlowCmptServiceImpl.java`, `CrtrInrtReqVO.java`, `CrtrInrtResVO.java`
- Service public selectCrtrInrt: `CrtrInrtResVO` 반환 (`result.setInrt` 패턴)
- Service private selectCrtrInrt: `BigDecimal` 반환 (분기: COMPOUNDED_SOFR, TERM/OVERNIGHT/INDEX SOFR, SPE_BOND_AAA_3M, else)
- Git: `kmc77` 브랜치

---

## 7. 인덱스 추가용 라인

```
| 2026-04-30 | Repricing Observation Date 응답 바인딩 디버깅 | [📄 보기](경로) |
```

파일명: `2026-04-30_Repricing_Observation_Date_응답바인딩_디버깅.md`
폴더: `TIL/2026/04/`

# 2026-04-28_Repricing oneditend 구현 및 CF 필드매핑 디버깅

## 오늘 한 것
- `grd_list_oneditend` 구현 — Observation Date 변경 시 기준금리 재조회
- `oneditend` 파라미터 버그 원인 파악 및 `getColumnIndex` 우회
- TypeError 디버깅 (submission ID, `submit()` → `com.sbm.execute`)
- `bgngYmdFxtnYn` YN 플래그 역할 파악 (회의록 기반)
- `CashFlowCmptImpl.java` NPE 원인 분석
- dataList1 컬럼명 vs CF 필드명 불일치 파악 → 화면단 매핑 방향 확정
- 코딩테스트 2문제

---

## 코딩테스트 (아침)

### Java — 글자 이어 붙여 문자열 만들기
```java
class Solution {
    public String solution(String my_string, int[] index_list) {
        String answer = "";
        for(int i = 0; i < index_list.length; i++) {
            answer += my_string.charAt(index_list[i]);
        }
        return answer;
    }
}
```

### SQL (Oracle) — 조건에 부합하는 중고거래 댓글 조회
```sql
SELECT B.TITLE, B.BOARD_ID, R.REPLY_ID, R.WRITER_ID, R.CONTENTS,
       TO_CHAR(R.CREATED_DATE, 'yyyy-mm-dd') AS CREATED_DATE
  FROM USED_GOODS_BOARD B
 INNER JOIN USED_GOODS_REPLY R ON B.BOARD_ID = R.BOARD_ID
 WHERE TO_CHAR(B.CREATED_DATE, 'yyyy-mm') = '2022-10'
 ORDER BY R.CREATED_DATE ASC, B.TITLE ASC
```

---

## WebSquare

### oneditend 파라미터 버그 — colId 자리에 value가 찍힘
`oneditend` 파라미터는 `(row, col, colId, value)` 인데, 실제로 찍어보면 `colId` 자리에 value가 들어옴.

**해결: `col` 인덱스로 비교**
```js
// colId로 비교하면 안됨 (버그)
if(colId != "inrtCmptYmd") return;

// col 인덱스로 비교 (정상)
if(col != grd_list.getColumnIndex("inrtCmptYmd")) return;
```

### com.sbm.execute 사용법
`submit()` 대신 사용. 콜백 함수 직접 전달.
```js
com.sbm.execute("sbm_id", callbackFn);
```

### dataList1 hidden 컬럼 추가
서버에 전달해야 하지만 화면에 보여주지 않아도 되는 컬럼은 `hidden="true"` 처리.
```xml
<w2:column id="rpmtYmd" dataType="text" hidden="true"></w2:column>
```

---

## 회의록 파악 — bgngYmdFxtnYn 역할

| 상황 | bgngYmdFxtnYn | 쿼리 동작 |
|---|---|---|
| 초기 조회 | 세팅 안 함 | 레그함수(`B-O`) 자동 적용 |
| 수동 일자 변경 | `"Y"` | 입력 일자 그대로 적용 (레그함수 무시) |

---

## CF 필드매핑 — dataList1 컬럼명 vs CF 필드명

조회쿼리 엑셀의 소문자 메모 = CF 임플(`CashFlowCmptImpl.java`)이 요구하는 필드명.
AS 처리가 아닌 주석이므로 dataList1 컬럼명과 CF 필드명이 다름.
**원칙: 서버단 CF 로직 수정 금지 → 화면단에서 맞춰서 세팅**

| dataList1 컬럼 | CF 필드명 | 처리 |
|---|---|---|
| `aplcnYmd` | `rpmtYmd` | setCellData 매핑 |
| `cmptInrtClssCd` | `inrtClssCd` | setCellData 매핑 |
| `invsPrtmBgngYmd` | `bgngYmd` | setCellData 매핑 (컬럼명 재확인 필요) |
| `invsPrtmEndYmd` | `endYmd` | setCellData 매핑 (컬럼명 재확인 필요) |
| 없음 | `intAplcnWaysClssCd` | 하드코딩 `'89030003'` |
| 없음 | `adltPrcsMethClssCd` | 하드코딩 `'11020002'` |
| 없음 | `ntnAreaCd` | 하드코딩 `'KR'` |

---

## 확정된 코드

### dataList1 hidden 컬럼 추가
```xml
<w2:column id="rpmtYmd"            dataType="text" hidden="true"></w2:column>
<w2:column id="inrtClssCd"         dataType="text" hidden="true"></w2:column>
<w2:column id="bgngYmd"            dataType="text" hidden="true"></w2:column>
<w2:column id="endYmd"             dataType="text" hidden="true"></w2:column>
<w2:column id="intAplcnWaysClssCd" dataType="text" hidden="true"></w2:column>
<w2:column id="adltPrcsMethClssCd" dataType="text" hidden="true"></w2:column>
<w2:column id="ntnAreaCd"          dataType="text" hidden="true"></w2:column>
<w2:column id="bgngYmdFxtnYn"      dataType="text" hidden="true"></w2:column>
```

### grd_list_oneditend + submitdone
```js
scwin.grd_list_oneditend = function(row, col, colId, value) {
    if(col != grd_list.getColumnIndex("inrtCmptYmd")) return;
    if(com.util.isEmpty(value)) return;

    var dl = $p.getComponentById("dataList1");

    dl.setCellData(row, "inrtCmptYmd",         value);
    dl.setCellData(row, "bgngYmdFxtnYn",        "Y");
    dl.setCellData(row, "rpmtYmd",              dl.getCellData(row, "aplcnYmd"));
    dl.setCellData(row, "inrtClssCd",           dl.getCellData(row, "cmptInrtClssCd"));
    dl.setCellData(row, "bgngYmd",              dl.getCellData(row, "invsPrtmBgngYmd"));
    dl.setCellData(row, "endYmd",               dl.getCellData(row, "invsPrtmEndYmd"));
    dl.setCellData(row, "intAplcnWaysClssCd",   "89030003");
    dl.setCellData(row, "adltPrcsMethClssCd",   "11020002");
    dl.setCellData(row, "ntnAreaCd",            "KR");

    scwin.selectedIndex = row;
    com.sbm.execute("sbm_selectObservationDateInrt", scwin.sbm_selectObservationDateInrt_submitdone);
};

scwin.sbm_selectObservationDateInrt_submitdone = function(e) {
    // dataList1 자동 업데이트됨
};
```

---

## 내일 할 것
1. `invsPrtmBgngYmd`, `invsPrtmEndYmd` 실제 컬럼명 쿼리 엑셀 재확인
2. hidden 컬럼 추가 후 NPE 재테스트
3. `saveRepricingAplcn` 백엔드 구현 (Controller → ServiceImpl → Mapper → SQL)
4. 선종코드 처리 방식 사수 확인
5. Git 커밋/푸시

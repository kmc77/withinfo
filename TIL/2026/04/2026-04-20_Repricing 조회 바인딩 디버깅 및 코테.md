# 2026-04-20_Repricing 조회 바인딩 디버깅 및 코테

---

## 🧩 코딩테스트

### SQL - 업그레이드 된 아이템 구하기 (프로그래머스)

**풀이 접근:**  
ITEM_TREE를 JOIN해서 PARENT_ITEM_ID가 RARE인 아이템의 자식 아이템을 구하는 문제.  
서브쿼리로 RARE인 ITEM_ID 목록을 뽑고 IN 조건으로 필터링.

```sql
select F.ITEM_ID, F.ITEM_NAME, F.RARITY
from ITEM_INFO F
join ITEM_TREE T
on F.ITEM_ID = T.ITEM_ID
where T.PARENT_ITEM_ID IN (
    select ITEM_ID from ITEM_INFO where RARITY = 'RARE'
)
order by F.ITEM_ID desc;
```

**결과:** ✅ 테스트 통과

---

### Java - 주사위 게임 3 (프로그래머스)

**풀이 접근:**  
4개 주사위를 정렬 후 같은 값 개수 패턴으로 분기.

```java
import java.util.*;

class Solution {
    public int solution(int a, int b, int c, int d) {
        int answer = 0;
        int[] list = {a, b, c, d};
        Arrays.sort(list);

        if (list[0] == list[3]) {
            answer = 1111 * list[0];
        } else if (list[0] == list[1] || list[2] == list[3]) {
            answer = (int) Math.pow(list[1] * 10 + (list[0] + list[3] - list[1]), 2);
        }

        return answer;
    }
}
```

**결과:** ❌ 테스트 3, 4 실패 - 미해결 (추후 재풀이 필요)

---

## 🛠️ Repricing 금리관리 개발 (CWMI_SFCM201M.xml)

### 1. DataCollection dlt_ vs dma_ 혼용 문제

**증상:** 공통 팝업 행 더블클릭 시 `dlt_rclcInrtMngReqVO is not defined` 에러

**원인:**  
Submission submitdone 함수 내에서 `dlt_rclcInrtMngReqVO.setRowPosition()` 호출했는데,  
DataCollection에 `dlt_`는 선언되어 있지 않고 `dma_rclcInrtMngReqVO`만 존재.

**해결:**  
- `dlt_rclcInrtMngReqVO.setRowPosition(...)` → `dataList1.setRowPosition(...)` 으로 수정
- `setRowPosition`은 DataList에 쓰는 메서드이므로 그리드 결과 리스트에 써야 함

**교훈:**
- `dma_` = DataMap (단건 조회조건)
- `dlt_` = DataList (복수 결과)
- DataCollection 탭에서 실제 선언된 id 반드시 확인

---

### 2. Submission Target 설정 오류

**증상:** 서버 200인데 그리드에 데이터 미바인딩

**원인:**  
Submission Target이 `dlt_rclcInrtMngReqVO`로 잘못 설정되어 있었음.  
서버 응답 키가 `rclcInrtMngListVO`인데 이를 받을 Target이 없었던 것.

**해결:**  
Submission Edit → Target을 `dataList1`으로 수정:
```
data:json,{"id":"dataList1","key":"rclcInrtMngListVO"}
```

**교훈:**
- Submission Reference = 요청 파라미터 (dma_)
- Submission Target = 응답 결과 (dataList)
- Target key는 서버 응답 JSON의 실제 키명과 일치해야 함

---

### 3. VO 필드 누락으로 인한 MyBatis 에러

**에러:**
```
There is no getter for property named 'chrgDeptNo'
in class kobc.ngs.sf.cs.cmsf.vo.RclcInrtMngReqVO
```

**원인:** SQL에서 `#{chrgDeptNo}` 참조하는데 ReqVO에 해당 필드 미선언

**해결:** ReqVO에 필드 추가
```java
private String chrgDeptNo;
```

**교훈:**
- SQL `#{}` 파라미터명은 ReqVO 필드명과 반드시 1:1 일치
- `@Getter` 사용 시 필드 추가만 하면 getter 자동 생성

---

### 4. 공통 팝업 결과값 VO 매핑

**핵심 개념:**  
공통 팝업에서 선택한 값은 별도 VO 없이 기존 `dma_`에 직접 set.

```js
scwin.callback_부서팝업 = function(result) {
    dma_rclcInrtMngReqVO.set("deptCd", result.deptCd);
    dma_rclcInrtMngReqVO.set("deptNm", result.deptNm);
}
```

**교훈:** 팝업 결과를 담기 위한 별도 VO/DataCollection 불필요. 기존 조회조건 dma_에 set하면 됨.

---

### 5. SQL 파라미터명 ↔ VO 필드명 불일치로 조회 0건

**증상:**  
- DBeaver에서 동일 쿼리 실행 시 3건 조회
- 앱에서는 `rclcInrtMngListVO: []` 빈 배열 반환
- DB에 `CHRG_DEPT_CD = '478'` 데이터 92건 존재 확인

**디버깅 과정:**
1. 네트워크 탭 확인 → 응답은 200, 리스트 빈 배열
2. ServiceImpl에 `System.out.println` 추가 → 파라미터 값 정상 확인
3. DB에서 `COUNT(*)` 조회 → 92건 존재 확인
4. SQL 파라미터명 vs VO 필드명 비교 → 불일치 발견

**교훈:**
- 서버 200 + 빈 배열 = 쿼리 0건 → SQL 파라미터 바인딩 문제 의심
- DBeaver 조회 가능 + 앱 0건 = 파라미터명 불일치 패턴
- ReqVO 필드명, SQL `#{}`, DataCollection key id 3개가 모두 일치해야 함
- 디버깅 순서: 네트워크 탭 → println → DB 직접 조회 → SQL 파라미터 비교

---

### 6. submitdone 함수 정리

**원본 (복사본 잔재 포함):**
```js
scwin.sbm_selectRprcInrtList_submitdone = function(e) {
    dlt_rclcInrtMngReqVO.setRowPosition(scwin.selectedIndex);  // ← 에러
    grd_list.setFocusedCell(scwin.selectedIndex, "apsnYmd", true);
    ibx_apsnYmd.focus();
    // ... 버튼 비활성화 코드
    var elapLinkId = dlt_rclcInrtMngReqVO.getCellData(...);    // ← 불필요
    wfm_elapEml.getWindow().scwin.selectElapEml(elapLinkId, emlSndngSn);  // ← 불필요
};
```

**수정 후:**
```js
scwin.sbm_selectRprcInrtList_submitdone = function(e) {
    if (com.util.isEmpty(scwin.selectedIndex)) {
        scwin.selectedIndex = 0;
    }

    if (dataList1.getRowCount() == 0) {
        com.win._message(gcm.MESSAGE_TYPE.INFO, "조회된 데이터가 없습니다.");
        return;
    }

    dataList1.setRowPosition(scwin.selectedIndex);
    grd_list.setFocusedCell(scwin.selectedIndex, "aplcnYmd", true);
};
```

**교훈:**
- 다른 화면에서 복사한 코드는 불필요한 로직 반드시 제거
- elapEml(경과메일) 같은 타 화면 전용 로직은 과감히 삭제
- 0건 처리 로직 submitdone에 항상 포함

---

### 7. onmodelchange 패턴

부서명 삭제 시 부서코드도 같이 초기화하는 패턴:

```js
scwin.dma_rclcInrtMngReqVO_onmodelchange = function(info) {
    if (info.key == "deptNm") {
        if (com.util.isEmpty(dma_rclcInrtMngReqVO.get("deptNm"))) {
            dma_rclcInrtMngReqVO.set("deptCd", "");
            dma_rclcInrtMngReqVO.set("deptNm", "");
        }
    }
};
```

---

## 💻 기타 개발 환경 팁

### Windows Tomcat 프로세스 킬

```cmd
netstat -ano | findstr :8080
taskkill /PID [PID번호] /F
```

### 비밀번호 변경

`Win + I` → 계정 → 로그인 옵션 → 비밀번호 → 변경

---

## 🔴 미해결 - 다음 세션 인계

**현상:** SQL 파라미터 정상 전달 확인됐으나 여전히 조회 0건

**다음 세션 즉시 확인:**
```java
// ServiceImpl mapper 호출 직전
System.out.println("deptCd: [" + rclcInrtMngReqVO.getDeptCd() + "]");
System.out.println("chrgDeptNo: [" + rclcInrtMngReqVO.getChrgDeptNo() + "]");
System.out.println("bizClssCd: [" + rclcInrtMngReqVO.getBizClssCd() + "]");
System.out.println("rpcgAplcnBgngYmd: [" + rclcInrtMngReqVO.getRpcgAplcnBgngYmd() + "]");
```

SQL 파라미터명과 VO getter명 1:1 매칭 확인 후 UNION ALL 4개 전부 통일.

**테스트 데이터:**
| 파라미터 | 값 |
|---------|-----|
| rpcgAplcnBgngYmd | 20260101 |
| rpcgAplcnEndYmd | 20261231 |
| bizClssCd | 0005 |
| deptCd | 478 |

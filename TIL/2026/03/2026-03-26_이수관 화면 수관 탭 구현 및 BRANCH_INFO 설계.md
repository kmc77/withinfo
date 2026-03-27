# 2026-03-26_이수관 화면 수관 탭 구현 및 BRANCH_INFO 설계

## 오늘 한 것

- 이수관 화면 수관 탭 UI 완성 및 백엔드 연동
- BRANCH_INFO 테이블 생성 및 더미 데이터 INSERT
- EMP_AUTH_TRANSFER에 TRANSFER_STATUS 컬럼 추가
- 이관 신청 → 수관자 승인/거절 프로세스 설계
- 체크박스 비활성화 문제 해결
- WebSquare btn_search 클래스 자동 숨김 문제 해결

---

## BRANCH_INFO 테이블

```sql
CREATE TABLE BRANCH_INFO (
    BRANCH_CD   VARCHAR(10)  NOT NULL COMMENT '지점코드',
    BRANCH_NM   VARCHAR(50)  NOT NULL COMMENT '지점명',
    BRANCH_TYPE VARCHAR(20)  NOT NULL COMMENT 'HQ/BR',
    PRIMARY KEY (BRANCH_CD)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO BRANCH_INFO VALUES ('BR001', '본사',      'HQ');
INSERT INTO BRANCH_INFO VALUES ('BR002', '진구지점',  'BR');
INSERT INTO BRANCH_INFO VALUES ('BR003', '해운대지점','BR');
INSERT INTO BRANCH_INFO VALUES ('BR004', '사상지점',  'BR');
INSERT INTO BRANCH_INFO VALUES ('BR005', '동래지점',  'BR');
```

---

## TRANSFER_STATUS 컬럼 추가

```sql
ALTER TABLE EMP_AUTH_TRANSFER
ADD COLUMN TRANSFER_STATUS VARCHAR(20) DEFAULT '신청중' COMMENT '신청중/이관완료/거절';
```

### 상태값 흐름

```
이관자 신청 → [신청중] → 수관자 승인 → [이관완료]
                       → 수관자 거절 → [거절]
```

---

## 이관 신청/승인/거절 프로세스 변경

### 기존 (즉시 처리)
```
이관하기 클릭 → EMP_PROGRAM_AUTH INSERT + 소프트딜리트 + 이력 INSERT
```

### 변경 후 (승인 프로세스)
```
이관하기 클릭 → EMP_AUTH_TRANSFER INSERT (TRANSFER_STATUS = '신청중')
수관자 승인   → EMP_PROGRAM_AUTH INSERT + 소프트딜리트 + TRANSFER_STATUS = '이관완료'
수관자 거절   → TRANSFER_STATUS = '거절'
```

---

## searchPending API

수관 탭에서 나에게 들어온 신청중 이관 목록 조회.

### Mapper
```xml
<select id="searchPending" parameterType="map" resultType="map">
    SELECT
        t.TRANSFER_NO,
        (SELECT EMP_NM FROM EMPLOYEE WHERE EMP_NO = t.FROM_EMP_NO) AS FROM_EMP_NM,
        t.PROGRAM_CD,
        t.PROGRAM_NM,
        t.TRANSFER_DT,
        t.TRANSFER_STATUS
    FROM EMP_AUTH_TRANSFER t
    WHERE t.TO_EMP_NO = #{EMP_NO}
    AND t.TRANSFER_STATUS = '신청중'
    ORDER BY t.TRANSFER_DT DESC
</select>
```

### Controller
```java
@RequestMapping("/with/sample/searchPending")
public @ResponseBody Map<String, Object> searchPending(@RequestBody Map<String, Object> param) {
    Result result = new Result();
    try {
        Map dmPendingEmp = (Map) param.get("dm_pending_emp");
        List<Map<String, Object>> list = transfer.searchPending(dmPendingEmp);
        result.setMsg(result.STATUS_SUCESS, "조회 성공");
        result.getResult().put("dlt_pending_list", list);
    } catch (Exception ex) {
        ex.printStackTrace();
        result.setMsg(result.STATUS_ERROR, "조회 실패");
    }
    return result.getResult();
}
```

---

## 트러블슈팅

### 1. WebSquare `btn_search` 클래스 자동 숨김

**증상**: 수관 탭 Search 버튼이 `display:none; visibility:hidden`으로 렌더링됨

**원인**: WebSquare가 `btn_search` 클래스 버튼을 `dfbox` 내부에서 자동으로 숨기는 동작이 있음

**해결**: `class="btn_search"` → `class="btn_cm"` 으로 변경하거나 `anchorWithGroupClass=""` 속성 제거

---

### 2. 그리드 체크박스 비활성화

**증상**: `grd_pending` 체크박스 클릭 불가

**원인**: `readOnly="true"` 속성이 체크박스 컬럼까지 비활성화시킴

**해결**: `readOnly="true"` 제거. 셀 편집은 각 body 컬럼의 `displayMode="label"`로 개별 제어

---

### 3. `sbm_searchHistory` ref 오염

**증상**: 이력 조회 시 `HttpMessageNotReadableException` 발생

**원인**: `sbm_searchHistory`의 `ref`에 `dm_pending_emp`가 잘못 들어가 있었음

**해결**: 이력 조회는 파라미터 없으므로 `ref='data:json,[]'`로 수정

---

### 4. IS_USE 컬럼 한글명 수정

`EMP_PROGRAM_AUTH.IS_USE` 컬럼 — 실제 용도는 소프트딜리트용이므로 한글명 수정

```sql
ALTER TABLE EMP_PROGRAM_AUTH
MODIFY COLUMN IS_USE CHAR(1) COMMENT '삭제여부 Y:삭제 N:정상';
```

---

## 수관 탭 프론트 구조

```
수관 탭
├── 직원선택 드롭다운 (sbx_to_employee2 / dm_pending_emp)
├── Search 버튼 → sbm_searchPending 실행
├── 신청 목록 그리드 (grd_pending / dlt_pending_list)
│   └── 이관자 / 화면코드 / 화면명 / 신청일 / 상태 / 선택(체크박스)
└── 버튼 영역
    ├── 거절 버튼
    └── 수관 처리 버튼
```

### 핵심 포인트
- `dm_pending_emp` — 이관 탭 `dm_to_emp`와 충돌 방지를 위해 별도 dataMap 생성
- `CHK_YN2` — `dlt_from_auth`의 `CHK_YN`과 ID 충돌 방지

---

## 내일 할 것

- [ ] 수관 처리(승인) API 구현 (`/with/sample/approve`)
- [ ] 거절 API 구현 (`/with/sample/reject`)
- [ ] JS 승인/거절 함수 구현 (`btn_approve_onclick`, `btn_reject_onclick`)
- [ ] EMPLOYEE 테이블 BRANCH_CD 추가 (예빈 협의 후)
- [ ] 이관 신청 로직 변경 — 즉시처리 → 신청중 INSERT만

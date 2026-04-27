# 2026-04-17_Repricing 단위테스트 작성 및 ReqVO 필드 누락 수정

## 오늘 한 것

* `RclcInrtMngReqVO` 필드 누락(`inqEndAplcnYmd`) 원인 분석 및 수정
* `selectRepricingTrgt` SQL 구조 및 파라미터 매핑 확인
* 화면 초기 로드 시 조건 검색 입력 필드 포커스 추가
* 단위테스트 작성 진행 (조회 부분)

---

## 1. RclcInrtMngReqVO 필드 누락 수정

### 증상
조회 실행 시 아래 에러 발생:
```
There is no getter for property named 'inqEndAplcnYmd'
in class kobc.ngs.sf.cs.cmsf.vo.RclcInrtMngReqVO
```

### 원인
VO에 시작일(`inqBgngAplcnYmd`)만 선언되어 있고 종료일 누락.

### 수정
```java
// RclcInrtMngReqVO.java
private String inqBgngAplcnYmd;   // 기존
private String inqEndAplcnYmd;    // 추가
private String bizClssCd;
private String chrgDeptNo;
```

> `@Getter` 어노테이션 사용 중이므로 필드 추가만으로 getter 자동 생성됨

---

## 2. selectRepricingTrgt SQL 구조 확인

### 화면 ↔ 파라미터 매핑

| 화면 필드 | 파라미터 | 필수 |
|---|---|---|
| Repricing 적용일자 (시작) | `#{inqBgngAplcnYmd}` | ✅ |
| Repricing 적용일자 (종료) | `#{inqEndAplcnYmd}` | ✅ |
| 사업구분 | `#{bizClssCd}` | ✅ |
| 담당부서 | `#{chrgDeptNo}` | ✅ |

### UNION ALL 구조 (4개)

| 순번 | BIZ_CLSS_CD | 구분 | FROM 주요 테이블 |
|---|---|---|---|
| 1 | `'0005'` | 자금 | `TIV_FLCTN_INRT_APLCN_B` + `TAW_FND_RSNG_I` |
| 2 | `'0005'` | 운용 | `TIV_FLCTN_INRT_APLCN_B` + `TAW_FND_WRKG_I` |
| 3 | `'0001'` | 투자 | `TIV_FLCTN_INRT_APLCN_B` + `TIV_INVS_BOND_I` |
| 4 | `'0002'` | 보증 | `TIV_FLCTN_INRT_APLCN_B` + `TGU_ASSR_STPL_CNCL_I` |

**주의사항:**
- `<if>` 태그 없이 전부 고정 AND 조건 → 필수값 미입력 시 결과 0건 또는 오류
- `bizClssCd` 선택 시 해당 구분 쿼리만 결과 반환, 나머지 3개는 0건 (정상 동작)
- 화면에서 `*` 필수 표시된 3개 필드 모두 validation 걸려 있어 빈 값 조회 불가

---

## 3. 화면 초기 포커스 설정

onload 이벤트에서 조건 검색 첫 번째 입력 필드에 `focus()` 추가.

> ⚠️ blur 콜백 안에서 다시 focus() 호출 시 무한루프 발생 — onload 직접 호출만 허용

---

## 4. 단위테스트 작성 중 확인한 용어 정리

| 결과값 | 의미 |
|---|---|
| Pass | 정상 통과 |
| Fail | 실패 |
| N/A | Not Applicable — 해당 없음 (미구현, 환경 제약, 조건 성립 안됨) |

### 조회 단위테스트 체크 항목

| 항목 | 설명 |
|---|---|
| 리스트박스 코드명/코드값 정확성 | 드롭다운 표시 텍스트(코드명)와 실제 저장값(코드값)이 설계서와 일치하는지 |
| 다중 조건 검색 시 모든 조건 적용 여부 | 조건 여러 개 입력 시 전부 AND로 반영되는지, 일부 조건이 쿼리에서 누락되지 않는지 |
| 필수값 미입력 validation | 적용일자/사업구분/담당부서 미입력 시 조회 차단 여부 |

---

## 잔여 작업

| 항목 | 상태 | 비고 |
|------|------|------|
| `inqEndAplcnYmd` 추가 후 조회 정상 동작 확인 | 미완 | Tomcat 재시작 후 확인 필요 |
| 단위테스트 작성 완료 | 미완 | 조회 부분 진행 중 |
| saveRepricingAplcn 백엔드 구현 | 미완 | |
| 선종코드 별도처리 | 미완 | 사수 확인 필요 |
| Git 커밋/푸시 | 미완 | |

---

## TIL

* **ReqVO 필드 누락 디버깅**: `There is no getter for property named 'xxx'` → VO 필드 선언 여부 즉시 확인. `@Getter` 사용 시 필드만 추가하면 됨
* **UNION ALL 쿼리에서 `<if>` 미사용 시**: 동적 조건 없이 고정 AND이므로 파라미터 누락 시 0건 또는 null 비교 오류 — 프론트 validation 필수
* **단위테스트 N/A 사용 기준**: 환경 제약, 미구현, 해당 케이스가 현재 범위에 없을 때 사용. Pass/Fail 판정 불가한 경우
* **다중 조건 테스트 방법**: 조건 2개 이상 동시 입력 후 결과가 모든 조건을 만족하는지 직접 DB 조회로 교차 검증
* **리스트박스 코드값 검증**: 공통코드 그룹 기준으로 설계서 목록과 화면 드롭다운 항목 1:1 대조

---

## 전일(04-16) 이월 잔여 항목

| 항목 | 상태 |
|------|------|
| otherwise 분기 합계 금액 최종 검증 | 미완 |
| 테스트 데이터 정리 (4/10 이후 H 테이블 DELETE) | 미완 |
| 지정일자 당해년도 유효성 검사 복원 | 미완 |
| Git 커밋/푸시 (국가필수선대 + Repricing) | 미완 |
